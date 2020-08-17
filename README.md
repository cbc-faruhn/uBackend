# uBackend
This Application provides a basic RESTful API to directly interact with databases incl. (two-factor) authorization.

## Roles
uBackend comes with four internally used / allocated roles for providing functionality:

| Role | Description |
| --- | --- |
| admin | Administrativ access, incl. remote reload of configuration |
| internal | any user that has been authorized (automatically added at authorization time) |
| interactive | any user that is allowed to create a session bearer token |
| public | any user that has not been authorized |
