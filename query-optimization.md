# Query Optimization

## .lean()

By default, Mongoose turns raw database BSON records into fully hydrated **Mongoose Documents**. This process attaches internal getters, setters, validation state, change tracking, and instance methods (`.save()`, `.isModified()`) to every object.

When you only need to read data (e.g., in `GET` API endpoints), this hydration creates unnecessary CPU and memory overhead.

Appending `.lean()` to a query tells Mongoose to skip document hydration and return **plain JavaScript objects (POJOs)** directly from the driver.

### Trade-offs :

- **No Instance Methods or `save()`:**  cannot call `.save()`, `.validate()`, `.remove()`, or any custom schema methods on the returned objects

- **No change Tracking :** modification of properties on a lean object is not tracked, meaning it cannot automatically generate `$set` update operations or track dirty fields.

- **No getters, setters, or middlewares** : Schema getters/setters, default values, and `pre`/`post` query/document middleware hooks do not run on lean results.

> Always use `.lean()` for read-only query endpoints (e.g., rendering lists, dashboards, JSON APIs).

## Field Selection (Projection)

**Field Selection (Projection)** ensures that we pull only the exact fields required.

`Model.find().select('...')`

```js
// 1. String Syntax (Inclusion)
// Pull ONLY name, email, and _id
const users = await User.find().select('name email').lean();

// 2. String Syntax (Exclusion)
// Pull everything EXCEPT password and __v
const usersClean = await User.find().select('-password -__v').lean();

// 3. Object Syntax
const usersObj = await User.find().select({ name: 1, email: 1, _id: 0 }).lean();
```

## Database Indexing

An **Index** creates a special data structure (a B-Tree) that keeps field ordered, allowing MongoDB to perform an **Index Scan (`IXSCAN`)** and locate matching records instantly.

### Types of Indexes

#### Single-Field Index

Best for fields that are frequently used in filters, sorts, or uniqueness checks.

```js
const userSchema = new mongoose.Schema({
  email: {
    type: String,
    unique: true // Automatically creates a unique single-field index
  },
  username: {
    type: String,
    index: true // Creates a standard single-field index
  }
});
```

#### Compound Index (multi field)

```js
const orderSchema = new mongoose.Schema({
  customer: mongoose.Schema.Types.ObjectId,
  status: String,
  createdAt: Date
});

// Compound Index: 
orderSchema.index({ status: 1, createdAt: -1 });
```

**Index Direction Rule:** `1` represents ascending order; `-1` represents descending order. Direction matters when sorting on multiple fields in compound indexes.

### **The Compound Index "ESR" Rule**

When creating compound indexes, follow the **ESR Rule** for field ordering:

1. **E** = **Equality**: Put fields with exact matches first (e.g., `status: 'completed'`).

2. **S** = **Sort**: Put fields used in sorting next (e.g., `createdAt: -1`).

3. **R** = **Range**: Put fields with range operators (`$gt`, `$lt`, `$in`) last.

## Query Execution Analysis with .explain()

To verify whether your queries are actually utilizing indexes, chain `.explain('executionStats')` to your Mongoose query.

```js
const stats = await User.find({ status: 'active' })
  .sort({ createdAt: -1 })
  .explain('executionStats');

console.log(stats.executionStats);
```
