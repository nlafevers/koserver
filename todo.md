# To Do List

At the beginning of work on each step, prior to making any changes to any code, place a mark in the box next to that step (eg. [-]) so that it is clear what work was being done if there is a sudden interruption. After the completion of each step, git commit the changes with a descriptive message, then update this document to show the current state of progress by checking the box next to the step (eg. [x]).



## HTTP Routing

- [ ] Current status: KOPDS uses `go-chi/chi/v5` and KOSYNC uses `net/http.ServeMux`.  Desired outcome: both KOPDS and KOSYNC use the same http routing, and since minimizng dependencies is preferred, that should be the standard Go router that KOSYNC uses unless there is some reason KOPDS cannot be refactored to use it.  Additional tasks: update the `README.md` for both KOPDS and KOSYNC to reflect any changes made in this step.



## User Management 

- [ ] Current behavior: `create-user` overwrites/updates an existing user for both KOPDS and KOSYNC.  Desired behavior: `create-user` fails if the user already exists for both KOPDS and KOSYNC.  Additional tasks: update usage documentation in the `README.md` for both KOPDS and KOSYNC if, and only if, the overwrite/update functionality is currently mentioned.



## Logging

- [ ] Develop a matrix or table showing all possible commands/requests/inputs and what log level is expected for it for both KOPDS and KOSYNC
- [ ] Current behavior: when deployed as a Docker container with `LOG_LEVEL=INFO`, `docker logs -f <app>` does not show entries for CLI user management events.  Desired behavior: when deployed as a Docker container, `docker logs -f <app>` does show entries for CLI user management events.
- [ ] Current behavior: when deployed as a Docker container with `LOG_LEVEL=INFO` or `LOG_LEVEL=DEBUG`, KOSYNC does display successful progress updates (`PUT`) but does not display successful get progress requests (`GET`)(unsuccessful updates and requests are both logged).  Desired behavior: when deployed as a Docker container with `LOG_LEVEL=INFO` of `LOG_LEVEL=DEBUG`, KOSYNC does display both successful and unsuccessful progress updates (`PUT`) and get progress requests (`GET`).
- [ ] Current behavior: when deployed as a Docker container with `LOG_LEVEL=INFO`, KOSYNC does not display every successful HTTP request, but KOPDS does (in a different format than the other log entries).  What is KOPDS doing differently, and why do the entries of HTTP requests appear in a different format than entries for starting up the container or creating a new user?
