# uBackend
This Application provides a basic RESTful API to directly interact with databases incl. (two-factor) authorization.

## Configuration

### Configuration File / Basic Structure

### Database

### Users

#### Local Users

#### Database Users

### Authentication

#### Types of Authentication

* Basic Authentication
* Bearer Token
* Session (Bearer Token based)

#### Two-factor Authentication

### Roles

#### Predefined Roles
uBackend comes with four internally used / allocated roles for providing functionality:

| Role | Description |
| --- | --- |
| public | any user that has not been authorized (not to be manually assigned to the user record in configuration or in the database) |
| internal | any user that has been authorized (virtually added at authorization time; no need to assign manually to the user record in configuration or in the database) |
| admin | Administrativ access, incl. remote reload of configuration |
| interactive | any user that is allowed to create a session bearer token that is self-prolonging with each transaction within the validity |

#### Defining own roles

### Access Control

#### Basic Structure of an access object



#### Role-based Access Control

#### Filtering of to be returned records

#### Filtering of to be changed/added records

### Custom API Endpoints
