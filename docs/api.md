# API

* [normalize](#normalizedata-schema)
* [schema](#schema)
  - [Array](#arraydefinition-schemaattribute)
  - [Entity](#entitykey-definition---options--)
  - [Object](#objectdefinition)
  - [Union](#uniondefinition-schemaattribute)
  - [Values](#valuesdefinition-schemaattribute)

## `normalize(data, schema)`

Normalizes input data per the schema definition provided.

* `data`: **required** Input JSON (or plain JS object) data that needs normalization.
* `schema`: **required** A schema definition

### Usage

```js
import { normalize, schema } from 'normalizr';

const myData = { users: [ { id: 1 }, { id: 2 } ] };
const user = new schema.Entity('users');
const mySchema = { users: [ user ] }
const normalizedData = normalize(myData, mySchema);
```

## `schema`

### `Array(definition, schemaAttribute)`

Creates a schema to normalize an array of entities. If the input value is an `Object` instead of an `Array`, the normalized result will be an `Array` of the `Object`'s values. 

*Note: The same behavior can be defined with shorthand syntax: `[ mySchema ]`*

* `definition`: **required** A singular schema that this array contains *or* a mapping of schema to attribute values.
* `schemaAttribute`: *optional* (required if `definition` is not a singular schema) The attribute on each entity found that defines what schema, per the definition mapping, to use when normalizing.  
Can be a string or a function. If given a function, accepts the following arguments:  
    * `value`: The input value of the entity.
    * `parent`: The parent object of the input array.
    * `key`: The key at which the input array appears on the parent object.

#### Instance Methods

* `define(definition)`: When used, the `definition` passed in will be merged with the original definition passed to the `Array` constructor. This method tends to be useful for creating circular references in schema.

#### Usage

To describe a simple array of a singular entity type:

```js
const data = [ { id: '123', name: 'Jim' }, { id: '456', name: 'Jane' } ];
const userSchema = new schema.Entity('users');

const userListSchema = new schema.Array(userSchema);
// or use shorthand syntax:
const userListSchema = [ userSchema ];

const normalizedData = normalize(data, userListSchema);
```

#### Output

```js
{
  entities: {
    users: {
      '123': { id: '123', name: 'Jim' },
      '456': { id: '456', name: 'Jane' }
    }
  },
  result: {
    users: [ '123', '456' ]
  }
}
```

If your input data is an array of more than one type of entity, it is necessary to define a schema mapping. For example:

```js
const data = [ { id: 1, type: 'admin' }, { id: 2, type: 'user' } ];

const userSchema = new schema.Entity('users');
const adminSchema = new schema.Entity('admins');
const myArray = new schema.Array({
  admins: adminSchema,
  users: userSchema
}, (input, parent, key) => `${input.type}s`);

const normalizedData = normalize(data, myArray);
```

#### Output

```js
{
  entities: {
    admins: { '1': { id: 1, type: 'admin' } },
    users: { '2': { id: 2, type: 'user' } }
  },
  result: [
    { id: 1, schema: 'admins' },
    { id: 2, schema: 'users' }
  ]
}
```

### `Entity(key, definition = {}, options = {})`

* `key`: **required** The key name under which all entities of this type will be listed in the normalized response. Must be a string name.
* `definition`: A definition of the nested entities found within this entity. Defaults to empty object.  
You *do not* need to define any keys in your entity other than those that hold nested entities. All other values will be copied to the normalized entity's output.
* `options`:
    - `idAttribute`: The attribute where unique IDs for each of this entity type can be found.  
    Accepts either a string key or a function. Defaults to `id`.  
    As a function, accepts the following arguments, in order: 
      * `value`: The input value of the entity.
      * `parent`: The parent object of the input array.
      * `key`: The key at which the input array appears on the parent object.
    - `mergeStrategy(entityA, entityB)`: Strategy to use when merging two entities with the same `id` value. Defaults to merge the more recently found entity onto the previous.
    - `processStrategy(value, parent, key)`: Strategy to use when pre-processing the entity. Use this method to add extra data, defaults, and/or completely change the entity before normalization is complete. Defaults to returning a shallow copy of the input entity.  
    *Note: It is recommended to always return a copy of your input and not modify the original.*  
    The function accepts the following arguments, in order: 
      * `value`: The input value of the entity.
      * `parent`: The parent object of the input array.
      * `key`: The key at which the input array appears on the parent object.

#### Instance Methods

* `define(definition)`: When used, the `definition` passed in will be merged with the original definition passed to the `Entity` constructor. This method tends to be useful for creating circular references in schema.
* `key`: Returns the key provided to the constructor.

#### Usage

```js
const data = { id_str: '123', url: 'https://twitter.com', user: { id_str: '456', name: 'Jimmy' } };

const user = new schema.Entity('users', {}, { idAttribute: 'id_str' });
const tweet = new schema.Entity('tweets', { user: user }, { 
    idAttribute: 'id_str',
    // Apply everything from entityB over entityA, except for "favorites"
    mergeStrategy: (entityA, entityB) => ({
      ...entityA,
      ...entityB,
      favorites: entityA.favorites
    }),
    // Remove the URL field from the entity
    processStrategy: (entity) => omit(entity, 'url')
});

const normalizedData = normalize(data, tweet);
```

#### Output

```js
{
  entities: {
    tweets: { '123': { id_str: '123', user: '456' } },
    users: { '456': { id_str: '456', name: 'Jimmy' } }
  },
  result: '123'
}
```

### `Object(definition)`

Define a plain object mapping that has values needing to be normalized into Entities. *Note: The same behavior can be defined with shorthand syntax: `{ ... }`*

* `definition`: **required** A definition of the nested entities found within this object. Defaults to empty object.  
You *do not* need to define any keys in your object other than those that hold other entities. All other values will be copied to the normalized output.

#### Instance Methods

* `define(definition)`: When used, the `definition` passed in will be merged with the original definition passed to the `Object` constructor. This method tends to be useful for creating circular references in schema.

#### Usage

```js
// Example data response
const data = { users: [ { id: '123', name: 'Beth' } ] };

const user = new schema.Entity('users');
const responseSchema = new schema.Object({ users: new schema.Array(user) });
// or shorthand
const responseSchema = { users: new schema.Array(user) };

const normalizedData = normalize(data, responseSchema);
```

#### Output

```js
{
  entities: {
    users: { '123': { id_str: '123', name: 'Beth' } }
  },
  result: { users: [ '123' ] }
}
```

### `Union(definition, schemaAttribute)`

Describe a schema which is a union of multiple schemas. This is useful if you need the polymorphic behavior provided by `schema.Array` or `schema.Values` but for non-collection fields.

* `definition`: **required** An object mapping the definition of the nested entities found within the input array
* `schemaAttribute`: **required** The attribute on each entity found that defines what schema, per the definition mapping, to use when normalizing.  
Can be a string or a function. If given a function, accepts the following arguments:  
  * `value`: The input value of the entity.
  * `parent`: The parent object of the input array.
  * `key`: The key at which the input array appears on the parent object.

#### Instance Methods

* `define(definition)`: When used, the `definition` passed in will be merged with the original definition passed to the `Union` constructor. This method tends to be useful for creating circular references in schema.

#### Usage

```js
const data = { owner: { id: 1, type: 'user', name: 'Anne' } };

const user = new schema.Entity('users');
const group = new schema.Entity('groups');
const unionSchema = new schema.Union({
  user: user,
  group: group
}, 'type');

const normalizedData = normalize(data, { owner: unionSchema });
```

#### Output

```js
{
  entities: {
    users: { '1': { id: 1, type: 'user', name: 'Anne' } }
  },
  result: { owner: { id: 1, schema: 'user' } }
}
```

### `Values(definition, schemaAttribute)`

Describes a map whose values follow the given schema.

* `definition`: **required** A singular schema that this array contains *or* a mapping of schema to attribute values.
* `schemaAttribute`: *optional* (required if `definition` is not a singular schema) The attribute on each entity found that defines what schema, per the definition mapping, to use when normalizing.  
Can be a string or a function. If given a function, accepts the following arguments:  
  * `value`: The input value of the entity.
  * `parent`: The parent object of the input array.
  * `key`: The key at which the input array appears on the parent object.

#### Instance Methods

* `define(definition)`: When used, the `definition` passed in will be merged with the original definition passed to the `Values` constructor. This method tends to be useful for creating circular references in schema.

#### Usage

```js
const data = { firstThing: { id: 1 }, secondThing: { id: 2 } };

const item = new schema.Entity('items');
const valuesSchema = new schema.Values(item);

const normalizedData = normalize(data, valuesSchema);
```

#### Output

```js
{
  entities: {
    items: { '1': { id: 1 }, '2': { id: 2 } }
  },
  result: { firstThing: 1, secondThing: 2 }
}
```
