# Basics

## Connection

installation : `npm install mongoose`

create a function for database connection:

```js
import mongoose from "mongoose";
export const connectDB = async () => {
  try {
    await mongoose.connect('mongodb://127.0.0.1:27017/my_database');
  } catch (err) {
    console.log("Connection Error :: ",err);
    process.exit(1);
  }
};
```

- If `my_database` does not exist yet, MongoDB will automatically create it the first time we insert data.

- **Connection Lifecycle:** In production apps, `mongoose.connect()` is called once when the server starts up. Mongoose manages the internal connection pool automatically.

import and initialize in index.js(entry file):

```js
import { connectDB } from "...";
import express from "express";

const app = express();

connectDB().then(() => {
  app.listen(process.env.PORT);
});
```

> While Mongoose has a built-in feature called **operation buffering** (which queues up queries until the connection is actually made), waiting for the connection ensures that the app doesn't accept incoming traffic if the database is unreachable.

> **The Mongoose connection persists as long as your server is running**.

Any file in the project can simply import `mongoose` and use the defined models to perform CRUD operations immediately.

## Schema

The **blueprint**. It defines what fields a document has, what data types those fields accept, and what constraints apply

## Mongoose SchemaTypes

- String

- Number

- Boolean

- Date

- Schema.Types.ObjectId: unique IDs(for referencing other documents)

- Array

- Map

- Buffer: For raw binary data

## Contraint Rules

### For all types:

- required

- unique

- immutable : prevents updates to the field once the document is created

### For String:

- enum: 

- match: a regex check

- minLength/ maxLength

- trim

### For Number

- min/max

### For Date

- min/max

```js
const schemaName = new mongoose.Schema({
    fieldName: {
        // 1. Define the type first
        type: DataType,
        //2. add simple constraint
        constraintName1: value,
        // 3. Add constraints with custom messages
        constraintName2: [value, 'Error message if violated'],
        // 4. Custom functional rules
        validate: {
          validator: function(v) { return true/false; },
          message: 'Custom message'
        }
    }
})
```

##### Customer Validator

It can be defined using `validate` property in the schema definition, which can 

- either accept a synchronous or asynchronous function, or

- an object containing a validator function and a custom error message

The argument of the validator function(v) is the value being checked.

Inside the validator function, `this` refers to the current document being saved. This allows us to validate one field based on the value of another fields. 

To use this validators in update queries:

```js
{ 
    runValidators: true, 
    context: 'query' // Crucial flag to enable this
  }
```

- By default, most validators only run on `.save()` and `.create()`. 

- In case of `.updateOne()` or `.findByIdAndUpdate()`, one must explicitly pass the option `{runValidators: true}` in the query options.

- Custom validators do not run if the field value is missing or undefined (for non-required fields).

- **Handling Arrays :** If we use a validator to an array field, the validator runs on the **entire array** object. If you want to validate **each individual item** inside an array, you need to use a nested schema definition or use Mongoose's cast rules.

### Model

A **Model** is a wrapper around a MongoDB collection. It is constructed from a Schema and serves as the primary tool for performing database-level CRUD operations.

```js
const userSchema = new mongoose.Schema({
  name: String,
  email: String,
  role: { type: String, default: 'user' }
});

//creating a model
const User = mongoose.model('User', userSchema);
```

Mongoose automatically determines the collection name by converting the Model name to **lowercase and pluralizing it** (e.g., a model named `'User'` targets the `'users'` collection; `'Person'` targets `'people'`).

### Document

A **Document** is a single instance created from a Model. It directly represents **one JSON-like BSON record** inside a MongoDB collection.
