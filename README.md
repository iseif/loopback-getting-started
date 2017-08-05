# loopback-getting-started

A simple tutorial for getting started with the Loopback framework
This tutrial will cover:

  - Setting up a working web applicaiton with fornend and backend
  - Set up models for the web application
  - Defining the models relations and ACLs
  - Authentication and Authrization features
  - 3rd party login support
  - Connecting the models to a real DB
  - Angular frotnend that communicate with the backend server

### Set up

Please make sure you have the following:
  - [Node.js](https://nodejs.org/)
  - [NPM](https://www.npmjs.com/)
  - loopback global npm libraries

Install the follwoing loopback dependencies

```sh
$ npm install -g loopback-cli
$ nnpm install -g strongloop
$ install -g loopback-sdk-angular-cli
```

### 1. Goal
We will create a simple web application (MyNotes) that will enable users to create and manage personal notes with the abaility to archive them once they are irelevant.

### 2. Generate the application

```sh
$ mkdir MyNotes
$ cd MyNotes
$ lb
? What's the name of your application? MyNotes
? Which version of LoopBack would you like to use? 3.x (current)
? What kind of application do you have in mind? api-server (A LoopBack API server with local User auth)
$ node server/server.js
```

Now we have a working server. You can explore the main page by opening http://localhost:3000/
Loopback comes with a builtin User model for handling users.
All models in loopback comes with a full REST representation out of the box.
You can explore and interact with your models at any time by opening the Swagger explorer at http://localhost:3000/explorer

Try and create a user by using the POSTthod on the /Users endpoint and giving the following object:
```json
{
    "email": "your@email.com",
    "password": "12345"
}
```
Notice that we got a valid response from the server stating the user was created.

Lets try to get all the users we have by running the GET method on the /Users endpoint.
Notice we got a 401 error code - which makes sense, as users data is sensative and and such no once can read it without the proper access - we will disucess ACLs later on.

### 3. Attach DB

Loopback comes with a in-memory DB for devloping purpuses which makes it easier to start working right away.
Connecting a real DB is as easy as a cli command.

> You can skip this step for now if you don't have a test DB set up

For the purposed of this tutrial I will connect a MSSQL DB:
```sh
$ lb datasource
? Enter the datasource name: sql
? Select the connector for sql: Microsoft SQL (supported by StrongLoop)
? Connection String url to override other settings (eg: mssql://username:password@localhost/database):
? host: testmyapp.database.windows.net
? port: 1433
? user: myapp
? password: *********
? database: testMyApp
? Install loopback-connector-mssql@^2.5 Yes
```

Notice that the `server/datasources.json` file has a new "sql" object that describes the DB conneciton we just set up.

Since MSSQL requires an encrypted connection, we need to declare it in the mathcing object by adding 
```json
"options": 
{
      "encrypt": true
}
```

### 4. Auto create DB schema

Loopback can auto create all the schemas that describes our models. This is very convenient for developing since we don't want to create the schemas ourselves.

To enable this feature, we need to add the following file:
`server/boot/autoupdate.js`
```javascript
'use strict';

module.exports = function (app) {
  var datasources = require('../datasources.json');

  function autoUpdateAll() {
    Object.keys(datasources).forEach(function (key) {
      var DS = app.dataSources[key];
      if (DS.connected) {
        DS.autoupdate(function (err) {
          if (err) throw err;
          console.log('DS ' + key + ' updated');
        });
      } else {
        DS.once('connected', function () {
          DS.autoupdate(function (err) {
            if (err) throw err;
            console.log('DS ' + key + ' updated');
          });
        });
      }
    });
  }

  autoUpdateAll();
};
```

> In production you should disable this feature as it changes schemas even if they alredy exists, which can lead to data loss!

Rerun the application and add the user again (it was deleted from last time as it was saved in memory)
Open your prefrence of DB explorer and notice that a lof of tables were auto created for you and that the user table has 1 entry of the user you just created.

### 5. Create models

As our application holds notes for users, we need to have a representation of a note in the DB - a loopback model:
 - A note will belong to a single user
 - A note can be archivable
 - A note will be composed of a title and text content.
 - A note will have a creation date

Lets jump right ahead and create the model:
```sh
$ lb model
? Enter the model name: note
? Select the datasource to attach note to: sql (mssql)
? Select model's base class PersistedModel
? Expose note via the REST API? Yes
? Custom plural form (used to build REST URL):
? Common model or server only? server
```

Now lets add its properties:
```sh
Let's add some note properties now.

Enter an empty property name when done.
? Property name: id
   invoke   loopback:property
? Property type: string
? Required? Yes
? Default value[leave blank for none]:

Let's add another note property.
Enter an empty property name when done.
? Property name: username
   invoke   loopback:property
? Property type: string
? Required? Yes
? Default value[leave blank for none]:

Let's add another note property.
Enter an empty property name when done.
? Property name: created
   invoke   loopback:property
? Property type: date
? Required? Yes
? Default value[leave blank for none]:

Let's add another note property.
Enter an empty property name when done.
? Property name: archived
   invoke   loopback:property
? Property type: boolean
? Required? Yes
? Default value[leave blank for none]: false

Let's add another note property.
Enter an empty property name when done.
? Property name: title
   invoke   loopback:property
? Property type: string
? Required? Yes
? Default value[leave blank for none]:

Let's add another note property.
Enter an empty property name when done.
? Property name: content
   invoke   loopback:property
? Property type: string
? Required? Yes
? Default value[leave blank for none]:
```

Notice that under `server/models` we have 2 files that represnt the Note model:
 - `note.js` - here we can customize the behaviour of the model
 - `note.json` - JSON description of the model

We want that the `id` field of our model to be the PK and make it auto generated with UUID value. To do that, edit the `server/models/note.json` file by changing the `id` property to be as follows:
```json
"id": 
{
    "type": "string",
    "required": true,
    "id": true,
    "generated": false,
    "defaultFn": "uuidv4"
}
```

We want that the `created` field of our model to be auto generated with the current timestamp. To do that, edit the `server/models/note.json` file by changing the `created` property to be as follows:
```json
"created": 
{
    "type": "date",
    "required": true,
    "defaultFn": "now"
}
```

Notice also that the `server/model-config.js` file declares our model and connects it to the DB we created earlier.

Lets connect the pre defined models in that file to our DB as well by changing the "dataSource" property to "sql".

Restart the server and open the explorer. You will now see the notes model there. You can play around with it by adding, editing and deleting notes. All of the changes you will make will be persisted to the newly created table in the DB.

### 6. Add archive method to the Note model

We want users to be able to archive their notes. We can do that by adding custom methods to the model.

Please add the following to the exports function in `server/models/note.js`:
```javascript
  Note.prototype.archive = function (cb) {
    var note = this;
    console.log('archving note', note.id);
    var delta = {archived: true};
    note.patchAttributes(delta)
      .then(function () {
        return cb();
      })
      .catch(function (err) {
        var errToSend = new Error();
        errToSend.status = 500;
        errToSend.message = 'Failed archiving note';
        return cb(errToSend);
      });
  };
  Note.remoteMethod(
    'archive',
    {isStatic: false}
  );
```

We added a method called `archive` to the Note model, which operats on a specific instance of it. When the archive method is called on a note, it will change its `archived` property to true.

> we can achieve the change of property by using the PATCH method on the Note model directly, but for the sake of this tutrial, we added a custom method for it

Open the explorer and notice the `archive` method. Try it out on one of your already created notes

### 7. Extned user models

As mentioned before, loopback comes with predefined models for handling users and authentication.
For purposes outside of the scope of this tutrial, we need to extend them.

```sh
$ lb model
? Enter the model name: user
? Select the datasource to attach user to: sql (mssql)
? Select model's base class User
? Expose user via the REST API? Yes
? Custom plural form (used to build REST URL):
? Common model or server only? server
Let's add some user properties now.

Enter an empty property name when done.
? Property name: email
   invoke   loopback:property
? Property type: string
? Required? Yes
? Default value[leave blank for none]:
```

```sh
$ lb model
? Enter the model name: accessToken
? Select the datasource to attach accessToken to: sql (mssql)
? Select model's base class AccessToken
? Expose accessToken via the REST API? No
? Common model or server only? server
Let's add some accessToken properties now.

Enter an empty property name when done.
? Property name: id
   invoke   loopback:property
? Property type: string
? Required? Yes
? Default value[leave blank for none]:
```

Now lets remove the old model configs from `server/model-config.js` by removing the "User" and "AccessToken" keys

Edit the `server/models/user.json` file by changing the `email` property to be as follow:
```json
"email": 
{
    "type": "string",
    "required": true,
    "id": true,
    "generated": false
}
```

Edit the `server/models/access-token.json` file by changing the `id` property to be as follow:
```json
"id":
{
    "type": "string",
    "required": true,
    "id": true,
    "generated": false,
    "defaultFn": "uuidv4"
}
```

### 8. Define relations

Now that we have the user and note models, we need to define the relations between them:
 - a user had many notes
 - a note belongs to one user

```sh
$ lb relation
? Select the model to create the relationship from: user
? Relation type: has many
? Choose a model to create a relationship with: note
? Enter the property name for the relation: notes
? Optionally enter a custom foreign key:
? Require a through model? No
```

```sh
$ lb relation
? Select the model to create the relationship from: note
? Relation type: belongs to
? Choose a model to create a relationship with: user
? Enter the property name for the relation: user
? Optionally enter a custom foreign key:
```

```sh
$ lb relation
? Select the model to create the relationship from: accessToken
? Relation type: belongs to
? Choose a model to create a relationship with: user
? Enter the property name for the relation: user
? Optionally enter a custom foreign key:
```

```sh
$ lb relation
? Select the model to create the relationship from: user
? Relation type: has many
? Choose a model to create a relationship with: accessToken
? Enter the property name for the relation: accessTokens
? Optionally enter a custom foreign key:
? Require a through model? No
```

Check out the models json files and notice the added relations section.

Open the explore and notice that new resources were added to note and user that describe the relations. For example, a user can directly ask for all of his notes by the `\users\:id\notes` endpoint

### 9. Define ACLs

Our users privacy is very important to us. As such, we don't want to expose a users notes to other users.
Loopback comes with build in ACL mechanism to control who can access what resource.

Lets define the following ACLs:
 - disable access to all models
 - allow users to write notes
 - allow notes to be editable by their owner only
 - allow notes to be archived by their owner only
 - allow users to get their own user data

```sh
$ lb acl
? Select the model to apply the ACL entry to: (all existing models)
? Select the ACL scope: All methods and properties
? Select the access type: All (match all types)
? Select the role All users
? Select the permission to apply Explicitly deny access
```

```sh
$ lb acl
? Select the model to apply the ACL entry to: user
? Select the ACL scope: A single method
? Enter the method name __create__notes
? Select the role The user owning the object
? Select the permission to apply Explicitly grant access
```

```sh
$ lb acl
? Select the model to apply the ACL entry to: note
? Select the ACL scope: All methods and properties
? Select the access type: Write
? Select the role The user owning the object
? Select the permission to apply Explicitly grant access
```

```sh
$ lb acl
? Select the model to apply the ACL entry to: note
? Select the ACL scope: A single method
? Enter the method name archive
? Select the role The user owning the object
? Select the permission to apply Explicitly grant access
```

```sh
$ lb acl
? Select the model to apply the ACL entry to: user
? Select the ACL scope: All methods and properties
? Select the access type: Read
? Select the role The user owning the object
? Select the permission to apply Explicitly grant access
```

Check out the models json files and notice the added acls section.


