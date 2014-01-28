__seraph_model__ provides some convenient functions for storing and retrieving
typed nodes from a neo4j database. It is intended to work with 
[seraph](https://github.com/brikteknologier/seraph). 

**using seraph-model < 0.6.0? please read [the changelist](#changelist)!**

<a name="quick"/>
### Quick example

```javascript
var db = require('seraph')('http://localhost:7474')
var model = require('seraph-model');

var User = model(db, 'user');

User.save({ name: 'Jon', city: 'Bergen' }, function(err, saved) {
  if (err) throw err;

  User.findAll(function(err, allUsers) {
    // allUsers -> [{ name: 'Jon', city: 'Bergen', id: 0 }]
  });
  User.where({ city: 'Bergen' }, function(err, usersFromBergen) {
    // usersFromBergen -> all user objects with city == bergen
  });
})

```

## Compatibility

seraph-model 0.5.2 works with Neo4j-2.0.0-M05 and Neo4j-2.0.0-M06.

To check if it works with your version, you should check out the repo, and
change the Neo4j version at the start of the tests to the version you're running

## Changelist

Is [here](#changelist).

# Documentation

## How to do things

* [Creating a new Model](#create)
* [Adding preparers](#preparation)
* [Adding validators](#validation)
* [beforeSave/afterSave events](#saveevents)
* [Setting a properties whitelist](#settingfields)
* [Composition of models](#composition)
* [Setting a unique key](#uniqueness)
* [Computed fields](#computed-fields)
* [Schemas](#schemas)

## Model instance methods
* [model.read](#read)
* [model.exists](#exists)
* [model.save](#save)
* [model.push](#push)
* [model.saveComposition](#saveComposition)
* [model.findAll](#findAll)
* [model.where](#where)
* [model.prepare](#prepare)
* [model.validate](#validate)
* [model.fields](#fields)
* [model.setUniqueKey](#setUniqueKey)
* [model.useTimestamps](#useTimestamps)
* [model.addComputedField](#addComputeField)
* [model.cypherStart](#cypherStart)

<a name="create"/>
## Creating a new model

__seraph_model(seraphDbObject, modelTypeName)__

You can create a new model by calling the function returned by requiring
`seraph_model`. There are no instances of this model, only objects, which are 
passed to the model itself in order to perform work on it. Much like seraph
itself.

It works by labelling each object with a `type` that you specify.

### Example
```javascript
var db = require('seraph')('http://localhost:7474');
var Beer = require('seraph_model')(db, 'beer');

Beer.save({name: 'Pacific Ale', brewery: 'Stone & Wood'}, function(err, beer) {
  // saved!
});
```

After running this, your node is saved, and labelled as a `beer`, so a cypher
query like `MATCH node:beer RETURN node` would return your node.

<a name="preparation"/>
## Adding preparers

__Preparers__ are functions that are called upon an object to transform it
before saving it. A preparer is a function that takes an object and a callback,
and calls back with an error and the updated object.

Preparers can also do validation. If a preparer returns an error, it will be
passed to the save callback as if it were a validation error. However, if you
just want to do validation and not mutate the object, use a
[validator](#validation) instead.

Preparers are called before validators.

You can manually prepare an object by using the [model.prepare](#prepare)
function.

### Example

```javascript
var prepareFileSize = function(object, callback) {
  fs.stat(object.file, function(err, stat) {
    if (err) return callback('There was an error finding the file size');
    object.filesize = stat.size;
    callback(null, object);
  });
}

model.on('prepare', prepareFileSize);

model.save({file: 'foo.txt'}, function(err, object) {
  // object -> { file: 'foo.txt', filesize: 521, id: 0 }
});

mode.save({file: 'nonexistant.txt'}, function(err, object) {
  // err -> 'There was an error finding the file size'
});
```

<a name="validation"/>
## Adding validators
__Validators__ are functions that are called with an object before it is saved.
If they call back with anything that is not falsy, the saving process is halted,
and the error from the validator function is returned to the save callback.

Validators are called after preparers.

You can manually validate an object by using the [model.validate](#validate)
function.

### Example

```javascript
var validateAge = function(person, callback) {
  if (object.age >= 21) {
    callback();
  } else {
    callback('You must be 21 or older to sign up!');
  }
}

model.on('validate', validateAge);

model.save({ name: 'Jon', age: 23 }, function(err, person) {
  // person -> { name: 'Jon', age: 23, id: 0 }
});

model.save({ name: 'Jordan', age: 17 }, function(err, person) {
  // err -> 'You must be 21 or older to sign up!'
});
```

<a name="saveevents"/>
## Save events

There's a few events you can listen on:

* `beforeSave` fired after preparation and validation, but before saving.
* `afterSave` fired after saving. 

### Example

```javascript
model.on('beforeSave', function(obj) {
  console.log(obj, "is about to be saved");
})
```

<a name="settingfields"/>
## Setting a properties whitelist

__Fields__ are a way of whitelisting which properties are allowed on an object 
to be saved. Upon saving, all properties which are not in the whitelist are 
stripped. Composited properties are automatically whitelisted.

### Example

```javascript
beer.fields = ['name', 'brewery', 'style'];

beer.save({
  name: 'Rye IPA', 
  brewery: 'Lervig', 
  style: 'IPA',
  country: 'Norway'
}, function(err, theBeer) {
  // theBeer -> { name: 'Rye IPA', brewery: 'Lervig', style: 'IPA', id: 0 }
})
```

<a name="composition"/>
## Composition of Models.

Composition allows you to relate two models so that you can save nested objects
faster, and atomically. When two models are composed, even though you might be
saving 10 objects, only 1 api call will be made.

With this, you can also nest objects, which can make your life a bit easier when
saving large graphs of different objects.

**Composited objects will also be implicitly retrieved when reading from the
database, to a specified depth.**.

**example**

```javascript
var beer = model(db, "Beer");
var hop = model(db, "Hop");

beer.compose(hop, 'hops', 'contains_hop');

var pliny = {
  name: 'Pliny the Elder',
  brewery: 'Russian River',
  hops: [
    { name: 'Columbus', aa: '13.9%' },
    { name: 'Simcoe', aa: '12.3%' },
    { name: 'Centennial', aa: '8.0%' }
  ]
};

// Since objects were listed on the 'hops' key that I specified, they will be 
// saved with the `hop` model, and then related back to my beer.
beer.save(pliny, function(err, saved) {
  // if any of the hops or the beer failed validation with their model, err
  // will be populated and nothing will be saved.
  
  console.log(saved); 
  /* -> { brewery: 'Russian River',
          name: 'Pliny the Elder',
          id: 11,
          hops: 
           [ { name: 'Columbus', aa: '13.9%', id: 12 },
             { name: 'Simcoe', aa: '12.3%', id: 13 },
             { name: 'Centennial', aa: '8.0%', id: 14 } ] }
  */

  db.relationships(saved, function(err, rels) {
    console.log(rels) // -> [ { start: 11, end: 12, type: 'contains_hop', properties: {}, id: 0 },
                      // { start: 11, end: 13, type: 'contains_hop', properties: {}, id: 1 },
                      // { start: 11, end: 14, type: 'contains_hop', properties: {}, id: 2 } ]
  });

  // Read directly with seraph
  db.read(saved, function(err, readPlinyFromDb) {
    console.log(readPliny)
    /* -> { brewery: 'Russian River',
            name: 'Pliny the Elder',
            id: 11 }
    */
  })

  // Read with model, and you get compositions implicitly.
  beer.read(saved, function(err, readPliny) {
    console.log(readPliny)
    /* -> { brewery: 'Russian River',
            name: 'Pliny the Elder',
            id: 11,
            hops: 
             [ { name: 'Columbus', aa: '13.9%', id: 12 },
               { name: 'Simcoe', aa: '12.3%', id: 13 },
               { name: 'Centennial', aa: '8.0%', id: 14 } ] }
    */
  });

  hop.read(14, function(err, hop) {
    console.log(hop); // -> { name: 'Centennial', aa: '8.0%', id: 14 }
  });
});
```

### Updating models with compositions

You can use the regular `model.save` function to update a model with 
compositions on it. If the compositions differ from the previous version of the
model, the relationships to the previously composed nodes will be deleted **but
the nodes themselves will not be**. If you want to update the base model but
don't want the overhead that the compositions involves, you can use `model.save`
with `excludeCompositions` set to true. See the [model.save](#save) docs for
more info.

A couple of alternatives for updating compositions exist: [`model.push`](#push)
for pushing a single object to a composition without having to first read the
model from the database, and [`model.saveComposition`](#saveComposition) for 
updating an entire composition in one go.

### model.compose(composedModel, key, relationshipName[, opts])

Add a composition.

* `composedModel` — the model which is being composed
* `key` — the key on an object being saved which will contained the composed 
  models.
* `relationshipName` — the name of the relationship that is created between
  a root model and its composed models. These relationships are always outgoing.
* `opts` - an object with a set of options. possible options are documented
  below.

#### composition options

* `many` (default = `false`) — whether this is a *-to-many relationship. If 
  truthy, the this composition will always be represented as an array on the 
  base object.
* `orderBy` (default = `null`) - how this composition should be ordered. This
  can be set to either the name of a property on the composed node to order with
  (ascending), or an object with the name of the property value and the order
  direction. Possible values might include: `'age'`, 
  `{property: 'age', desc: true}`, `{property: 'age', desc: false}`.
* `updatesTimestamp`: (default = `false`) - if true, whenever a composed model is
  saved, it will update the `updated` timestamp of the root model. Does nothing
  if `this` is not using timestamps.

### model.readComposition(objectOrId, compositionKey, callback)

Read a single composition from a model.

* `objectOrId` — an id or an object that contains an id that refers to a model.
* `compositionKey` – the composition which to retrieve.
* `callback` — callback for result, format (err, resultingComp). `resulingComp`
  will either be an array of composed objects or a single object if there was
  only one

Example (from the above context)
```javascript
beer.readComposition(pliny, 'hops', function(err, hops) {
  console.log(hops); 
  /* [ { name: 'Columbus', aa: '13.9%', id: 12 },
      { name: 'Simcoe', aa: '12.3%', id: 13 },
      { name: 'Centennial', aa: '8.0%', id: 14 } ]  */
});
```
<a name="uniqueness"/>
## Setting a unique key or index 

In neo4j, you can enforce uniqueness of nodes by using a uniqueness constraint
on a given key for a label. You can add this constraint yourself, but doing so
through seraph-model will give you the option to use the existing node in the event of a 
conflict. 

### Unique Key

Specifiying a unique key will create a constraint on that key. This means that
no two nodes saved as this kind of model can have the same value for that key.

For example:

```javascript
var Car = model(db, 'car');
Car.setUniqueKey('model');
Car.save({make: 'Citroën', model: 'DS4'}, function(err, ds4) {
  // ds4 -> { id: 1, make: 'Citroën', model: 'DS4' }
  Car.save({make: 'Toyota', model: 'DS4'}, function(err, otherDs4) {
    // err.statusCode -> 409 (conflict)
  });
});

Car.save({make: 'Subaru'}, function(err, subaru) {
  // err -> 'The `model` key is not set, but is required to save this object'
});
```

You can also specify that instead of returning a conflict error, that you want
to just return the old object when you attempt to save a new one at the same
index. For example:

```javascript
var Tag = model(db, 'tag');
Tag.setUniqueKey('tag', true);
Tag.save({tag: 'finnish'}, function(err, tag) {
  // tag -> { id: 1, tag: 'finnish' }
  
  // presumably later on, someone wants to save the same tag 
  Tag.save({tag: 'finnish'}, function(err, tag) {
    // instead of saving another tag 'finnish', the first one was returned
    // tag -> { id: 1, tag: 'finnish' }
  });
});
```

<a name="computed-fields"/>
## Computed fields

Computed fields are fields on a model that exist transiently (they aren't stored
in the database) and can be composed of other fields on the object or external
information. You specify the field that you want to be computed, and the function 
that should be used to compute the value of that field for the model, and it will 
automatically be computed every time the model is read (and removed from the
model just before saving). You can use the [addComputedField](#addComputedField)
to add a computed field.

Example:

```
var Car = model(db, 'car');
Car.addComputedField('name', function(car) {
  return car.make + ' ' + car.model;
});
Car.addComputedField('popularity', function(car, cb) {
  fetchPopularityRating(car.make, car.model, function(err, rating) {
    if (err) return cb(err);
    cb(null, rating.numberOfOwners);
  });
});

Car.save({ make: 'Citroën', model: 'DS4' }, function(err, car) {
  // car.name = 'Citroën DS4'
  // car.popularity = 8599
});
```

<a name="schemas"/>
## Schemas

Schemas are a way of defining some constraints on a model and enforcing them
while saving. An example of a schema might be:

```javascript
var User = model(db, 'user');
User.schema = {
  name: { type: String, required: true },
  email: { type: String, match: emailRegex, required: true },
  age: { type: Number, min: 16, max: 85 },
  expiry: Date
}
```

Setting a schema will automatically use the keys of the schema as the model's
[`fields`](#fields) property.

Each of the constraints and their behaviour are explained below.

* [`type`](#schema.type)
  + [`'date'` or `Date`](#schema.type.date)
  + [`'string'` or `String`](#schema.type.string)
  + [`'number'` or `Number`](#schema.type.number)
  + [`'array'` or `Array`](#schema.type.number)
  + [`'boolean'` or `Boolean`](#schema.type.number)
  + [Other types](#schema.type.others)
* [`default`](#schema.default)
* [`trim`](#schema.trim)
* [`lowercase`](#schema.lowercase)
* [`uppercase`](#schema.uppercase)
* [`required`](#schema.required)
* [`match`](#schema.match)
* [`enum`](#schema.enum)
* [`min`](#schema.min)
* [`max`](#schema.max)

<a name="schema.type">
### `type`

A `type` property on the schema indicates the type that this property should be.
Upon saving, seraph-model will attempt to coerce properties that have a `type`
specified into that type.

<a name="schema.type.date"/>
#### `'date'` or `Date`

Expects a date, and coerces it to a number using [`Date.getTime`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date/getTime).
Values will be parsed using [Moment.js' date parser](http://momentjs.com/docs/#/parsing/).

Examples of coercion:

```
new Date("2013-02-08T09:30:26")    ->   1360315826000
"2013-02-08T09:30:26"              ->   1360315826000
1360315826000                      ->   1360315826000
```

<a name="schema.type.string"/>
#### `'string'` or `String`

Expects a string. Values will be coerced to a string using `.toString()`.

<a name="schema.type.number"/>
#### `'number'` or `Number`

Expects a number. Values will be coerced to a number. If the coercion results
in `NaN`, validation will fail.

<a name="schema.type.boolean"/>
#### `'boolean'` or `Boolean`

Expects a boolean. Values that are not already a boolean will be coerced based on
their truthiness (i.e. `!!value`), with the exception of `'0'` which is coerced
to `false`.

<a name="schema.type.array"/>
#### `'array'` or `Array`

Expects an array. If the value is not an array, it will be coerced to an array
by inserting the value into an array and returning that.

Examples of coercion:

```
[1,2,3]     -> [1,2,3]
'cat'       -> ['cat']
```

<a name="schema.type.others"/>
#### Other types

You can give your own types to check against. If `type` is set to a string value
that is not one of the above, the value's type is checked with 
`typeof value == type`. If `type` is a function, the value's type is checked with
`value instanceof type`. 

<a name="schema.default"/>
### `default`

Supply a default value for this property. If the property is undefined or null
upon saving, the property will be set to this value.

**Default value:** `undefined`.
**Example values:** `'Anononymous User'`, `500`, `[1, 2, 3]`. 

Example:

```javascript
User.schema = {
  name: { default: 'Anonymous User' }
}
```

<a name="schema.trim"/>
### `trim`

Trim leading/trailing whitespace from a string.

**Default value:** `false`.
**Example values:** `true`, `false`. 

<a name="schema.uppercase"/>
### `uppercase`

Transform a string to uppercase.

**Default value:** `false`.
**Example values:** `true`, `false`. 

<a name="schema.lowercase"/>
### `lowercase`

Transform a string to lowercase.

**Default value:** `false`.
**Example values:** `true`, `false`. 

<a name="schema.required"/>
### `required`

Ensure this property exists. Validation will fail if it null or undefined.

**Default value:** `false`.
**Example values:** `true`, `false`. 

<a name="schema.match"/>
### `match`

Values should match this regular expression. Validation will value if the value
does not.

**Default value:** `undefined`.
**Example values:** `/^user/i`, `new RegExp("^user", "i")`. 

<a name="schema.enum"/>
### `enum`

Values should be one of the values in the enum. Validation will fail if the value
is not in the enum.

**Default value:** `undefined`.
**Example values:** `['male', 'female']`, `[10, 20, 30]`. 

<a name="schema.min"/>
### `min`

Values should be greater than or equal to the given number. Validation will fail
if the value is less.

**Default value:** `undefined`.
**Example values:** `10`, `0.05`. 

<a name="schema.max"/>
### `max`

Values should be less than or equal to the given number. Validation will fail
if the value is greater.

**Default value:** `undefined`.
**Example values:** `100`, `0.95`. 

<a name="save"/>
#### `model.save(object, [excludeCompositions,] callback(err, savedObject))`

Saves or updates an object in the database. The steps for doing this are:

1. `object` is prepared using [model.prepare](#prepare)
2. `object` is validated using [model.validate](#validate). If validation
   fails, the callback is called immediately with an error.
3. The `beforeSave` event is fired.
4. A cypher query is assembled that will save/update the node with the appropriate
   label, as well as any relevant composited nodes.
5. `object` is saved.
6. The `afterSave` event is fired.

If `excludeCompositions` is truthy, any composed models attached to `object`
will not be altered in the database (they will be ignored), and the object which 
is returned will exclude compositions.

<a name="push"/>
#### `model.push(rootId, compName, object(s), callback(err, savedObject(s)))`

Pushes a single object as a composed model on the model represented by `rootId`.
This does not read the database first so there is no danger of a race condition.

<a name="saveComposition"/>
#### `model.saveComposition(rootId, compName, objects, callback(err, savedObjects))`

Updates a composition set on a model. The models composed under `compName` on the
model will be replaced with those specified by objects. This can be a partial
update if you have an already existing array of composited objects.

<a name="read"/>
#### `model.read(idOrObject, callback(err, model))`

Reads a model from the database given an id or an object containing the id. 
`model` is either the returned object or `false` if it was not found.

<a name="exists"/>
#### `model.exists(idOrObject, callback(err, doesExist))`

Check if a model exists.

<a name="findAll"/>
#### `model.findAll(callback(err, allOfTheseModels))`

Finds all of the objects that were saved with this type.

<a name="where"/>
#### `model.where(callback(err, matchingModels))`

This is a operationally similar to 
[seraph.find](https://github.com/brikteknologier/seraph#node.find), but is
restricted to searching for nodes marked as this kind of model only. Will also
return composited nodes. 

<a name="prepare"/>
#### `model.prepare(object, callback(err, preparedObject))`

Prepares an object by using the `model.preparers` array of functions to mutate
it. For more information, see [Adding preparers](#preparation)

<a name="validate"/>
#### `model.validate(object, callback(err, preparedObject))`

Validates that an object is ready for saving by calling each of the functions in
the `model.validators` array. For more information, see 
[Adding validators](#validation)

<a name="fields"/>
#### `model.fields`

This is an array of property names which acts as a whitelist for property names
in objects to be saved. If it is set, any properties in objects to be saved that
are not included in this array are stripped. Composited properties are 
automatically whitelisted. See 
[Setting a properties whitelist](#settingfields) for more information and
examples.

<a name="setUniqueKey"/>
#### `model.setUniqueKey(keyName, [returnOldOnConflict = false], [callback])`

Adds a uniqueness constraint to the database that makes sure `keyName` has a
unique value for any nodes labelled as this kind of model. If the constraint
already exists, no changes are made.

See the [using a unique key](#unique-key) section for more information and
examples.

<a name="useTimestamps"/>
#### `model.useTimestamps([createdField = 'created', [updatedField = 'updated'])`

If called, the model will add a `created` and `updated` timestamp field to each
model that is saved. These are timestamps based on the server's time (in ms). 

You can also use the `model.touch(node, callback)` function to update the
`updated` timestamp without changing any of the node's properties. This is useful
if you're updating composed models seperately but still want the base model to
be updated.

<a name="addComputedField"/>
#### `model.addComputedField(fieldName, computer)`

Add a [computed field](#computed-fields) to a model.

<a name='#changelist'/>
# Changelist

## 0.6.0

* Models now use [labels](http://docs.neo4j.org/chunked/milestone/graphdb-neo4j-labels.html) (new in neo4j 2) instead of legacy indexes to keep track of their type.
* Removed all legacy indexing. Any legacy indexes you use should be now created
  manually. The `afterSave` or `beforeSave` events are recommended for this
  purpose.
* `setUniqueKey` now uses neo4j 2.0.0 uniqueness constraints.
* `cypherStart` becomes redundant.
* `addUniqueKey` now has a callback, since it is now adding a uniqueness constraint
  to the database.
* Saving a model that has its uniqueness set to `returnOld` will now update the
  existing node's properties on save, to the specified ones (old behaviour was
  to discard the specified properties, make no changes, and return the existing node).
* All timestamps are now in milliseconds, and can no longer be customised.
* New option on `compose`: `updatesTimestamp` - allows the composed node to update
  the `updated` timestamp of any nodes it is composed upon, when updating. This
  functionality existed already, but was not optional. It is now opt-in.
* Both read and write now use only a single API call.

