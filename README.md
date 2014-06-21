# JSON API Test

JSON driven testing of REST APIs.

This is a test framework targeted at JSON REST APIs. It comes in the form of a Node.js package called `jsonapitest`
that is available on the command line to run your tests. Tests are specified in JSON files and grouped into
test suites. Each test contains a list of API calls with assertions about the
response. The framework supports using JSON Schema to validate the structure of responses. All HTTP traffic is logged extensively
by the test runner to help debug test failures. Any data (database records, user credentials etc.) that the tests
need are specified in JSON format and this data can be interpolated into the API calls.

## Motivation

I had a REST API implemented in Node.js and I started out writing my API tests with Mocha and Supertest. Although this approach worked
I ended up with test code that was time consuming and complex. Also, I didn't like the fact that my
tests were coupled to the implementation of the API. I tried doing some semi automated testing with curl and although I appreciate the simplicity
of curl the approach wasn't sufficiently structured and automated for my needs. What I was looking for was a declarative and black-box
way to do API testing. Here are a few selling points for `jsonapitest`:

* Black box testing of APIs means the tests are not tied to the implementation behind the API (i.e. programming language, database etc.)
* Black box testing will encourage you to design more complete and user friendly APIs (since you cannot easily get at the implementation behind the API)
* By having test definitions be simple and pure data structures you are not tied to any particular test framework implementation. This means the test runner, the http client and the assertion engine could all be re-implemented and swapped out fairly easily.
* The fact that you are constrainted to a simple JSON structure for tests will help keep your tests dumb and devoid of complicated logic. This makes test maintenance easier.
* Since test specifications are pure data they lend themselves well to building for example a testing UI or API documentation.
* Debugging is helped by the verbose logging of all HTTP requests and responses (this log is also in JSON format)
* It's easy to point the test runner at different environments (i.e. a test, development or staging server)

## Installation

```
npm install jsonapitest -g
```

## Example

Specify your test in a JSON file:

```json
{
  "data": {
    "schema": {
      "user": {
        "type": "object",
        "properties": {
          "id": {"type": "integer"},
          "name": {"type": "string"},
          "email": {"type": "string", "format": "email"}
        }
      }
    },
    "users": {
      "member": {
        "id":1,
        "email":"joe@example.com"
      }
  	}
  },
  "suites": [
  	{
  		"name": "users",
    	"tests": [
	      {
	        "name": "get_user_success",
	        "description": "Fetch info about a user",
	        "api_calls": [
	          {
	            "request": "https://api.some-hostname.com/v1/users/{{users.member.id}}",
              "status": 200,
	            "assert": {
	              "select": "body",
	              "schema": "{{schema.user}}",
	              "equal_keys": {
	                "id": "{{users.member.id}}",
	                "email": "{{users.member.email}}"
	              }
	            }
	          }
	        ]
	      },
	      {
	        "name": "get_user_missing",
	        "description": "Trying to fetch info about a user that doesn't exist",
	        "api_calls": [
	          {
	            "request": "https://api.some-hostname.com/v1/users/99999999",
	            "status": 404
	          }
	        ]
	      }
    	]
  	}
  ]
}
```

Run it:

```
jsonapitest path-to-your-test-file.json
```

Check out the [Parse CRUD example](doc/examples/parse/README.md) for a more realistic example.

## Test File Structure

Tests are written in one or more JSON files and you may choose any file structure you like.
With a small test case you may want to put all test code in a single file. A more typical
structure is to divide your test code into three sets of files: configuration, data, and test suites.
Here is an example:

```
test/integration/config.json
test/integration/data.json
test/integration/users_test.json
test/integration/articles_test.json
```

Here config.json will contain the configuration, data.json the data, users_test.json the
test suite for users, and articles_test.json the test suite for articles. To run the tests:

```
jsonapitest test/integration/config.json test/integration/data.json test/integration/users_test.json test/integration/articles_test.json
```

Or more conveniently:

```
jsonapitest test/integration/*.json
```

The order in which test files are given to the test runner determines the execution order of tests. Since test suites
are supposed to be independent of eachother this typically won't affect the outcome. The order comes into play
if you have files with overlapping config or data properties. In this case later files will take precedence over earlier ones through a deep merge of the config and data properties. An example of how this may be used is when you would like to override the
default configuration or data. Suppose you usually run your tests against a local
test or development server but at times would like to run them against a remote staging server. You could then have
a configuration file at test/integration/env/staging.json:

```json
{
  "config": {
    "defaults": {
      "api_call": {
        "request": {
          "base_url": "https://my.staging.api.example"
        }
      }
    }
  }
}
```

And run your tests against staging like so:

```
jsonapitest test/integration/*.json test/integration/env/staging.json
```

## The Anatomy of Test Files

The JSON in test files may contain one or more of the following top level properties:

```
config
data
suite
suites
```

### Configuration

The config property is an optional property where you can point out the path to a log file where HTTP requests are logged,
the base_url of your server, and any default headers and response status of your API calls:

```json
"config": {
  "log_path": "log/integration-test-results.json",
  "defaults": {
    "api_call": {
      "request": {
        "base_url": "http://localhost:3001",
        "headers": {
          "X-API-CALL-ID": "{{$api_call_id}}",
          "X-Token": "secret-api-token-goes-here",
          "Accept": "application/json",
          "Content-Type": "application/json"
        }
      },
      "status": 200
    }
  }
}
```

For the `X-API-CALL-ID` header above we are interpolating a built in variable called `$api_call_id` that will be set to a unique hex
digest for each API call (see more under [Data Interpolation](#data-interpolation)). This is a technique you can use to make it easier
to find test request in your server logs.

### Data

The data property is a free form custom container for any kind of data that you need for your tests, i.e. database data, user
credentials etc. Data is interpolated in api calls with the double curly syntax (i.e. {{my_data}}, see [Data Interpolation](#data-interpolation)).

You will most likely need to populate your database with test data before running your tests. If so any script that you write for this
should probably use the JSON data defined by the `data` property (i.e. database records or documents). In general its a good idea
to write your API tests so that they make as few assumptions about the state of the system as possible. However,
in some test scenarios you really need to know exactly what the state of the system is in order to be able to make assertions and achieve high test coverage.
One approach that works in some projects is to run tests against a copy of the production data with a small amount of known test data added to it. Data population is currently outside the scope of this framework.

### Test Suite

Use the suite property to define a single test suite:

```json
{
  "suite": {
    "name": "users",
    "description": "CRUD operations on the user resource",
    "tests": [
      {
        "name": "get_user_success",
        "description": "Fetch info about a user",
        "api_calls": [
          {
            "request": {
              "path": "/v1/users/{{users.member.id}}"
            },
            "assert": [{
              "select": "body",
              "schema": "{{schema.user}}",
              "equal_keys": {
                "id": "{{users.member.id}}",
                "email": "{{users.member.email}}"
              }
            }]
          }
        ]
      }
    ]
  }
}
```

A test suite is made up of a name, an optional description, and an array of tests. Each test in turn has a name and an optional
description and an array of api calls. To define many test suites in a single file, use the suites (plural) property and have it
point to an array of suite objects.

### API Call

The API call lies at the heart of your API testing and it is made up of an HTTP request and one or more assertions against the response. An API call can also save data from the HTTP response for use in later API calls.

### The Request

The `request` property of each API call is an object with the following properties:

* `method` - the HTTP verb (i.e. GET, PUT, POST, DELETE etc.). Defaults to "GET".
* `path` - the path to make the request to. If a `base_url` has been configured then the `url` property will be set to the base_url joined with the path
* `url` - specify the full URL here instead of the path if you need a URL different from the base_url
* `headers` - any custom HTTP headers to use for the request
* `params` - any query or post parameters to pass with the request
* `files` - an array with paths to files to be uploaded

You can also let the `request` property be a string for simple requests:

```json
{
  "request": "DELETE /v1/users/{{users.member.id}"
}
```

Notice that you can also append query parameters to the path instead of using the `params` property:

```json
{
  "request": "/v1/users?limit=10&offset=10"
}
```

### Assertions

Assertions are made up of a select property and one ore more assertion properties. The select property determines which part of the response the assertions should be applied to (typically the body or some subset of the body). The following properties are available in the HTTP response object:

```
status
headers
body
response_time
```

#### Status assertions

Since making an assertion about the response status code is so common some syntactic sugar is available:

```json
"api_calls": [
  {
    "request": {
      "path": "/v1/users"
    },
    "status": 200
  }
]
```

This is translated to:

```json
"api_calls": [
  {
    "request": {
      "path": "/v1/users"
    },
    "assert": [
      {
        select: "status",
        equal: 200
      }
    ]
  }
]
```

#### Assert: schema

TODO

#### Assert: equal/not_equal

TODO

#### Assert: contains/not_contains

TODO

#### Assert: length

TODO

### Saving Data

TODO

## Data Interpolation

Happens at API call time.

## Merging Objects

TODO

## Recommended Reading

* [Understanding JSON Schema Book](http://spacetelescope.github.io/understanding-json-schema/UnderstandingJSONSchema.pdf)
