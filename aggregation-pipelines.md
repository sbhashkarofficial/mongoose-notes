# Aggregation Pipelines

```js
// Basic syntax: Takes an array of pipeline stage objects
const results = await Model.aggregate([
  { $stage1 },
  { $stage2 },
  { $stage3 }
]);
```

## Essential Aggregation Pipeline Stages

### match

Filters documents to pass only those that match specified criteria to the next stage.

Always place `$match` as early as possible in the pipeline so MongoDB can utilize database indexes!

```js
{ $match: { status: 'completed', price: { $gte: 50 } } }
```

**The object used inside a `$match` stage is identical to the query filter object**

### group

To group input documents by a specified `_id` expression and computes summary values (counts, totals, averages) across each group.

**Group Accumulators:** `$sum`, `$avg`, `$min`, `$max`, `$push`, `$addToSet`.

```js
{
    $group: {
        _id: '$category', //field to group by (prefixed with $)
        totalRevenue: {$sum: '$price'}, //sums up the 'price' field for each group
        totalProduct: {$sum: 1} //counts number of documents in each group
    }
}
```

### project

Selects, excludes, renames, or computes brand-new fields for output documents.

```js
{
  $project: {
    _id: 0,                           // Exclude _id
    productName: '$title',            // Rename 'title' to 'productName'
    discountedPrice: { $multiply: ['$price', 0.9] } // Compute dynamic field
  }
}
```

### `$sort`, `$limit`, and `$skip` (Pagination & Ordering)

```js
{ $sort: { totalRevenue: -1 } }, // Sort by totalRevenue descending
{ $skip: 10 },                  // Skip first 10 documents
{ $limit: 5 }                   // Limit result to 5 documents
```

### lookup

Performs a `LEFT OUTER JOIN` to combine documents from another collection into the matching pipeline document. This is vastly faster and more flexible than Mongoose `.populate()`.

```js
{
    $lookup: {
        from: 'authors',          // Target collection name in MongoDB (plural, lowercase)
        localField: 'authorId',   // Field in the current collection
        foreignField: '_id',      // Matching field in the target collection
        as: 'authorDetails'       // Name of the array field where matching docs are placed
    }
}
```

> MongoDB's $lookup engine is smart enough to handle array matching natively. If localField is an array of ObjectIds (e.g., comments: [ObjectId("1"), ObjectId("2")]), MongoDB automatically loops through every ID in the array, finds all matching documents, and collects them into the as result array.

### unwind

Deconstructs an array field from input documents to output a document for **each** element of the array.

> If we use `$unwind` on an array that might be **empty**, **null**, or **missing**, that entire document disappears from our results. To prevent this, use the `preserveNullAndEmptyArrays` option:

### facet

lets us run multiple independent aggregation sub-pipelines simultaneously on the same set of input documents within a single query.

**Independent Sub-Pipelines:** Inside `$facet`, we define multiple key-value pairs where each key is a **field name** and its value is an array of standard aggregation stages (such as `$match`, `$group`, `$sort`, `$project`, or `$count`).

**Single Output Document:** It returns a single output document where each **named field** contains an array of results from its respective sub-pipeline.

## ACID Transactions in Mongoose

When executing complex operations that involve multiple database writes (e.g., deducting funds from Account A and adding funds to Account B), we need **ACID Transactions**.

Transactions ensure that either **all operations succeed, or none do (atomic rollback)**, preventing corrupt or half-written states.

Transactions in Mongoose use **Sessions** (`mongoose.startSession()`). One must pass the `{ session }` object to every database operation inside the transaction.

```js
const transferFunds = async (fromAccountId, toAccountId, amount) => {
  // 1. Start a database session
  const session = await mongoose.startSession();

  try {
    // 2. Start the transaction block
    session.startTransaction();

    // 3. Operation A: Deduct amount from sender
    const sender = await Account.findByIdAndUpdate(
      fromAccountId,
      { $inc: { balance: -amount } },
      { new: true, runValidators: true, session } // MUST pass session
    );

    if (sender.balance < 0) {
      throw new Error('Insufficient funds!'); // Trigger abort
    }

    // 4. Operation B: Add amount to receiver
    await Account.findByIdAndUpdate(
      toAccountId,
      { $inc: { balance: amount } },
      { new: true, runValidators: true, session } // MUST pass session
    );

    // 5. Commit all changes to the database permanently
    await session.commitTransaction();
    console.log('Transaction committed successfully.');
  } catch (error) {
    // 6. Roll back all changes if any error occurred
    await session.abortTransaction();
    console.error('Transaction aborted due to error:', error.message);
    throw error;
  } finally {
    // 7. Always end the session to release database connections
    session.endSession();
  }
};
```
