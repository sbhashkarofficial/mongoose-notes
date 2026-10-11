# CRUD

## Mongoose Queries

It return a Mongoose Query Object. 

Always either `await` it or attach `.exec()`. Can use both.

```js
const user = await User.findOne({ email: 'alice@example.com' }).exec();
```

## Create Operations

### 1. Model.create()

```js
const newUser = await User.create({
    username: 'alex',
    password: 'alex@sb.com'
});

//multiple documents insertion
cont newUsers = await User.create([
    {username: 'bob', email: 'bob@sb.com'},
    {username: 'jane', email: 'jane@sb.com'}
])
```

### new Model() + .save()-instance method

```js
const user = new User({});
await user.save();
```

## 2. Read Operations

- `User.find(filter)`: Array of documents

- `User.fiindOne(filter)`: single document (first match) / null

- `User.findById(id)` : single document / null

## 3. Update Operations

### Model.updateOne

```js
Model.updateOne(filter, update, options)
```

### options

```js
//when using findByIdAndUpdate or findOneAndUpdate
{
    new: true, //return the updated document (default is false)
    runValidators: true, //Enforce Schema validation rules during updates
    upsert: false //If true, creates a new document if no match is found
}
```

### filter

It is a plain js object that tells MongoDB which documents to match.

- All conditions in a single object are implicitly combined with a **LOGICAL AND**.

```js
// Match documents where status is 'active' AND role is 'admin'
const filter = { 
  status: 'active', 
  role: 'admin' 
};
```

- To filter properties inside nested objects or subdocuments, wrap the path in quotes using dot notation:

```js
// Match users whose address city is 'New York'
const filter = { 
  'address.city': 'New York' 
};
```

#### Comparison Operators

query operators starting with a `$` symbol to handle range and inequality comparisons:

- `$eq`/ `$ne`: equal to/ not equal to - 
  `{ age: { $eq: 25 } }`

- `$gt`/ `$lt`: greater than/ less than

- `$gte`/ `$lte`: greater than or equal to/ less than or equal to

- `$in` / `$nin`: in or not in - in matches any value in an array 
  `{ sub: { $in: ['math', 'eng', 'hin'] } }` 

We can chain multiple operators on a single field:

```js
// Find products priced between $20 and $100 inclusive
const filter = {
  price: { 
    $gte: 20, 
    $lte: 100 
  }
};
```

#### Logical Operators

##### $or

```js
{
    $or: [{<condition_1>}, {<condition_2>}]
}
//find users who are either an 'admin' or have a status of 'VIP'
const filter = {
    $or: [
        {role: 'admin'},
        {status: 'VIP'}
    ]
}
const users = await User.find(filter);
```

#### $and

In MongoDB, adding multiple key-value pairs inside the same filter object creates an **implicit AND**. In most cases, you do not need to write `$and` explicitly.

```js
//Implicit AND (preferred)
const filter1 = {
    role: 'amdin',
    status: 'active'
};

//Explicit AND 
const filter2 = {
    $and: [
        {role: 'admin'},
        {status: 'active'}
    ]
}
```

Explicit `AND` is required when: 

- We have multiple conditions on the same field

```js
const filter = {
    $and: [
        {price: {$exists: true}},
        {price: {$ne: 1.99}}
    ]
};
//this is a better and preferred way
{
    price: {
        $exists: true,
        $ne: 1.99
    }
}
```

- We need to combine multiple `$or` clauses.

```js
// Find users who are (Admin OR Editor) AND (Active OR Pending)
const filter = {
  $and: [
    { $or: [{ role: 'admin' }, { role: 'editor' }] },
    { $or: [{ status: 'active' }, { status: 'pending' }] }
  ]
};
```

The order of conditions does not affect the query as well as performance.

#### $nor

The `$nor` operator evaluates an array of expressions and returns documents that **fail all** of the conditions in the array. It is the exact opposite of `$or`.

```js
{ $nor: [ { <condition1> }, { <condition2> }, ... ] }
```

> `$nor` also matches documents where those fields do not exist at all, because a missing field fails the condition inside `$nor`.

#### $not

The `$not` operator performs a logical `NOT` on a specific operator expression. It negates the condition that follows it.

It takes an **object expression**, not an array.

```js
{ <field>: { $not: { <operator-expression> } } }
```

```js
//find products where the price is not greater than 50
const filter = {
  price: { 
    $not: { $gt: 50 } 
  }
};
```

### Other Operators

##### $type

Matches fields that hold a specific BSON/JavaScript data type.

```js
const filter = {age: {$type: 'number'}};
```

##### $expr

Allows us to compare two fields inside the same document. Used with logical operators.

```js
const filter = {$expr: {$gt: ['$spent', '$budget']}};
```

#### Array Query Operators

##### $elemMatch

Matches documents if an array field contains **at least one single element** that satisfies all specified criteria.

```js
// Document structure: { items: [ { name: 'laptop', qty: 5 }, { name: 'phone', qty: 20 } ] }

// Find orders containing containing an item where name is 'laptop' AND qty is >= 5
const filter = {
    items: {
        $elemMatch: {name: 'laptop', qty: {$gte: 5}}
    }
}
```

##### $all

Matches arrays that contain **every listed element**, regardless of their order in the array.

```js
// Match posts tagged with BOTH 'node' and 'mongodb'
const filter = {
  tags: { $all: ['node', 'mongodb'] }
};
```

##### $size

Matches documents where an array field has a specific number of elements.

```js
// Find users who have exactly 3 items in their 'roles' array
const filter = {
  roles: { $size: 3 }
};
```

#### Text & Pattern Matching

##### `$text` and `$search`

Performs text search on fields that have a **Text index** built in the db. It is vastly faster than regex.

```js
const filter = {
    $text: {$serach: 'something...`}
};
```

##### `$regex` and `$options`

Performs pattern searches on string fields.

```js
// Case-insensitive ('i') search for usernames starting with 'admin'
const filter = {
  username: { $regex: '^admin', $options: 'i' }
};
```

### Model.findOneAndUpdate

To update the first document matching `filter`.

`findOneAndUpdate(filter, update, options)` 

### Model.findByIdAndUpdate

To update a single document identified by `_id`.

`findByIdAndUpdate(id, update, options)`

### Model.updateMany

To update all documents matching filter

`updateMany(filter, update)`

### Structure of the Update Object

```js
{
  $operator1: { 
    field1: value1, 
    field2: value2 
  },
  $operator2: { 
    field3: value3 
  }
}
```

- If you pass a plain object without any `$` operators to Mongoose in `findByIdAndUpdate` or `findOneAndUpdate`, **Mongoose automatically wraps the fields inside `$set`**

- To modify multiple fields simultaneously, we can:
  
  1. Include **multiple fields under a single operator** (e.g., updating multiple values with `$set`).
  
  2. Combine **multiple update operators** in the same update object (e.g., updating text with `$set` while incrementing a counter with `$inc`).

- To update nested object always use **dot notation.**

### Atomic Update Operators

```js
// 1. $set — Updates specific field values (Default behavior in Mongoose)
await User.updateOne(
  { _id: userId },
  { $set: { email: 'newemail@example.com', isActive: true } }
);

// 2. $unset — Removes a field from the document
await User.updateOne(
  { _id: userId },
  { $unset: { temporaryToken: '' } } //similar to setting it undefined
//The value assigned to the field in the object (e.g., 1 or "") does not matter.
);

// 3. $inc — Increments (or decrements) a numerical field
await Product.updateOne(
  { _id: productId },
  { $inc: { stockQuantity: -1, viewCount: 1 } } // Decrease stock by 1, increase views by 1
);

// 4. $push — Adds an item to an array field
await User.updateOne(
  { _id: userId },
  { $push: { roles: 'editor' } }
);

// 5. $addToSet — Adds an item to an array ONLY if it doesn't already exist (Prevents duplicates)
await User.updateOne(
  { _id: userId },
  { $addToSet: { tags: 'javascript' } }
);

// 6. $pull — Removes matching items from an array
await User.updateOne(
  { _id: userId },
  { $pull: { roles: 'guest' } }
);
```

### Model.findById and .save()-instance method

- triggers `pre('save')` and `post('save')` hooks. 

- automatically runs all schema validations.

- it is slower than `findByIdAndUpdate()`

### 4. Delete Operations

1. delete a single document by id and return the deleted document

```js
const deletedUser = await User.findByIdAndDelete(id);
```

2. delete the first document matching a criteria

```js
await User.deleteOne(filter);
```

3. Delete all documents matching a criteria

```js
await User.deleteMany(filter);
```
