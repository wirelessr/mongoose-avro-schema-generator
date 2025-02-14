<p align="center">
  <img alt="Mongoose Avro Schema Creator" src="./.github/logo.svg" width="888">
</p>

<p align="center">
  <a href="https://travis-ci.org/researchgate/mongoose-avro-schema-generator"><img alt="Build Status" src="https://travis-ci.org/researchgate/mongoose-avro-schema-generator.svg?branch=master"></a>
  <a href="https://codecov.io/gh/researchgate/mongoose-avro-schema-generator"><img src="https://codecov.io/gh/researchgate/mongoose-avro-schema-generator/branch/master/graph/badge.svg" /></a>  
  <a href="https://dependencyci.com/github/researchgate/mongoose-avro-schema-generator"><img alt="Dependency Status" src="https://dependencyci.com/github/researchgate/mongoose-avro-schema-generator/badge"></a>
  <a href="https://greenkeeper.io/"><img alt="Greenkeeper" src="https://badges.greenkeeper.io/researchgate/mongoose-avro-schema-generator.svg"></a>
  <a href="https://www.npmjs.com/package/@researchgate/mongoose-avro-schema-generator"><img alt="NPM version" src="https://img.shields.io/npm/v/@researchgate/mongoose-avro-schema-generator.svg"></a>
</p>

A node module that generates Apache avro schemas from mongoose schemas.

## Getting Started
### Prerequesites
The Mongoose Avro Schema Generator requires `node >= 9.2.0`. The only dependency is `mongoose >= 5.0.9`, a connection to MongoDB is not necessary.

### Installation
Using yarn:
```shell
yarn add mongoose-avro-schema-generator@https://github.com/wirelessr/mongoose-avro-schema-generator
```

Using npm:
```shell
npm install mongoose-avro-schema-generator@https://github.com/wirelessr/mongoose-avro-schema-generator
```

### Quick Start
In order to generate schemas for all registered mongoose models we import the module, create a new `MongooseAvroSchemaGenerator` and run the `generate()` method.

Let's first register a simple mongoose model.
```js
let schema = new Schema({
    something: String,
});
mongoose.model('mySchema', schema);
```
Now we instantiate the Mongoose Avro Schema Generator with a the mongoose instance.
```js
const Generator = require('mongoose-avro-schema-generator');
const mongooseAvroSchemaGenerator = new Generator(mongoose);
```
Then `mongooseAvroSchemaGenerator.generate()` will output an array of all generated schemas.
```json
[
    {
        "dbcollection": "myschemas",
        "dbtype": "mongodb",
        "fields": [
            {
                "name": "something",
                "type": ["null", "string"],
                "default": null
            },
            {
                "name": "_id",
                "type": [
                    "null",
                    {
                        "subtype": "objectid",
                        "type": "string"
                    }
                ],
                "default": null
            },
            {
                "name": "__v",
                "type": ["null", "double"],
                "default": null
            }
        ],
        "name": "mySchema",
        "namespace": "some.namespace"
    }
]
```
Please note that the schema also contains the auto-generated fields `_id` and `_v`. In the following chapters we will have a look at further details.

## Usage
Generate avro schemas for all registered mongoose models:
```js
mongooseAvroSchemaGenerator.generate();
```
Restrict the schema generation to a set of models:
```js
mongooseAvroSchemaGenerator.generate(['User', 'Transaction']);
```
Override the default "mongoose" namespace by setting the second parameter in the constructor: 
```js
new Generator(mongoose, 'some.custom.namespace');
```

## Mapping
This section will explain the mapping from mongoose types to avro schemas.

### Primitive Types
The following table lists all the primitive types and their mapped equivalent in the avro schema.

| Native/Mongoose Type | Avro Type |
| -------------------- | --------- |
| `String` | `"string"` |
| `Schema.Types.String` | `"string"` |
| `Boolean` | `"boolean"` |
| `Number` | `"double"` |
| `Buffer` | `"bytes"` |
| `Date` | `{ "type": "long", "subtype": "date" }` |
| `Schema.Types.ObjectId` | `{ "type": "string", "subtype": "objectid" }` |

Please note that `Date` and `Schema.Types.ObjectId` are mapped to objects with a more specific subtype.

Warning: The `Schema.Types.Mixed` type is not supported. The same applies to the equivalent empty object literal `{}` or the empty array `[]`. Trying to generate a schema from a model with such a type will result in an error.

### Arrays
Arrays are mapped to the avro type array. For example the mongoose field
```some: [Number]```
is getting mapped to the following field in the avro schema:
```json
{
    "name": "some",
    "type": [
        "null", 
        {
            "type": "array",
            "items": ["null", "double"]
        }
    ],
    "default": null
}
```
We will later see how the default and null values are generated.

### Embedded Documents
Embedded documents are mapped to avro records. For example
```some: { thing: String }```
is represented by following avro definition:
```json
{
	"name": "some",
	"type": [
		"null",
		{
			"name": "someEmbedded",
			"type": "record",
			"fields": [{
				"name": "thing",
				"type": ["null", "string"],
				"default": null
			}]
		},
		"record"
	],

	"default": null
}
```
Please note the name "someEmbedded" of the embedded record which is autogenerated from the parent object with name "some". This is necessary because in an avro schema embedded documents always need a name.

Records can also appear as array items. Like with embedded documents the name will be inferred from the parent object. For example
```some: [{ thing: String }]```
will be mapped to an avro schema containing the following field:
```json
{
	"name": "some",
	"type": [
		"null",
		{
			"type": "array",
			"items": {
				"type": [
					"null",
					{
						"name": "someItemEmbedded",
						"type": "record",
						"fields": [{
							"name": "thing",
							"type": ["null", "string"],
							"default": null
						}]
					}
				]
			}
		}
	],
	"default": null
}
```
Please note the name "someItemEmbedded" of the embedded record which is autogenerated from the parent object with name "some". The same applies for records embedded in arrays of arrays which will have the suffix "ItemItem". In general every array layer appends another "Item" suffix.

### Attributes
The Mongoose Avro Schema Generator makes use of the mongoose attributes `required` and `default`.

#### Nullable
The avro type `null` will be included automatically if no `required : true` attribute is set for a field in mongoose.
Hence the avro schema for the following mongoose schema
```js
let schema = new Schema({
    something: { type: String },
});
mongoose.model('mySchema', schema);
```
will have a union type of `null` and `string`
```json
{
    "name": "something",
    "type": ["null", "string"],
    "default": null
}
```
However if we add the required attribute
```js
let schema = new Schema({
    something: { type: String, required: true },
});
mongoose.model('mySchema', schema);
```
the type will be restricted to string.
```json
{
  "name": "something",
  "type": "string"
}
```
We note that also the default value of `null` has been removed.

#### Default Values
Default values are either defined explicitly or are implicitly inferred.

##### Explicit Defaults
The mongoose schema
```js
new Schema({
    something: { type: String, default: 'foo' },
});
```
will be transformed into an avro schema containing the following field
```json
{
    "name": "something",
    "type": ["null", "string"],
    "default": "foo"
}
```
Arrays will always have a default of `null`.

Important: Functional default values are not supported and will be ignored, i.e. treated as if the field doesn't have any default.
##### Implicit Defaults
If a field can be null and no explicit default value is defined, the default will be set to `null`.

## Complex Example
Let's register a schema with mongoose.
```js
const mongoose = require('mongoose');
const mongooseAvroSchemaGenerator = require('mongoose-avro-schema-generator');
mongooseAvroSchemaGenerator.init(mongoose);

let schema = new Schema({
    something: { type: [[Number]], default: ['foo'] },
    else: [String]
});
mongoose.model('mySchema', schema);
```
Then `mongoosevroSchemaGenerator.generate(['mySchema'], { namespace: 'some.namespace' })` will return the following avro schema:
```json
[
    {
        "dbcollection": "myschemas",
        "dbtype": "mongodb",
        "type": "record",
        "fields": [
            {
                "name": "something",
                "type": [
                    "null",
                    {
                        "type": "array",
                        "items": [
                            "null",
                            {
                                "type": "array",
                                "items": ["null", "string"]
                            }
                        ]
                    }
                ],
                "default": null
            },
            {
                "name": "else",
                "type": [
                    "null",
                    {
                        "type": "array",
                        "items": ["null", "string"]
                    }  
                ],
                "default": null
            },
            {
                "default": null,
                "name": "_id",
                "type": [
                    "null",
                    {
                        "subtype": "objectid",
                        "type": "string"
                    }
                ]
            },
            {
                "default": null,
                "name": "__v",
                "type": ["null", "double"]
                
            }
        ],
        "name": "mySchema",
        "namespace": "some.namespace"
    }
]
```

## Running tests
Mocha tests for Mongoose Avro Schema Generator can be found in `/test`. A yarn job is configured to run those tests using `yarn test`.


