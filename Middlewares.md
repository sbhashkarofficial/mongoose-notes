# Middlewares

**Middleware** (also known as pre and post hooks) are functions that automatically execute control flow **before (`pre`)** or **after (`post`)** specific database operations take place.

## Types of Mongoose Hooks

Mongoose supports four distinct categories of middleware:

1. **Document Middleware:** Runs on individual Document instances (e.g., `save`, `validate`, `remove`). 
   
   1. In these hooks, `this` refers to the **Document**.

2. **Query Middleware:** Runs on Query objects (e.g., `find`, `findOne`, `findOneAndUpdate`, `updateOne`).
   
   1. These are attached to the functions called on the *Model* `(e.g., User.find())`.
   
   2.  In these hooks, `this` refers to the **Query object.**

3. **Aggregate Middleware:** Runs during aggregation pipeline execution (`aggregate`). 
   
   1. In these hooks, `this` refers to the **Aggregation object**.

4. **Model Middleware:** Runs on Model static methods (e.g., `insertMany`).

#### Query Object

represents an **unexecuted database operation**. It holds the state of the request—including filter conditions, field selections, sorting, options, and update payloads—before sending it to MongoDB.

Inside a `pre` query hook, calling filter methods like `.where()`, `.find()`, or `.select()` **appends conditions to the active query chain**.

### pre Hooks vs. post Hooks

These are set up on a user schema.

> Define the hooks **before**  compiling the model using `mongoose.model()`. If we add hooks after compilation, they will not execute.

`pre` hooks run before an action is persisted to the database. They are ideal for data mutation, hashing, sanitization, or throwing error validations.

```js
userSchema.pre('save', async function(next)){
    //...
    next();
}
```

`post` hooks run after the action has completed successfully in the database. They are ideal for side effects like sending welcome emails, logging audit trails, or clearing cache.

> Always use standard `function()` syntax instead of arrow functions `() => {}` so that Mongoose can correctly bind the `this` context to the document or query!

##### next()

- Call `next()` with no arguments when your middleware task completes successfully.

- If something goes wrong (e.g., password hashing fails, or validation fails), pass an `Error` object into `next(error)`.
  
  Mongoose will **immediately stop** the hook execution chain, abort the database operation, and pass the error directly to your application's `catch` block or Express error handler.

- In `post` hooks (which run *after* the database operation completes), `next()` passes control to any remaining `post` hooks before returning the final result to your caller.

- It is not necessary to use next when we throw an error or reject the promise, Mongoose catches it automatically.


