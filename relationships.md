# Reference vs Embed

1. **Embedded Documents (Subdocuments)**: nesting related data directly inside the parent document. Used in `one-to-one` relationships

2. **Referenced Documents**: Storing documents in separate collections and linking them using their `ObjectId`. Used in `one-to-many` or `many-to-many`.

## Embedded Documents

```js
const ChildSchema = new mongoose.Schema({
    field1: {type: ...},
    field2: {type: ...}
});

const ParentSchema = new mongoose.Schema({
    fielda: String,
    childField: ChildSchema
    //childs: [ChildSchema]
})
```

For array of subdocuments: `childs: [childSchema]`

We can use a Mongoose schema directly as a type within another schema declaration

```js
// 1. Define the sub-schema
const addressSchema = new Schema({
  street: String,
  city: String
});

// 2. Pass it directly as the 'type' in the parent schema
const userSchema = new Schema({
  name: String,
  address: {
    type: addressSchema, 
    required: true,
    default: () => ({})
  }
});
```

> If we have exported a model and want to know its schema we can use `.schema` property.

#### Querying

We use dot notations to query embeded fields:

`address.city`;

## Referenced Documents

Reference other model by:

- using `type` as `mongoose.Schema.Types.ObjectId` and 

- setting `ref` to the model name used in `mongoose.model('ModelName',....)`

```js
// Parent / Independent Model: Author
const authorSchema = new mongoose.Schema({
  name: String,
  bio: String
});

const Author = mongoose.model('Author', authorSchema);

// Dependent Model: Book
const bookSchema = new mongoose.Schema({
  title: String,
  publishYear: Number,
  // Reference to the Author model
  author: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'Author', // MUST match the model name registered in mongoose.model('Author', ...)
    required: true
  }
});
```

### Querying

By default, querying `Book.find()` returns only the raw `ObjectId` with the full document.

#### query.populate(collection_name)

It fetches the matching document from the specified collection, and replace the `ObjectId` with the full document.

```js
const populatedBook = await Book.findById(bookId).populate('author');
```

##### Selecting Specific Fields

Pass a second space-separated string parameter containing the fields you want to **include** or **exclude**

```js
// Only fetches the 'name' and 'email' fields from the User document
await Post.findOne().populate('author', 'name email');

// Fetches the User document but EXCLUDES the 'password' field
await Post.findOne().populate('author', '-password');
```

##### Populating Multiple Paths

If we have multiple reference fields in the same schema, we can chain multiple `.populate()` calls together, or pass them as an array.

```js
// Method A: Chaining
await Post.findOne()
  .populate('author')
  .populate('category');

// Method B: Array notation (Cleaner)
await Post.findOne().populate(['author', 'category']);
```

##### Passing a Configuration Object

When we need to apply filters, sorting, or multi-level embedding, we can pass a configuration object to `.populate()`

```js
const book = await Book.find().populate({
  path: 'author',       // [REQUIRED] The schema path/field to populate
  select: 'name email', // [OPTIONAL] Specific fields to include or exclude
  model: 'Author',      // [OPTIONAL] Explicit model name (if not defined in schema ref)
  match: { isActive: true }, // [OPTIONAL] Query filters for populated documents
  options: {           // [OPTIONAL] Options like limit, skip, and sort
    sort: { name: 1 },
    limit: 5,
    skip: 0,
    lean: true
  },
  populate: { ... }     // [OPTIONAL] Nested populate for multi-level references
});
```

- If a populated document fails the `match` filter, Mongoose sets the populated property to `null` (or an empty array `[]` for arrays of references). It does **not** filter out the parent document itself.

- ### **`options` (Object)**
  
  Passes query modifier options (such as `sort`, `limit`, `skip`, and `lean`) directly to the underlying query executed against the populated collection.

##### Deep / Nested Population

to perform **multi-level population** (populating references inside already populated documents).

```js
// Book -> Author -> Publisher
const books = await Book.find().populate({
  path: 'author',
  select: 'name publisher',
  populate: {
    path: 'publisher',
    select: 'companyName address',
    match: { country: 'USA' }
  }
});
```
> It doesn't matter whether we pass where the relationship is one to one or one to many