# koserver — Operations Guide

Operational reference for the KOPDS server running on `<vm-name>`.
Covers how the Calibre library is mounted, how to switch between library sources,
the AppArmor confinement it depends on, and the failure modes worth recognising.

---


> Everything specific to a particular deployment is a placeholder in angle brackets — host
> names, usernames, ports, project and zone IDs, share and directory names, rclone remote
> names. Substitute your own; nothing here is environment-specific beyond those.

---

## Contents

- [1. Overview](#1-overview)
- [2. Quick Reference](#2-quick-reference)
- [3. Getting In](#3-getting-in)
- [4. The Two Library Sources](#4-the-two-library-sources)
  - [4.1 Personal library — NAS over Tailscale + SFTP (active)](#41-personal-library--nas-over-tailscale--sftp-active)
  - [4.2 Shared library — Nextcloud over WebDAV (dormant)](#42-shared-library--nextcloud-over-webdav-dormant)
  - [4.3 Switching between them](#43-switching-between-them)
- [5. Build Procedures](#5-build-procedures)
  - [5.1 Tailscale on the VM](#51-tailscale-on-the-vm)
  - [5.2 NAS service account and SSH key](#52-nas-service-account-and-ssh-key)
  - [5.3 Creating the rclone remote](#53-creating-the-rclone-remote)
  - [5.4 Repointing the mount](#54-repointing-the-mount)
- [6. AppArmor](#6-apparmor)
- [7. Index and Users](#7-index-and-users)
- [8. Troubleshooting](#8-troubleshooting)
- [9. Gotchas](#9-gotchas)
- [10. Maintenance](#10-maintenance)

---

## 1. Overview

`<vm-name>` is a GCP **e2-micro** (free tier, `<zone>`, project `<your-gcp-project>`)
running Ubuntu 26.04 LTS. It serves an OPDS feed to KOReader clients.

The library itself is **not stored on the VM** — the free tier's 30GB disk can't hold it.
Instead the library is mounted read-only over the network, and kopds maintains its own
small SQLite index locally.

```
   KOReader client
         │  OPDS / HTTPS
         ▼
   ┌─────────────────────────────────────────────┐
   │ <vm-name> (GCP e2-micro)                  │
   │                                             │
   │  kopds.service ──── kopds.db (local index)  │
   │       │                                     │
   │       │ reads                               │
   │       ▼                                     │
   │  /srv/calibre/library  ◄── kopds-library    │
   │                            (rclone mount)   │
   └────────────────┬────────────────────────────┘
                    │ Tailscale (WireGuard)
                    │ SFTP
                    ▼
          <nas-host> (Synology NAS, home LAN)
          /volume<N>/<library-share>/<library-dir>
```

**Why the mount is load-bearing:** kopds copies *metadata* into `kopds.db` at scan time,
so browsing works from the local index alone. Book **downloads** stream from the mount
every time. If the mount is down, the catalogue still lists books and every download 404s.

**Two-tier scanner:** the background worker stats `metadata.db` (mtime + size) every 30
minutes — nearly free over a WAN link. Only a change triggers a full read.

---

## 2. Quick Reference

| Item | Value |
|---|---|
| Service user | `kopds` (uid 999, gid 988), `nologin` shell |
| Home | `/var/lib/kopds` |
| Binary | `/opt/kopds/kopds` |
| Listen port | `8080` |
| Mountpoint | `/srv/calibre/library` |
| Index DB | `/var/lib/kopds/kopds.db` |
| Cover cache | `/var/lib/kopds/cache/` |
| rclone VFS cache | `/var/lib/kopds/rclone-cache/` |
| rclone config | `/var/lib/kopds/rclone.conf` |
| SSH key (to NAS) | `/var/lib/kopds/.ssh/id_ed25519` |
| NAS account | `<dsm-user>` — read-only on the library share, no interactive shell |
| Sync interval | 30m |
| Storage cap | 2000 MB |

**Units**

| Unit | Role |
|---|---|
| `kopds.service` | The OPDS server |
| `kopds-library.service` | rclone mount; **must be up before** `kopds.service` |
| `/etc/systemd/system/kopds-library.service.d/override.conf` | Drop-in selecting the active library source |

`srv-calibre-library.mount` sometimes appears in `systemctl list-units`. It is systemd's
automatic reflection of the live kernel mount, **not** a config file. Ignore it; drive
everything from `kopds-library.service`.

**Everyday commands**

```bash
# Status
systemctl status kopds kopds-library

# Restart the mount (also forces a reconnect to the NAS)
sudo systemctl restart kopds-library.service

# Is the mount live?
findmnt /srv/calibre/library

# List the library — note: MUST run as kopds (see Gotchas)
sudo -u kopds ls /srv/calibre/library/ | head

# rclone commands need the config path passed explicitly
sudo -u kopds RCLONE_CONFIG=/var/lib/kopds/rclone.conf rclone listremotes
```

---

## 3. Getting In

```bash
gcloud compute ssh <vm-name> --tunnel-through-iap
```

Full form if the defaults aren't set:

```bash
gcloud compute ssh <vm-name> \
  --project=<your-gcp-project> --zone=<zone> --tunnel-through-iap
```

The IAP tunnel drops on reboot with a `ConnectionCreationError` / broken pipe traceback.
That's expected — just re-run the command once the VM is back.

---

## 4. The Two Library Sources

Both rclone remotes are defined; only one is mounted at a time. The mountpoint stays
`/srv/calibre/library` regardless, which keeps the AppArmor rules and unit dependencies valid.

| Remote | Type | Source | Status |
|---|---|---|---|
| `<nas-remote>` | sftp | Synology NAS over Tailscale | **Active** |
| `<shared-remote>` | webdav | Nextcloud shared library | Dormant — origin offline |

### 4.1 Personal library — NAS over Tailscale + SFTP (active)

Set up 2026-08-14. The NAS sits behind the home LAN with no port forwarding, so
Tailscale provides the path and SFTP the transport.

**Why SFTP over SMB/NFS:** one TCP port, no RPC surface, survives the tunnel dropping,
and rclone handles reconnects and bandwidth throttling on top of it.

| Parameter | Value |
|---|---|
| Tailscale host | `<nas-host>` |
| SSH/SFTP port | non-default (DSM was moved off 22) |
| DSM user | `<dsm-user>` — dedicated service account, not a general-purpose login |
| Account scope | read-only on `<library-share>`; no other share visible; no interactive shell |
| Auth | ed25519 key, no passphrase, authorized on that account only |
| rclone path | `<library-share>/<library-dir>` |
| Real NAS path | `/volume<N>/<library-share>/<library-dir>` |

**Credential scope is the security boundary here.** The key on the VM is passwordless, so
the blast radius of a compromised VM is exactly whatever that key can reach. Scoping the
DSM account to read-only on one share means the answer is "the books" and nothing else —
see [§5.2](#52-nas-service-account-and-ssh-key).

**The path is not the filesystem path.** Synology's SFTP root presents *shares* flattened
across volumes, so there is no `/volume<N>` prefix. Listing the bare remote shows what's
actually addressable:

```bash
sudo -u kopds RCLONE_CONFIG=/var/lib/kopds/rclone.conf \
  rclone lsd <nas-remote>:
```

**Active ExecStart** (in the drop-in):

```
/usr/bin/rclone mount <nas-remote>:<library-share>/<library-dir> \
  /srv/calibre/library --read-only --vfs-cache-mode full --vfs-cache-max-size 2G \
  --dir-cache-time 10m --cache-dir /var/lib/kopds/rclone-cache --bwlimit 8M
```

Flag notes:

- `--vfs-cache-mode full` — **important.** rclone's FUSE layer has no POSIX advisory
  locking. Under `minimal`/`writes`, SQLite opening `metadata.db` can see a non-atomic
  read mid-page. Full mode pulls the whole file down before the driver touches it.
  Costs ~50MB extra RSS versus `minimal`.
- `--dir-cache-time 10m` — kept **below** the 30m sync interval, so the scanner's stat
  check is never served a stale mtime and silently skips a sync.
- `--read-only` — the mount is a source of truth, never written to.
- No `--allow-other` — kopds runs as `kopds` and is the only reader, so it isn't needed.
  Avoiding it also avoids depending on `user_allow_other` in `/etc/fuse.conf`.

### 4.2 Shared library — Nextcloud over WebDAV (dormant)

The original source: `<shared-remote>:<shared-library-path>`, reached over WebDAV through a
Cloudflare tunnel. The origin went offline and is expected to stay down for some months
from August 2026.

The remote definition is **preserved untouched** in `rclone.conf` — reactivating is a
drop-in edit, not a rebuild.

Symptom when the origin is down:

```
Failed to create file system for "<shared-remote>:<shared-library-path>":
read metadata failed: error code: 1033: 530
```

Cloudflare **1033 / HTTP 530** means Cloudflare accepted the request but found no
`cloudflared` connector at the origin. The tunnel is down at the far end — nothing to fix
on this VM. Confirm independently with `curl -sSI https://<origin-host>/`.

Note this failure only surfaces on **restart**. A long-established mount holds its
connection and can mask an origin that's been unreachable for weeks.

### 4.3 Switching between them

Everything lives in the drop-in, so switching is a one-file edit.

```bash
sudo systemctl edit kopds-library.service
```

Back to the shared library — replace the `ExecStart` lines with:

```ini
[Service]
ExecStart=
ExecStart=/usr/bin/rclone mount <shared-remote>:<shared-library-path> /srv/calibre/library \
  --read-only --vfs-cache-mode full --vfs-cache-max-size 2G --dir-cache-time 10m \
  --cache-dir /var/lib/kopds/rclone-cache --bwlimit 8M
```

The empty `ExecStart=` is **required** — it clears the original before setting the new one.
Without it systemd appends and refuses to start.

The same one-line edit covers a **credential or remote-name change** — repointing at a
renamed remote (e.g. after moving to a scoped NAS account, [§5.2](#52-nas-service-account-and-ssh-key))
touches only the remote name in `ExecStart`. The mountpoint, every mount flag, the library
path within the remote, and the dormant `<shared-remote>` remote all stay as they are. And
because the mountpoint doesn't move, the AppArmor override needs no change either — see
[§6](#6-apparmor), where that path-specificity is the whole point.

Verify in this order:

```bash
# 1. Config resolves and the remote can actually list the library.
sudo -u kopds RCLONE_CONFIG=/var/lib/kopds/rclone.conf rclone config show <remote>
sudo -u kopds RCLONE_CONFIG=/var/lib/kopds/rclone.conf \
  rclone lsd <remote>:<library-path> | tail -5

sudo systemctl daemon-reload

# 2. Verify the merge produced exactly ONE ExecStart line
systemctl show kopds-library.service -p ExecStart | tr ';' '\n' | grep -i argv

sudo systemctl restart kopds-library.service

# 3. Mount exists...
findmnt /srv/calibre/library

# 4. ...and is serving the right data.
sudo -u kopds ls -l /srv/calibre/library/metadata.db
sudo -u kopds ls "/srv/calibre/library/<some-author>/"
```

**Steps 3 and 4 are not the same check.** `findmnt` succeeding proves only that a FUSE
mount exists at that path — it says nothing about what's behind it. A wrong path, a wrong
remote, or a half-dead session can all present a mount that lists nothing. Confirm
`metadata.db` has a plausible size and read an actual author directory.

Then let kopds re-index (see [§7](#7-index-and-users)).

To revert entirely to the repo's stock unit: delete
`/etc/systemd/system/kopds-library.service.d/override.conf` and `daemon-reload`.

**Running both at once** is possible — a second mount unit at a different mountpoint plus
a second library entry. Note that a new mountpoint needs a matching AppArmor rule
([§6](#6-apparmor)).

---

## 5. Build Procedures

Reference for rebuilding from scratch. Skip if the system is working.

### 5.1 Tailscale on the VM

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
tailscale status | grep -i <nas-host>
```

**On tags:** a tag must be declared in the tailnet policy's `tagOwners` *before* any node
can claim it, or `tailscale up --advertise-tags=...` fails with
`requested tags [...] are invalid or not permitted`. Tags are optional — plain `tailscale up`
works under a default-permissive policy, but leaves the VM as a full tailnet peer: if it's
compromised, the attacker reaches every host on the tailnet, not just the library.

To lock it down, tag the VM `tag:kopds` and grant it exactly one destination — the NAS on
the SFTP port. **This tailnet is on the newer `grants` syntax, not the legacy `acls`
block, and the two cannot be mixed in one policy file** (a policy mixing them is rejected on
save). Applied and verified 2026-08-15:

```json
{
  "tagOwners": {
    "tag:kopds": ["autogroup:admin"]
  },

  "hosts": {
    "nas": "<nas-tailscale-ip>"
  },

  "grants": [
    // User-owned devices: unrestricted. autogroup:member EXCLUDES tagged
    // nodes, so this grant does not cover the VM.
    { "src": ["autogroup:member"], "dst": ["*"], "ip": ["*"] },

    // koserver VM: SFTP to the NAS only. Nothing else, in or out.
    { "src": ["tag:kopds"], "dst": ["nas"], "ip": ["tcp:<ssh-port>"] }
  ],

  "tests": [
    // Refuses to save if a future edit re-broadens the VM's access.
    { "src": "tag:kopds", "accept": ["nas:<ssh-port>"] }
  ]
}
```

Leave the existing `ssh`, `autoApprovers`, and any other blocks untouched — merge these
keys in, don't replace the file. The SSH policy typically uses `autogroup:self` (user-owned
devices only); a tagged node isn't user-owned, so the VM is excluded from SSH automatically
with no change needed.

Three things that differ from the legacy `acls` syntax and are easy to get wrong:

- **`"src": ["*"]` silently defeats tagging.** `*` keeps matching the VM even after it's
  tagged, so the restriction never takes effect. The user-devices grant *must* be scoped to
  `autogroup:member` (which excludes tagged nodes) for the tag to mean anything. This is the
  single load-bearing change.
- **Ports go in a separate `ip` field**, not appended to the destination as `host:port`.
  Write `tcp:<ssh-port>` to pin the protocol as well.
- **The `tests` block is evaluated on every save** and blocks a policy that fails it — cheap
  insurance that a later edit can't quietly remove the VM's NAS access. Keep it to the
  `accept` case; `deny` cases (e.g. `nas:22`) can fail on save because the test evaluates
  policy paths, not what's actually listening.

Use your DSM SFTP port, not 22. The NAS does **not** need its own tag — the grant points at
it by host alias. Save the policy first, then on the VM
`sudo tailscale up --advertise-tags=tag:kopds --force-reauth` and approve at the printed
URL. Confirm the tag applied with `tailscale status --json | grep -A3 -i '"Tags"'` (a tagged
node also renders with its FQDN rather than an owner email in `tailscale status`). Verify
reachability with the `/dev/tcp` probes in [§8](#8-troubleshooting) — **not** `tailscale
ping`, which probes the WireGuard layer beneath the grants filter and answers regardless of
policy. To roll back, revert the policy in the console (it keeps version history) and run
`sudo tailscale up --force-reauth` without the `--advertise-tags` flag.

The home LAN also has a subnet router. Don't use it for this — going direct to the NAS's
own tailnet node gives a point-to-point WireGuard path and keeps the subnet router out of the
data path for every book download.

### 5.2 NAS service account and SSH key

The VM holds a passwordless key. Whatever that key can reach is what an attacker who owns
the VM can reach, so the account it authorizes should be able to do exactly one thing:
read the library share. Do **not** reuse a general-purpose or administrator account.

DSM prerequisites (all in Control Panel):

- **Terminal & SNMP → Terminal** → Enable SSH service (note the port)
- **File Services → FTP → SFTP** → Enable SFTP (same sshd, same port)
- **User & Group → Advanced** → Enable user home service — without a home there's
  nowhere for `authorized_keys` to live and key auth silently fails. This must be on
  *before* you create the account, so the home is provisioned with it.

#### Create the service account

**Control Panel → User & Group → Create**, then through the wizard:

| Page | What to set |
|---|---|
| User info | Name, optional description, password. Uncheck the notification mail; leave email blank |
| Join groups | Leave at the default — **do not** add `administrators` |
| Assign shared folder permissions | **Read-only** on the library share (`<library-share>`). Leave every other share at No access |
| User quota | `Homes` can't be set to No access, so cap it — 1 GB is plenty for an `authorized_keys` file |
| Application permissions | **Deny all**, then allow **FTP** and **SFTP** only |

Staying out of `administrators` is what denies the account an interactive shell (see
below). The read-only permission on the share is what makes the restriction real —
`--read-only` on the rclone mount is a client-side flag and protects nothing on its own.

**Acceptance criteria.** Run these once the key is installed and the rclone remote exists
([§5.3](#53-creating-the-rclone-remote)) — they are what "correctly scoped" actually means:

```bash
# 1. Only the library share and the account's own home should be visible.
sudo -u kopds RCLONE_CONFIG=/var/lib/kopds/rclone.conf rclone lsd <nas-remote>:

# 2. Writes must be refused by the NAS, not just by rclone.
sudo -u kopds RCLONE_CONFIG=/var/lib/kopds/rclone.conf \
  rclone mkdir <nas-remote>:<library-share>/write-test
```

Expect exactly two entries from the first (`<library-share>` and `home`) and
`Permission denied` from the second. For contrast, a general-purpose account sees every
share on the NAS — backups, sync targets, and the other content shares included. That gap
is the entire point of the exercise.

#### Generate the key on the VM

```bash
sudo -u kopds mkdir -p /var/lib/kopds/.ssh && sudo -u kopds chmod 700 /var/lib/kopds/.ssh
sudo -u kopds ssh-keygen -t ed25519 -f /var/lib/kopds/.ssh/id_ed25519 -N ""
sudo cat /var/lib/kopds/.ssh/id_ed25519.pub
```

#### Install the public key on the NAS

The service account has no shell, so you can't log in as it to write its own
`authorized_keys`. Do it from a **separate DSM admin session**, then hand ownership over:

```bash
ssh -p <ssh-port> <dsm-admin>@<nas-host>

sudo mkdir -p /var/services/homes/<dsm-user>/.ssh
sudo tee /var/services/homes/<dsm-user>/.ssh/authorized_keys > /dev/null
# paste the public key, Enter, Ctrl-D

sudo chown -R <dsm-user>:users /var/services/homes/<dsm-user>/.ssh
sudo chmod 700 /var/services/homes/<dsm-user>/.ssh
sudo chmod 600 /var/services/homes/<dsm-user>/.ssh/authorized_keys
```

Three things differ from the usual key-install procedure:

- **The `chown` is mandatory and easy to forget.** sshd refuses an `authorized_keys` file
  the authenticating user does not own, and the file is created as root here.
- **`ssh-copy-id` will not work** — it needs a shell on the target.
- **Do not `chmod 755` the home.** DSM creates it `711`, which sshd accepts; the
  requirement is that the home not be group- or world-writable, not that it be `755`.

#### Verify with `sftp`, not `ssh`

```bash
sudo -u kopds sftp -i /var/lib/kopds/.ssh/id_ed25519 -P <ssh-port> <dsm-user>@<nas-host>
# at the sftp> prompt:  ls    then    quit
```

> **`ssh <dsm-user>@<nas-host> 'echo OK'` will FAIL on a correctly configured account, and
> this looks exactly like a broken setup.** Key auth succeeds — DSM's login banner prints,
> which only happens post-authentication — and *then* you get
> `Permission denied, please try again.` SFTP with the same key works fine.
>
> The mechanism is DSM's default group behaviour: an interactive shell is restricted to
> members of `administrators`, while the SFTP subsystem stays available to any account
> granted SFTP. Nothing was configured to produce this — no `rssh`, no chroot, no
> sftp-only group, and DSM's SFTP chroot setting was left alone. **Treat it as observed
> behaviour, not a guarantee**, and re-verify on your own DSM version rather than
> assuming it. Note that this is *not* a chroot: the account is confined by share
> permissions, not by a jail.

#### Revoke the old key

If the VM previously authenticated as a broader account, remove its key from that
account's `authorized_keys`. The file usually holds unrelated keys (a workstation, say),
so filter by the key comment rather than deleting the file:

```bash
ssh -p <ssh-port> <dsm-admin>@<nas-host>
grep -v '<vm-key-comment>' ~/.ssh/authorized_keys > /tmp/ak.new \
  && mv /tmp/ak.new ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
cat ~/.ssh/authorized_keys
```

Keep that admin session **open** until you've verified — a slip here can lock you out.

> **A plain `ssh -i <key> <olduser>@<nas-host>` does not prove revocation.** DSM offers
> password auth as a fallback, so the command prompts for a password and, if you supply
> one, succeeds — which reads as a failed revocation when it isn't. Disable the fallback:
>
> ```bash
> ssh -o PasswordAuthentication=no -o BatchMode=yes \
>   -i /var/lib/kopds/.ssh/id_ed25519 -p <ssh-port> \
>   <olduser>@<nas-host> 'echo SHOULD_FAIL'
> ```
>
> Expect `Permission denied (publickey,password)` with no prompt.

Two follow-ups worth doing deliberately:

- **This is a privilege reduction, not a key rotation.** If you reused the existing
  keypair, the same key material still authenticates — only the account it authorizes
  changed. Key exposure is unchanged. True rotation is a separate step: generate a new
  keypair, install it, repoint `key_file` in `rclone.conf`, then revoke the old one.
- **Remove superseded private keys from the VM.** If a key was renamed or replaced during
  the work, the old file can linger in `/var/lib/kopds/.ssh/`. Check with
  `sudo ls -l /var/lib/kopds/.ssh/` and delete anything no longer referenced by
  `rclone.conf`.

### 5.3 Creating the rclone remote

```bash
sudo -u kopds RCLONE_CONFIG=/var/lib/kopds/rclone.conf rclone config
```

`n` → name → `sftp` → then:

| Prompt | Answer |
|---|---|
| `host` | `<nas-host>` |
| `user` | `<dsm-user>` |
| `port` | your DSM SSH port |
| `pass` | *(blank)* |
| `key_pem` | *(blank)* |
| `key_file` | `/var/lib/kopds/.ssh/id_ed25519` |
| `key_file_pass` | *(blank)* |
| `pubkey_file` | *(blank)* |
| `key_use_agent` | `false` |
| `use_insecure_cipher` | `false` |
| `disable_hashcheck` | `false` |
| advanced config | `n` |

The resulting stanza carries three fields you never typed:

```ini
shell_type = unix
md5sum_command = md5sum
sha1sum_command = sha1sum
```

rclone probes the remote for hash support during config and writes these itself — **it
does so even though the account has no interactive shell.** Harmless, but confusing if
you're diffing a config against this table and wondering where they came from.

Test before touching systemd:

```bash
sudo -u kopds RCLONE_CONFIG=/var/lib/kopds/rclone.conf \
  rclone lsd <nas-remote>:<library-share>/<library-dir>
```

### 5.4 Repointing the mount

See [§4.3](#43-switching-between-them).

---

## 6. AppArmor

The `fusermount3` profile confines the setuid helper that FUSE mounts go through.
Because kopds-library runs rclone as the unprivileged `kopds` user, that helper is on the
critical path — **if the profile is wrong, the mount fails.**

**History worth knowing:** the profile was disabled in June 2026 after repeated denials.
The original custom rule granted only `mount`, but the failures were `umount` and two
capabilities. Adding those three rules fixed it; the profile has been enforcing cleanly
since 2026-08-15.

**The override** at `/etc/apparmor.d/local/fusermount3`:

```
# rclone Calibre library mount at /srv
mount fstype=@{fuse_types} options=(nosuid,nodev,ro) -> /srv/calibre/library/,
umount /srv/calibre/library/,
capability setuid,
capability dac_override,
```

This file is the load-bearing piece and is **path-specific**. Moving the mountpoint
requires updating it in the same change, or the mount fails with a non-obvious cause.

The converse is just as useful: changes that *don't* move the mountpoint need nothing here.
Swapping the NAS credential and renaming the rclone remote ([§5.2](#52-nas-service-account-and-ssh-key))
left this override untouched, because the profile keys on the path, not on what's mounted there.

Distro profiles carry `include if exists <local/fusermount3>`, so the override survives
package upgrades and the shipped conffile stays pristine — no more dpkg conffile prompts.

**Checking state:**

```bash
sudo aa-status | grep -i fusermount        # should list under enforce
ls -l /etc/apparmor.d/disable/             # a symlink here = profile disabled
sudo journalctl -k -b | grep -i 'apparmor.*DENIED'
```

**Debugging denials** — always in complain mode, which logs but blocks nothing:

```bash
sudo rm -f /etc/apparmor.d/disable/fusermount3
sudo aa-complain /etc/apparmor.d/fusermount3
sudo systemctl restart kopds-library.service
sudo journalctl -k -b | grep -i fusermount | tail -20
```

Anything logged `ALLOWED` in complain mode is a rule you're missing. Add it to the
override, reload, repeat. When a full restart cycle is clean:

```bash
sudo aa-enforce /etc/apparmor.d/fusermount3
```

Escape hatch — disable and reload:

```bash
sudo ln -s /etc/apparmor.d/fusermount3 /etc/apparmor.d/disable/
sudo apparmor_parser -R /etc/apparmor.d/fusermount3
```

> **AppArmor problems don't surface immediately.** An established mount keeps working with
> a broken profile; the denial only bites on the *next* mount attempt. "The library still
> works" is not evidence the profile is correct — restart the unit and check the log.

**On dpkg conffile prompts** (`Modified (by you or by a script) since installation`):
`D` shows the diff, `Z` opens a shell. Answer **Y** to take the maintainer's version — your
old file is saved as `.dpkg-old` — then re-add customisations to `local/`. Answering `N`
keeps yours and saves theirs as `.dpkg-dist`, but risks an outdated profile against a newer
binary. Either way nothing is lost.

---

## 7. Index and Users

`kopds.db` holds **both** the derived book index and the user accounts:

| Table group | Contents | Safe to clear? |
|---|---|---|
| `books`, `authors`, `series`, `tags`, `formats`, `*_link`, `books_search*` | Derived from `metadata.db` | Yes — rebuilt on scan |
| `sync_state` | mtime/size/timestamp of the last seen `metadata.db` | Yes |
| `users` | **Accounts and passwords** | **No — not recoverable** |

So never delete `kopds.db` wholesale to force a reindex; you'd wipe the logins. Use the
`kopds reindex` command instead (below), which preserves `users` in every mode.

**Switching libraries usually needs no manual reset.** The scanner compares `metadata.db`
mtime and size against `sync_state`, and on a mismatch **replaces** rather than merges.
Verified 2026-08-15: 1011 books (shared) → 593 (personal), no stale book rows, all 4 users
intact.

**But the incremental read is threshold-gated, and a swap can leave the index dirty:**

- The scanner only re-reads books whose Calibre `last_modified` is newer than the last
  sync's stored timestamp. If the *new* library's books carry older timestamps (common for
  an established library), they are never inserted — leaving a partial or near-empty
  catalogue even though a sync "ran".
- Older binaries pruned book rows on a swap but did **not** garbage-collect the `authors`,
  `tags`, and `series` rows left behind, so the OPDS feed served entries from the previous
  library that resolved to nothing. The current binary GCs those orphans during prune.

When a swap leaves the catalogue stale, partial, or showing old authors/tags/series, force
a clean rebuild with `reindex` (below) rather than trusting the incremental sync.

**Forcing a full re-index** (requires the updated binary — a build carrying the `reindex`
subcommand). Stop the server first so it isn't racing the background worker:

```bash
sudo systemctl stop kopds
sudo -u kopds env KOPDS_DATABASE_PATH=/var/lib/kopds/kopds.db \
  KOPDS_LIBRARY_PATH=/srv/calibre/library /opt/kopds/kopds reindex
sudo systemctl start kopds
```

`reindex` clears `sync_state` and runs one full sync — re-reading every book, pruning what's
gone, and GCing orphaned authors/tags/series — while preserving `users`. Add `--purge` to
empty the derived tables first (corrupt-index recovery) or `--vacuum` to reclaim disk after
a large prune. Watch it with `journalctl -u kopds -f`.

Check state:

```bash
sudo -u kopds sqlite3 /var/lib/kopds/kopds.db \
  'SELECT COUNT(*) FROM books; SELECT COUNT(*) FROM users; SELECT * FROM sync_state;'
```

Back up before experimenting:

```bash
sudo -u kopds sqlite3 /var/lib/kopds/kopds.db ".backup '/var/lib/kopds/kopds.db.bak-$(date +%Y%m%d)'"
```

**Cover cache** (`/var/lib/kopds/cache/`) is an LRU keyed to the previous library and will
age out on its own. To clear it after a switch:

```bash
sudo systemctl stop kopds && sudo rm -rf /var/lib/kopds/cache/* && sudo systemctl start kopds
```

Startup logs `Loaded existing image cache entries count=N` — expect `0` after clearing.

**User management** (besides these and `reindex` above, the binary has no other subcommands):

```bash
/opt/kopds/kopds create-user <username>
/opt/kopds/kopds delete-user <username>
/opt/kopds/kopds change-password <username>
# add --password-stdin to read the password from stdin
```

---

## 8. Troubleshooting

**Books list but every download 404s** → the mount is down. `findmnt /srv/calibre/library`,
then `systemctl status kopds-library`.

**`error code: 1033: 530`** → Cloudflare tunnel down at the origin. Only affects the
dormant `<shared-remote>` remote. Nothing to fix on this VM. See [§4.2](#42-shared-library--nextcloud-over-webdav-dormant).

**Catalogue stale, partial, or listing old authors/tags/series after a library switch** →
the incremental scan is threshold-gated and doesn't always fully rebuild across a swap.
Force a clean rebuild with `kopds reindex`. See [§7](#7-index-and-users).

**`Permission denied` listing the mount, even as root** → expected. FUSE mounts are private
to the mounting user. Use `sudo -u kopds ls ...`.

**`fusermount3: entry for /srv/calibre/library not found in /etc/mtab`** → cosmetic. It's the
`ExecStartPre` unmount finding nothing to unmount on a clean boot. Non-fatal; systemd proceeds.

**Mount fails after an AppArmor change** → [§6](#6-apparmor). Check
`journalctl -k -b | grep DENIED`.

**`Couldn't find type of fs for "<remote>"`** → `sudo` dropped the environment and rclone
read root's config. Pass `RCLONE_CONFIG=/var/lib/kopds/rclone.conf` explicitly.

**`directory not found` from `rclone lsd`** → SFTP paths are relative to the share root, not
the filesystem root. List the bare remote (`rclone lsd <remote>:`) to see real paths.

**`ssh <dsm-user>@<nas-host>` gives `Permission denied` but SFTP works** → expected, not
broken. The service account is deliberately non-admin on DSM, which denies an interactive
shell while leaving SFTP available. The banner printing before the denial means key auth
already succeeded. Verify with `sftp`, never `ssh`. See [§5.2](#52-nas-service-account-and-ssh-key).

**A revoked key still connects** → you're hitting DSM's password-auth fallback, not a
failed revocation. Re-test with `-o PasswordAuthentication=no -o BatchMode=yes` and expect
`Permission denied (publickey,password)` with no prompt. See [§5.2](#52-nas-service-account-and-ssh-key).

**Reconnecting to the NAS** — a mount restart forces a fresh SFTP session:

```bash
sudo systemctl restart kopds-library.service
```

**Verifying the Tailscale grants restriction** ([§5.1](#51-tailscale-on-the-vm)) — test from
*inside* the VM with bash's built-in `/dev/tcp`, which needs nothing installed. **Do not use
`tailscale ping`**: it probes the disco/WireGuard layer *beneath* the packet filter that
grants enforce, so it succeeds regardless of policy and tells you nothing about the
restriction.

```bash
# Control: the one destination the VM is allowed → REACHABLE, instantly.
timeout 5 bash -c 'exec 3<>/dev/tcp/<nas-tailscale-ip>/<ssh-port>' && echo REACHABLE || echo BLOCKED

# Any other tailnet host/port (pick a peer with a known open port) → BLOCKED,
# after a ~5s timeout.
timeout 5 bash -c 'exec 3<>/dev/tcp/<other-tailnet-ip>/<other-port>' && echo REACHABLE || echo BLOCKED
```

A **~5s timeout** (not an instant connection refusal) is the signature of the grants
default-deny working. If the second probe returns `REACHABLE`, the grant isn't filtering —
most likely the tag didn't apply or the policy didn't save; re-check with
`tailscale status --json | grep -A3 -i '"Tags"'`. Note that `tailscale status` still *lists*
your other nodes even when the restriction is working — that's just the netmap; the grants
block the actual connections, not the listing.

**Health check after any change:**

```bash
systemctl is-active kopds kopds-library
findmnt /srv/calibre/library
sudo -u kopds ls -l /srv/calibre/library/metadata.db
sudo journalctl -k -b | grep -i 'apparmor.*DENIED'
```

---

## 9. Gotchas

Things that cost time during the build and the later credential hardening.

**`sudo cat > /path/file` doesn't work.** The redirect is performed by *your* shell before
`sudo` runs, so the write happens as your user. Use:

```bash
sudo tee /path/file > /dev/null <<'EOF'
...
EOF
```

**`usermod` refuses while the account has running processes** (`exit=8`,
`user kopds is currently used by process NNNN`). Identify with
`systemctl status <pid>`, stop the unit, run `usermod`, start it again.

**`kopds` had a declared-but-nonexistent home** (`/home/kopds`), standard for a hardened
service account. Repointed to `/var/lib/kopds` with `usermod -d`, where the config, key and
DB already live. The `nologin` shell doesn't block `sudo -u kopds <cmd>` — that execs the
binary directly rather than going through a login shell.

**Root can't read the FUSE mount.** Not a permissions bug; FUSE mounts are private to the
mounting user unless `--allow-other` is set.

**Synology SFTP paths ≠ filesystem paths.** Shares appear flattened at the SFTP root with
no `/volumeN` prefix.

**DSM's SSH port may be moved off 22**, and SFTP shares it — one setting covers both.

**A non-admin DSM account gets SFTP but no shell.** That's the desired end state for a
service account, but it breaks the reflexive `ssh <user>@<host> 'echo OK'` verification and
makes `ssh-copy-id` unusable — it needs a shell to write `authorized_keys`. Install the key
from an admin session and `chown` it to the service account afterwards
([§5.2](#52-nas-service-account-and-ssh-key)).

**Synology homes live under `/var/services/homes/<user>`**, not `/home/<user>`, and DSM
creates them mode `711` — which sshd accepts. Don't "fix" that to `755`.

**The post-quantum key exchange warning** on SSH to the NAS is DSM shipping an older
OpenSSH. Irrelevant inside a WireGuard tunnel.

---

## 10. Maintenance

**Reboot behaviour** (verified 2026-08-15): both units come up unattended in the right
order. `kopds-library` starts first and `kopds` waits for it — roughly 30s while rclone
establishes the SFTP session over Tailscale. Reboot after any significant change; it's the
only real test that the stack survives without hand-holding.

**Resource budget** on the e2-micro (1GB RAM):

| Process | RSS |
|---|---|
| kopds | ~15MB idle, grows during scan |
| rclone (`--vfs-cache-mode full`) | ~60MB |
| rclone (`minimal`, historical) | ~12MB |
| tailscaled | ~30MB |
| Caddy | ~25MB |

Comfortable, but swap is still worth having for the initial scan. If `--vfs-cache-max-size`
is raised above 2G, watch RSS — disk isn't the only cost.

**Network egress:** NAS → VM is ingress and free. VM → client devices counts against the
free tier's ~1GB/month. EPUBs are negligible; heavy PDF sessions are not.

**Confirming the VFS cache is working:** `systemctl status kopds-library` reports
`vfs cache: objects 1 ... total size 2.6Mi` — that's `metadata.db` cached whole, which is
exactly the torn-read protection `--vfs-cache-mode full` is there to provide.

**When the shared library returns:** edit the drop-in ([§4.3](#43-switching-between-them)),
`daemon-reload`, restart, let the scanner re-index. The `<shared-remote>` remote needs no
changes.
