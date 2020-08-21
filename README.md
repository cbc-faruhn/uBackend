# uBackend
This Application provides a basic RESTful API to directly interact with databases incl. (two-factor) authorization.

## Configuration

### Configuration File / Basic Structure

### Database

### Users

#### Local Users

#### Database Users

#### Default admin user

If no admin user (either username "admin" or any user having the role "admin") exists, an admin user is created at start-up of uBackend and credential data is printed to the terminal.
    
Make sure to write down the credentials and add the QR-Code/Token Secret to your favorite Authenticator app!

To prevent this behavior, uBackend must be started with the Argument --noAdmin.

### Authentication

#### Types of Authentication

* Basic Authentication
* Bearer Token
* Session (Bearer Token based)

#### Two-factor Authentication

### Access Control

Access can be controlled through Javascript Objects with a certain structure. To register a new access control object, it simply needs to be added to the `access` object through a property. The property's name needs to be equal to the table name or the name of the custom endpoint that you want to control access to.

You can simply use the boilerplate code from below as a starting point.

To be effective, the javascript code needs to be saved in a .js file in the folder "access".

#### Basic Access Control Object Structre

```javascript
// tableName = the physical name of the table in the database or the name of the custom endpoint
// alternatively use access['tableName'] if table name contains characters that are contrary to Javascript tokens
access.tableName = {
    // method must be one of: all, post, put, get, patch, delete or default
    method: {
        // array of all the roles, that are principially allowed to access
        // (if user doesn't have any of the mentioned roles, access is rejected)
        // (an empty array equals to the array ['public'])
        roles: [],

        // filter must return an array with at least one element to not filter-out the query;
        // filter must return null, [] or nothing (no return statetment) to filter-out the query
        // (alternatively it can throw an error to cancel the transaction)
        //
        // For:
        //     - get: return the array of allowed / to be returned records
        //     - post, put, patch: return the array of allowed changes
        //     - delete: return null, [] or nothing (no return statetment) if deletion is not allowed
        //
        // As you return an array of allowed records / changes,
        // you can also freely manipulate what is to be returned!
        filter: function(req, affectedRecords, changes) {
            return affectedRecords;
        }
    }
}
```

Following rules apply:
* Multiple methods can be added simply as further properties of the object.
* The method "default" will be tried if the actual method (post, put, get, patch, delete) hasn't been found.
* If you use "all" as a method, all other methods for that table are ignored (works like a catch-all).

#### Example / Boilerplate

Attention: this boilerplate code assumes that the table that is to be access controlled has at least five columns:
* state
* updated_by
* updated_at
* created_by
* created_at

Boilerplate code:
```javascript
// tableName = the physical name of the table in the database or the name of the custom endpoint
// alternatively use access['tableName'] if table name contains characters that are contrary to Javascript tokens
access.tableName = {
    // CREATE (HTTP method: POST)
    post: {
        // list all the roles, that are principially allowed to access
        roles: ['internal'],

        // returns a filtered array of allowed records (records can also be manipulated)
        // (alternatively it can throw an error to cancel the transaction)
        filter: function(req, affectedRecords, changes) {
        
            // adjust / override creation and update information
            for (let i = 0; i < changes.length; i++) {                
                changes[i].created_by = req.user.username;
                changes[i].created_at = new Date().getTime();
                
                changes[i].updated_by = req.user.username;
                changes[i].updated_at = new Date().getTime();
            }
        
            return changes;
        }
    },

    // CREATE (HTTP method: PUT)
    put: {
        // list all the roles, that are principially allowed to access
        roles: ['internal'],

        // returns a filtered array of allowed records (records can also be manipulated)
        // (alternatively it can throw an error to cancel the transaction)
        filter: function(req, affectedRecords, changes) {
            throw {
                message: 'PUTting a record is currently not yet allowed',
                status: 403
            }
        }
    },

    // READ (HTTP method: GET) -- if user has the role admin he/she will receive all records, otherwise they are filtered for "state=published"
    get: {
        // list all the roles, that are principially allowed to access
        roles: ['public'],

        // returns an array of allowed/filtered records to be sent to the client (records can also be manipulated)
        // (alternatively it can throw an error to cancel the transaction)
        filter: function(req, affectedRecords) {
            let allowedRecords = [];

            for (let i = 0; i < affectedRecords.length; i++) {
                // take over record if user is an admin OR if it has been published already
                if (req.user.roles.indexOf('admin') >= 0 || affectedRecords[i].state == 'published') {
                    allowedRecords.push(affectedRecords[i]);
                }
                
                // take over record if the user has created that record
                else if (affectedRecords[i].created_by == req.user.username) {
                  allowedRecords.push(affectedRecords[i]);
                }
            }

            if (affectedRecords.length > allowedRecords.length && allowedRecords.length == 0) {
                throw {
                    message: 'all records have been filtered for security reasons',
                    status: 403
                }
            }

            return allowedRecords;
        }
    },

    // UPDATE (HTTP method: PATCH)
    patch: {
        // list all the roles, that are principially allowed to access
        roles: ['internal'],

        // filter needs to return the (filtered, allowed) changes... worstcase: [] (empty array) means that no changes are allowed
        // (therefor the changes array and its objects can also be manipulated)
        // (furthermore it can throw an error to cancel the transaction)
        filter: function(req, affectedRecords, changes) {
        
            // for non-admin users only allow changes to records that were created by the current user
            if (req.user.roles.indexOf('admin') == -1) {
                let foreignAffectedRecordFound = false;
                for (let i = 0; i < affectedRecords.length && !foreignAffectedRecordFound; i++) {
                    foreignAffectedRecordFound = (affectedRecords[i].created_by != req.user.username);
                }
                
                if (foreignAffectedRecordFound) {
                    throw {
                        message: 'PATCH would affect records that the current user is not allowed to change',
                        status: 403
                    }
                }
            }

            // filter out changes to the id (primary key) to not endanger existing deep-links
            // and filter out changes to the creation information for audit integrity
            for (let i = 0; i < changes.length; i++) {
                for (field in changes[i]) {
                    if (field == 'id' || field == 'created_by' || field == 'created_at') {
                        delete changes[i][field];
                    }
                }
            }
            
            // adjust / override update information
            for (let i = 0; i < changes.length; i++) {
                changes[i].updated_by = req.user.username;
                changes[i].updated_at = new Date().getTime();
            }

            return changes;
        }
    },

    // DELETE (HTTP method: DELETE) is for admin only (authorization required)
    delete: {
        roles: ['admin'],

        // filter must return an array with at least one element to not filter-out the query;
        // filter must return null, [] or nothing (no return statetment) to filter-out the query
        // (alternatively it can throw an error to cancel the transaction)
        filter: function(req, affectedRecords) {
            return [true];
        }
    }
}
```

### Roles

#### Predefined Roles
uBackend comes with four internally used / allocated roles for providing functionality:

| Role | Description |
| --- | --- |
| public | any user that has not been authorized (not to be manually assigned to the user record in configuration or in the database) |
| internal | any user that has been authorized (virtually added at authorization time; no need to assign manually to the user record in configuration or in the database) |
| admin | Administrative access, incl. remote reload of configuration |
| interactive | any user that is allowed to create a session bearer token that is self-prolonging with each transaction within the validity |

#### Defining own roles

### Custom API Endpoints
