# Rested

[![Go Reference](https://pkg.go.dev/badge/github.com/clarify/rested.svg)](https://pkg.go.dev/github.com/clarify/rested)
[![Go Workflow](https://github.com/clarify/rested/actions/workflows/go.yml/badge.svg?branch=main)](https://github.com/clarify/rested/actions)

**This repository contain a _maintenance_ fork of [rs/rest-layer](https://github.com/rs/rest-layer). We are not planning or accepting any major features at this time, but will accept and conduct minor improvements and bugfixes as needed. Features that are not used by us may be deprecated or dropped in upcoming releases. Breaking changes might be introduced when suitable**

Requires Go 1.17 or newer and use of go modules.

The goals of this fork is:

- Better maintenance of features used by the team behind [Clarify](https://clarify.io).
- Deprecation of unmaintained and uncompleted features to reduce the maintenance surface.
- Gradual introduction of (potentially breaking) changes that can increase stability or reduce the chance programming errors.
- Support for models (structs) one way or anther.
- Eventually compatibility with a production-ready PostgreSQL storage backend of some sort.

| Package                                                                     | Description                                                                                            |
| --------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| [rest](https://pkg.go.dev/github.com/clarify/rested/rest)                   | A `net/http` handler to expose a REST-ful API.                                                         |
| [schema](https://pkg.go.dev/github.com/clarify/rested/schema)               | A validation framework for the API resources.                                                          |
| [resource](https://pkg.go.dev/github.com/clarify/rested/resource)           | Defines resources, manages the resource graph and manages the interface with resource storage handler. |
| [storers/mongo](https://pkg.go.dev/github.com/clarify/rested/storers/mongo) | Implementation of `resource.Storers` interface.                                                        |

## Documentation

- [Breaking Changes](#breaking-changes)
- [Extensions](#extensions)
- [Storage Handlers](#storage-handlers)
- [Usage](#usage)
- [Resource Configuration](#resource-configuration)
  - [Schema](#schema)
  - [Field Definition](#field-definition)
  - [Binding](#binding)
  - [Modes](#modes)
  - [Hooks](#hooks)
  - [Sub Resources](#sub-resources)
  - [Dependency](#dependency)
- [HTTP Request Headers](#http-request-headers)
  - [Prefer](#prefer)
- [HTTP Request Methods](#http-request-methods)
  - [OPTIONS](#options)
  - [HEAD](#head)
  - [GET](#get)
  - [POST](#post)
  - [PUT](#put)
  - [PATCH](#patch)
  - [DELETE](#delete)
- [Querying](#querying)
  - [Filtering](#filtering)
  - [Sorting](#sorting)
  - [Field Selection](#field-selection)
    - [Field Aliasing](#field-aliasing)
    - [Field Parameters](#field-parameters)
    - [Embedding](#embedding)
  - [Pagination](#pagination)
  - [Skipping](#skipping)
- [Authentication & Authorization](#authentication-and-authorization)
- [Conditional Requests](#conditional-requests)
- [Data Integrity & Concurrency Control](#data-integrity-and-concurrency-control)
- [Data Validation](#data-validation)
  - [Nullable Values](#nullable-values)
  - [Extensible Data Validation](#extensible-data-validation)
- [Timeout and Request Cancellation](#timeout-and-request-cancellation)
- [Logging](#logging)
- [CORS](#cors)
- [JSONP](#jsonp) DEPRECATED
- [Data Storage Handler](#data-storage-handler)
- [Custom Response Formatter / Sender](#custom-response-formatter--sender)
- [JSONSchema](#jsonschema)

## Extensions

As REST Layer is a simple `net/http` handler. You can use standard middleware to extend its functionalities. E.g.:

- [CORS](http://github.com/rs/cors)

## Storage Handlers

- [Memory](http://github.com/clarify/rested/tree/master/resource/testing/mem) (test only)
- [MongoDB](http://github.com/clarify/rested/storers/mongo)

## Usage

```go
package main

import (
    "log"
    "net/http"

    "github.com/clarify/rested/resource/testing/mem"
    "github.com/clarify/rested/resource"
    "github.com/clarify/rested/rest"
    "github.com/clarify/rested/schema/query"
    "github.com/clarify/rested/schema"
)

var (
    // Define a user resource schema
    user = schema.Schema{
        Description: `The user object`,
        Fields: schema.Fields{
            "id": {
                Required: true,
                // When a field is read-only, only default values or hooks can
                // set their value. The client can't change it.
                ReadOnly: true,
                // This is a field hook called when a new user is created.
                // The schema.NewID hook is a provided hook to generate a
                // unique id when no value is provided.
                OnInit: schema.NewID,
                // The Filterable and Sortable allows usage of filter and sort
                // on this field in requests.
                Filterable: true,
                Sortable:   true,
                Validator: &schema.String{
                    Regexp: "^[0-9a-v]{20}$",
                },
            },
            "created": {
                Required:   true,
                ReadOnly:   true,
                Filterable: true,
                Sortable:   true,
                OnInit:     schema.Now,
                Validator:  &schema.Time{},
            },
            "updated": {
                Required:   true,
                ReadOnly:   true,
                Filterable: true,
                Sortable:   true,
                OnInit:     schema.Now,
                // The OnUpdate hook is called when the item is edited. Here we use
                // provided Now hook which returns the current time.
                OnUpdate:  schema.Now,
                Validator: &schema.Time{},
            },
            // Define a name field as required with a string validator
            "name": {
                Required:   true,
                Filterable: true,
                Validator: &schema.String{
                    MaxLen: 150,
                },
            },
        },
    }

    // Define a post resource schema
    post = schema.Schema{
        Description: `Represents a blog post`,
        Fields: schema.Fields{
            // schema.*Field are shortcuts for common fields
            // (identical to users' same fields)
            "id":      schema.IDField,
            "created": schema.CreatedField,
            "updated": schema.UpdatedField,
            // Define a user field which references the user owning the post.
            // See bellow, the content of this field is enforced by the fact
            // that posts is a sub-resource of users.
            "user": {
                Required:   true,
                Filterable: true,
                Validator: &schema.Reference{
                    Path: "users",
                },
            },
            "published": {
                Required: true,
                Filterable: true,
                Default: false,
                Validator: &schema.Bool{},
            },
            "title": {
                Required: true,
                Validator: &schema.String{
                    MaxLen: 150,
                },
            },
            "body": {
                // Dependency defines that body field can't be changed if
                // the published field is not "false".
                Dependency: query.MustParsePredicate(`{"published": false}`),
                Validator: &schema.String{
                    MaxLen: 100000,
                },
            },
        },
    }
)

func main() {
    // Create a REST API resource index
    index := resource.NewIndex()

    // Add a resource on /users[/:user_id]
    users := index.Bind("users", user, mem.NewHandler(), resource.Conf{
        // We allow all REST methods
        // (rest.ReadWrite is a shortcut for []resource.Mode{resource.Create,
        //  resource.Read, resource.Update, resource.Delete, resource,List})
        AllowedModes: resource.ReadWrite,
    })

    // Bind a sub resource on /users/:user_id/posts[/:post_id]
    // and reference the user on each post using the "user" field of the posts resource.
    users.Bind("posts", "user", post, mem.NewHandler(), resource.Conf{
        // Posts can only be read, created and deleted, not updated
        AllowedModes: []resource.Mode{resource.Read, resource.List,
             resource.Create, resource.Delete},
    })

    // Create API HTTP handler for the resource graph
    api, err := rest.NewHandler(index)
    if err != nil {
        log.Fatalf("Invalid API configuration: %s", err)
    }

    // Bind the API under /api/ path
    http.Handle("/api/", http.StripPrefix("/api/", api))

    // Serve it
    log.Print("Serving API on http://localhost:8080")
    if err := http.ListenAndServe(":8080", nil); err != nil {
        log.Fatal(err)
    }
}
```

Just run this code (or use the provided [examples/demo](https://github.com/clarify/rested/blob/master/examples/demo/main.go)):

```sh
$ go run examples/demo/main.go
2015/07/27 20:54:55 Serving API on http://localhost:8080
```

Using [HTTPie](http://httpie.org/), you can now play with your API.

First create a user:

```sh
$ http POST :8080/api/users name="John Doe"
HTTP/1.1 201 Created
Content-Length: 155
Content-Location: /api/users/ar6ejgmkj5lfl98r67p0
Content-Type: application/json
Date: Mon, 27 Jul 2015 19:10:20 GMT
Etag: "1e18e148e1ff3ecdaae5ec03ac74e0e4"
Last-Modified: Mon, 27 Jul 2015 19:10:20 GMT
Vary: Origin

{
    "id": "ar6ejgmkj5lfl98r67p0",
    "created": "2015-07-27T21:10:20.671003126+02:00",
    "updated": "2015-07-27T21:10:20.671003989+02:00",
    "name": "John Doe",
}
```

As you can see, the `id`, `created` and `updated` fields have been automatically generated by our `OnInit` field hooks.

Also notice the `Etag` and `Last-Modified` headers. Those guys allow data integrity and concurrency control _down to the storage layer_ through the use of the `If-Match` and `If-Unmodified-Since` headers. They can also serve for conditional requests using `If-None-Match` and `If-Modified-Since` headers.

Here is an example of conditional request:

```sh
$ http :8080/api/users/ar6ejgmkj5lfl98r67p0 \
  If-Modified-Since:"Mon, 27 Jul 2015 19:10:20 GMT"
HTTP/1.1 304 Not Modified
Date: Mon, 27 Jul 2015 19:17:11 GMT
Vary: Origin
```

And here is a data integrity request following the [RFC-5789](http://tools.ietf.org/html/rfc5789) recommendations:

```sh
$ http PATCH :8080/api/users/ar6ejgmkj5lfl98r67p0 \
  name="Someone Else" If-Match:invalid-etag
HTTP/1.1 412 Precondition Failed
Content-Length: 58
Content-Type: application/json
Date: Mon, 27 Jul 2015 19:33:27 GMT
Vary: Origin

{
    "code": 412,
    "fields": null,
    "message": "Precondition Failed"
}
```

Retry with the valid etag:

```sh
$ http PATCH :8080/api/users/ar6ejgmkj5lfl98r67p0 \
  name="Someone Else" If-Match:'"1e18e148e1ff3ecdaae5ec03ac74e0e4"'

HTTP/1.1 200 OK
Content-Length: 159
Content-Type: application/json
Date: Mon, 27 Jul 2015 19:36:19 GMT
Etag: "7bb7a71b0f66197aa07c4c8fc9564616"
Last-Modified: Mon, 27 Jul 2015 19:36:19 GMT
Vary: Origin

{
    "created": "2015-07-27T21:33:09.168492448+02:00",
    "id": "ar6ejmukj5lflde9q8bg",
    "name": "Someone Else",
    "updated": "2015-07-27T21:36:19.904545093+02:00"
}
```

Note that even if you don't use conditional request, the `Etag` is always used by the storage handler to manage concurrency control between requests.

Another cool thing is sub-resources. We've set our `posts` resource as a child of the `users` resource. This way we can handle ownership very easily as routes are constructed as `/users/:user_id/posts`.

Lets create a post:

```sh
$ http POST :8080/api/users/ar6ejgmkj5lfl98r67p0/posts \
  title="My first post"
HTTP/1.1 200 OK
Content-Length: 212
Content-Type: application/json
Date: Mon, 27 Jul 2015 19:46:55 GMT
Etag: "307ae92df6c3dd54847bfc7d72422e07"
Last-Modified: Mon, 27 Jul 2015 19:46:55 GMT
Vary: Origin

{
    "id": "ar6ejs6kj5lflgc28es0",
    "created": "2015-07-27T21:46:55.355857401+02:00",
    "updated": "2015-07-27T21:46:55.355857989+02:00",
    "title": "My first post",
    "user": "ar6ejgmkj5lfl98r67p0"
}
```

Notice how the `user` field has been set with the user id provided in the route, that's pretty cool, huh?

We defined that we can create posts but we can't modify them, lets verify that:

```sh
$ http PATCH :8080/api/users/821d…/posts/ar6ejs6kj5lflgc28es0 \
  private=true
HTTP/1.1 405 Method Not Allowed
Content-Length: 53
Content-Type: application/json
Date: Mon, 27 Jul 2015 19:50:33 GMT
Vary: Origin

{
    "code": 405,
    "fields": null,
    "message": "Invalid method"
}
```

Let's list posts for that user now:

```sh
$ http :8080/api/users/ar6ejgmkj5lfl98r67p0/posts
HTTP/1.1 200 OK
Content-Length: 257
Content-Type: application/json
Date: Mon, 27 Jul 2015 19:51:46 GMT
Vary: Origin
X-Total: 1

[
    {
        "id": "ar6ejs6kj5lflgc28es0",
        "_etag": "307ae92df6c3dd54847bfc7d72422e07",
        "created": "2015-07-27T21:46:55.355857401+02:00",
        "updated": "2015-07-27T21:46:55.355857989+02:00",
        "title": "My first post",
        "user": "ar6ejgmkj5lfl98r67p0"
    }
]
```

Notice the added `_etag` field. This is to let you get etags of multiple items without having to `GET` each one of them through individual requests.

Now, let's get user's information for each posts in a single request:

```sh
$ http :8080/api/users/ar6ejgmkj5lfl98r67p0/posts fields=='id,title,user{id,name}'
HTTP/1.1 200 OK
Content-Length: 257
Content-Type: application/json
Date: Mon, 27 Jul 2015 19:51:46 GMT
Vary: Origin
X-Total: 1

[
    {
        "id": "ar6ejs6kj5lflgc28es0",
        "_etag": "307ae92df6c3dd54847bfc7d72422e07",
        "created": "2015-07-27T21:46:55.355857401+02:00",
        "updated": "2015-07-27T21:46:55.355857989+02:00",
        "title": "My first post",
        "user": {
            "id": "ar6ejgmkj5lfl98r67p0",
            "name": "John Doe"
        }
    }
]
```

Notice how we selected which fields we wanted in the result using the [field selection](#field-selection) query format. Thanks to sub-request support, the user name is included with each post with no additional HTTP request.

We can go even further and embed a sub-request list responses. Let's say we want a list of users with the last two posts:

```sh
$ http GET :8080/api/users fields='id,name,posts(limit:2){id,title}'
HTTP/1.1 201 Created
Content-Length: 155
Content-Location: /api/users/ar6ejgmkj5lfl98r67p0
Content-Type: application/json
Date: Mon, 27 Jul 2015 19:10:20 GMT
Etag: "1e18e148e1ff3ecdaae5ec03ac74e0e4"
Last-Modified: Mon, 27 Jul 2015 19:10:20 GMT
Vary: Origin

[
    {
        "id": "ar6ejgmkj5lfl98r67p0",
        "name": "John Doe",
        "posts": [
            {
                "id": "ar6ejs6kj5lflgc28es0",
                "title": "My first post"
            },
            {
                "id": "ar6ek26kj5lfljgh84qg",
                "title": "My second post"
            }
        ]
    }
]
```

Sub-requests are executed concurrently whenever possible to ensure the fastest response time.

## Resource Configuration

For REST Layer to be able to expose resources, you have to first define what fields the resource contains and where to bind it in the REST API URL namespace.

### Schema

Resource field configuration is performed through the [schema](https://pkg.go.dev/github.com/clarify/rested/schema) package. A schema is a struct describing a resource. A schema is composed of metadata about the resource and a description of the allowed fields through a map of field name pointing to field definition.

Sample resource schema:

```go
foo = schema.Schema{
    Description: "A foo object",
    Fields: schema.Fields{
        "field_name": {
            Required: true,
            Filterable: true,
            Validator: &schema.String{
                MaxLen: 150,
            },
        },
    },
}
```

Schema fields:

| Field         | Description                                                          |
| ------------- | -------------------------------------------------------------------- |
| `Description` | The description of the resource. This is used for API documentation. |
| `Fields`      | A map of field name to field definition.                             |

### Field Definition

The field definitions contains the following properties:

| Field        | Description                                                                                                                                                                                                                                                                                                                                                                         |
| ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Required`   | If `true`, the field must be provided when the resource is created and can't be set to `null`. The client may be able to omit a required field if a `Default` or a hook sets its content.                                                                                                                                                                                           |
| `ReadOnly`   | If `true`, the field can not be set by the client, only a `Default` or a hook can alter its value. You may specify a value for a read-only field in your mutation request if the value is equal to the old value, REST Layer won't complain about it. This lets your client `PUT` the same document it got with `GET` without having to take care of removing the read-only fields. |
| `Hidden`     | Hidden allows writes but hides the field's content from the client. When this field is enabled, PUTing the document without the field would not remove the field but use the previous document's value if any.                                                                                                                                                                      |
| `Default`    | The value to be set when resource is created and the client didn't provide a value for the field. The content of this variable must still pass validation.                                                                                                                                                                                                                          |
| `OnInit`     | A function to be executed when the resource is created. The function gets the current value of the field (after `Default` has been set if any) and returns the new value to be set.                                                                                                                                                                                                 |
| `OnUpdate`   | A function to be executed when the resource is updated. The function gets the current (updated) value of the field and returns the new value to be set.                                                                                                                                                                                                                             |
| `Params`     | Params defines the list of parameters allowed for this field. See [Field Parameters](#field-parameters) section for some examples.                                                                                                                                                                                                                                                  |
| `Handler`    | Handler defines a function able to change the field's value depending on the passed parameters. See [Field Parameters](#field-parameters) section for some examples.                                                                                                                                                                                                                |
| `Validator`  | A `schema.FieldValidator` to validate the content of the field.                                                                                                                                                                                                                                                                                                                     |
| `Dependency` | A query using `filter` format created with `` query.MustParsePredicate(`{"field": "value"}`) ``. If the query doesn't match the document, the field generates a dependency error.                                                                                                                                                                                                   |
| `Filterable` | If `true`, the field can be used with the `filter` parameter. You may want to ensure the backend database has this field indexed when enabled. Some storage handlers may not support all the operators of the filter parameter, see their documentation for more information.                                                                                                       |
| `Sortable`   | If `true`, the field can be used with the `sort` parameter. You may want to ensure the backend database has this field indexed when enabled.                                                                                                                                                                                                                                        |
| `Schema`     | An optional sub schema to validate hierarchical documents.                                                                                                                                                                                                                                                                                                                          |

REST Layer comes with a set of validators. You can add your own by implementing the `schema.FieldValidator` interface. Here is the list of provided validators:

| Validator               | Description                                                           |
| ----------------------- | --------------------------------------------------------------------- |
| [schema.String][str]    | Ensures the field is a string                                         |
| [schema.Integer][int]   | Ensures the field is an integer                                       |
| [schema.Float][float]   | Ensures the field is a float                                          |
| [schema.Bool][bool]     | Ensures the field is a Boolean                                        |
| [schema.Array][array]   | Ensures the field is an array                                         |
| [schema.Dict][dict]     | Ensures the field is a dict                                           |
| [schema.Object][object] | Ensures the field is an object validating against a sub-schema        |
| [schema.Time][time]     | Ensures the field is a datetime                                       |
| [schema.URL][url]       | Ensures the field is a valid URL                                      |
| [schema.IP][url]        | Ensures the field is a valid IPv4 or IPv6                             |
| [schema.Password][pswd] | Ensures the field is a valid password and bcrypt it                   |
| [schema.Reference][ref] | Ensures the field contains a reference to another _existing_ API item |
| [schema.AnyOf][any]     | Ensures that at least one sub-validator is valid                      |
| [schema.AllOf][all]     | Ensures that at least all sub-validators are valid                    |

[str]: https://pkg.go.dev/github.com/clarify/rested/schema#String
[int]: https://pkg.go.dev/github.com/clarify/rested/schema#Integer
[float]: https://pkg.go.dev/github.com/clarify/rested/schema#Float
[bool]: https://pkg.go.dev/github.com/clarify/rested/schema#Bool
[array]: https://pkg.go.dev/github.com/clarify/rested/schema#Array
[dict]: https://pkg.go.dev/github.com/clarify/rested/schema#Dict
[object]: https://pkg.go.dev/github.com/clarify/rested/schema#Object
[time]: https://pkg.go.dev/github.com/clarify/rested/schema#Time
[url]: https://pkg.go.dev/github.com/clarify/rested/schema#URL
[ip]: https://pkg.go.dev/github.com/clarify/rested/schema#IP
[pswd]: https://pkg.go.dev/github.com/clarify/rested/schema#Password
[ref]: https://pkg.go.dev/github.com/clarify/rested/schema#Reference
[any]: https://pkg.go.dev/github.com/clarify/rested/schema#AnyOf
[all]: https://pkg.go.dev/github.com/clarify/rested/schema#AllOf

Some common hook handler to be used with `OnInit` and `OnUpdate` are also provided:

| Hook           | Description                                                                                 |
| -------------- | ------------------------------------------------------------------------------------------- |
| `schema.Now`   | Returns the current time ignoring the input (current) value.                                |
| `schema.NewID` | Returns a unique identifier using [xid](https://github.com/rs/xid) if input value is `nil`. |

Some common field configuration are also provided as variables:

| Field Config           | Description                                                                                                                                            |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `schema.IDField`       | A required, read-only field with `schema.NewID` set as `OnInit` hook and a `schema.String` validator matching [xid](https://github.com/rs/xid) format. |
| `schema.CreatedField`  | A required, read-only field with `schema.Now` set on `OnInit` hook with a `schema.Time` validator.                                                     |
| `schema.UpdatedField`  | A required, read-only field with `schema.Now` set on `OnInit` and `OnUpdate` hooks with a `schema.Time` validator.                                     |
| `schema.PasswordField` | A hidden, required field with a `schema.Password` validator.                                                                                           |

Here is an example of schema declaration:

```go
// Define a post resource schema
post = schema.Schema{
    Fields: schema.Fields{
        // schema.*Field are shortcuts for common fields (identical to users' same fields)
        "id":      schema.IDField,
        "created": schema.CreatedField,
        "updated": schema.UpdatedField,
        // Define a user field which references the user owning the post.
        // See bellow, the content of this field is enforced by the fact
        // that posts is a sub-resource of users.
        "user": {
            Required: true,
            Filterable: true,
            Validator: &schema.Reference{
                Path: "users",
            },
        },
        // Sub-documents are handled via a sub-schema
        "meta": {
            Schema: &schema.Schema{
                Fields: schema.Fields{
                    "title": {
                        Required: true,
                        Validator: &schema.String{
                            MaxLen: 150,
                        },
                    },
                    "body": {
                        Validator: &schema.String{
                            MaxLen: 100000,
                        },
                    },
                },
            },
        },
    },
}
```

### Binding

Now you just need to bind this schema at a specific endpoint on the [resource.Index](https://pkg.go.dev/github.com/clarify/rested/resource#Index) object:

```go
index := resource.NewIndex()
posts := index.Bind("posts", post, mem.NewHandler(), resource.DefaultConf)
```

This tells the `resource.Index` to bind the `post` schema at the `posts` endpoint. The resource collection URL is then `/posts` and item URLs are `/posts/<post_id>`.

The [resource.DefaultConf](https://pkg.go.dev/github.com/clarify/rested/resource#pkg-variables) variable is a pre-defined [resource.Conf](https://pkg.go.dev/github.com/clarify/rested/resource#Conf) type with sensible defaults. You can customize the resource behavior using a custom configuration.

The `resource.Conf` type has the following customizable properties:

| Property                 | Description                                                                                                                                                                                                                  |
| ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `AllowedModes`           | A list of `resource.Mode` allowed for the resource.                                                                                                                                                                          |
| `PaginationDefaultLimit` | If set, pagination is enabled for list requests by default with the number of item per page as defined here. Note that the default ony applies to list (GET) requests, i.e. it does _not_ apply for clear (DELETE) requests. |
| `ForceTotal`             | Control the behavior of the computation of `X-Total` header and the `total` query-string parameter. See `resource.ForceTotalMode` for available options.                                                                     |

### Modes

REST Layer handles mapping of HTTP methods to your resource URLs automatically. With REST, there is two kind of resource URL paths: collection and item URLs. Collection URLs (`/<resource>`) are pointing to the collection of items, while item URL (`/<resource>/<item_id>`) points to a specific item in that collection. HTTP methods are used to perform [CRUDL](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete) operations on those resources.

You can easily dis/allow an operation on a per resource basis using `resource.Conf`'s `AllowedModes` property. The use of modes instead of HTTP methods in the configuration adds a layer of abstraction necessary to handle specific cases like `PUT` HTTP method performing a `create` if the specified item does not exist or a `replace` if it does. This gives you precise control of what you want to allow or not.

Modes are passed as configuration to resources as follow:

```go
users := index.Bind("users", user, mem.NewHandler(), resource.Conf{
    AllowedModes: []resource.Mode{resource.Read, resource.List, resource.Create, resource.Delete},
})
```

The following table shows how REST layer maps CRUDL operations to HTTP methods and `modes`:

| Mode      | HTTP Method | Context    | Description                                                                                                                          |
| --------- | ----------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `Read`    | GET         | Item       | Get an individual item by its ID.                                                                                                    |
| `List`    | GET         | Collection | List/find items using filters and sorts.                                                                                             |
| `Create`  | POST        | Collection | Create an item letting the system generate its ID.                                                                                   |
| `Create`  | PUT         | Item       | Create an item by choosing its ID.                                                                                                   |
| `Update`  | PATCH       | Item       | Partially modify the item following [RFC-5789](http://tools.ietf.org/html/rfc5789), [RFC-6902](https://tools.ietf.org/html/rfc6902). |
| `Replace` | PUT         | Item       | Replace the item by a new on.                                                                                                        |
| `Delete`  | DELETE      | Item       | Delete the item by its ID.                                                                                                           |
| `Clear`   | DELETE      | Collection | Delete all items from the collection matching the context and/or filters.                                                            |

### Hooks

Hooks are piece of code you can attach before or after an operation is performed on a resource. A hook is a Go type implementing one of the event handler interface below, and attached to a resource via the [Resource.Use](https://pkg.go.dev/github.com/clarify/rested/resource#Resource.Use) method.

| Hook Interface         | Description                                                                                                                                                                      |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [FindEventHandler]     | Defines a function called when the resource is listed with or without a query. Note that hook is called for both resource and item fetch as well a prior to updates and deletes. |
| [FoundEventHandler]    | Defines a function called with the result of a find on resource.                                                                                                                 |
| [GetEventHandler]      | Defines a function called when a get is performed on an item of the resource. Note: when multi-get is performed this hook is called for each items id individually.              |
| [GotEventHandler]      | Defines a function called with the result of a get on a resource.                                                                                                                |
| [InsertEventHandler]   | Defines a function called before an item is inserted.                                                                                                                            |
| [InsertedEventHandler] | Defines a function called after an item has been inserted.                                                                                                                       |
| [UpdateEventHandler]   | Defines a function called before an item is updated.                                                                                                                             |
| [UpdatedEventHandler]  | Defines a function called after an item has been updated.                                                                                                                        |
| [DeleteEventHandler]   | Defines a function called before an item is deleted.                                                                                                                             |
| [DeletedEventHandler]  | Defines a function called after an item has been deleted.                                                                                                                        |
| [ClearEventHandler]    | Defines a function called before a resource is cleared.                                                                                                                          |
| [ClearedEventHandler]  | Defines a function called after a resource has been cleared.                                                                                                                     |

[findeventhandler]: https://pkg.go.dev/github.com/clarify/rested/resource#FindEventHandler
[foundeventhandler]: https://pkg.go.dev/github.com/clarify/rested/resource#FoundEventHandler
[geteventhandler]: https://pkg.go.dev/github.com/clarify/rested/resource#GetEventHandler
[goteventhandler]: https://pkg.go.dev/github.com/clarify/rested/resource#GotEventHandler
[inserteventhandler]: https://pkg.go.dev/github.com/clarify/rested/resource#InsertEventHandler
[insertedeventhandler]: https://pkg.go.dev/github.com/clarify/rested/resource#InsertedEventHandler
[updateeventhandler]: https://pkg.go.dev/github.com/clarify/rested/resource#UpdateEventHandler
[updatedeventhandler]: https://pkg.go.dev/github.com/clarify/rested/resource#UpdatedEventHandler
[deleteeventhandler]: https://pkg.go.dev/github.com/clarify/rested/resource#DeleteEventHandler
[deletedeventhandler]: https://pkg.go.dev/github.com/clarify/rested/resource#DeletedEventHandler
[cleareventhandler]: https://pkg.go.dev/github.com/clarify/rested/resource#ClearEventHandler
[clearedeventhandler]: https://pkg.go.dev/github.com/clarify/rested/resource#ClearedEventHandler

Note that these are resource level hooks, and do not correspond one-to-one to `rest` operations. Note that a HTTP request to `GET` an item by ID, will result in a `Find` and not a `Get` call which will triggering the `OnFind` and `OnFound` hooks to be called, not `OnGet` and `OnGot`. Similarly, a `PATCH` or `PUT` request will call `Find` before it calls `Update`, which will trigger the same hooks. If your hooks logic require knowing which rest-level operation is performed see [rest.RouteFromContext](https://pkg.go.dev/github.com/clarify/rested/rest#RouteFromContext)

All hooks functions get a `context.Context` as first argument. If a network call must be performed from the hook, the context's deadline must be respected. If a hook returns an error, the whole request is aborted with that error. You can also use the context to pass data to your hooks from a middleware executed before REST Layer. This can be used to manage authentication for instance. See [examples/auth](https://github.com/clarify/rested/blob/master/examples/auth/main.go) to see an example.

Hooks that get passed both an an error and/or an item, such as [GotEventHandler], [UpdatedEventHandler], [DeletedEventHandler] should insert guards to handle the error being set and/or the item not being set; both can be true in some cases. It's also allowed to set items or errors to nil, which is why double pointers are often used.

```go
func (hook Hook) OnGot(ctx context.Context, item **resource.Item, err *error) {
    // Guard.
    if *err != nil || *item == nil {
        return
    }
    // ...
}
```

```go
func (hook Hook) OnGot(ctx context.Context, item **resource.Item, err *error) {
    // Overriding an error response.
    if *err != nil || *item == nil {
        (*err) = nil
        (*item) = fallbackItem()
    }
    // ...
}
```

### Sub Resources

Sub resources can be used to express a one-to-may parent-child relationship between two resources. A sub-resource is automatically filtered by its parent on the field specified as second argument of the `Bind` method.

To create a sub-resource, you bind you resource on the object returned by the binding of the parent resource. For instance, here we bind a `comments` resource to a `posts` resource:

```go
posts := index.Bind("posts", post, mem.NewHandler(), resource.DefaultConf)
// Bind comment as sub-resource of the posts resource
posts.Bind("comments", "post", comment, mem.NewHandler(), resource.DefaultConf)
```

The second argument `post` defines the field in the `comments` resource that refers to the parent. This field must be present in the resource and the backend storage must support filtering on it. As a result, we get a new hierarchical route as follow:

```txt
/posts/:post_id/comments[/:comment_id]
```

When performing a `GET` on `/posts/:post_id/comments`, it is like adding the filter `{"post":"<post_id>"}` to the request to comments resource.

Additionally, thanks to REST Layer's [embedding](#embedding), this relationship can be embedded in the parent object as a sub-query:

```txt
/posts?fields=id,title,comments(limit=5,sort=-updated){id,user{id,name},message}
```

Here we would get all post with their respective 5 last comments embedded in the `comments` field of each post object with the user commenting to post embedded in each comment's sub-document:

```json
[
  {
    "id": "abc",
    "comments": [
      {
        "id": "def",
        "user": {
          "id": "ghi",
          "name": "John Doe"
        },
        "message": "Last comment"
      }
    ]
  }
]
```

See [embedding](#embedding) for more information.

### Dependency

Fields can depend on other fields in order to be changed. To configure a dependency, set a filter on the `Dependency` property of the field using the [query.MustParsePredicate()](https://pkg.go.dev/github.com/clarify/rested/schema/queru#MustParsePredicate) method.

In this example, the `body` field can't be changed if the `published` field is not set to `true`:

```go
post = schema.Schema{
    Fields: schema.Fields{
        "published": schema.Field{
            Validator:  &schema.Bool{},
        },
        "body": {
            Dependency: query.MustParsePredicate(`{"published": true}`),
            Validator:  &schema.String{},
        },
    },
}
```

## HTTP Request Headers

### Prefer

Currently supported values are:

- [return=minimal](https://tools.ietf.org/html/rfc7240#section-4.2): When a request is successfully (HTTP Response Status of `200` or `201`), response body is not returned. For Response Status of `200 OK`, status becomes `204 No Content`. Can be used for e.g `PUT`, `POST` and `PATCH` methods, where returned body will be known by the client.
- [return=no-content](https://msdn.microsoft.com/en-us/library/hh537533.aspx): same as `return=minimal`.

```sh
$ echo '[{"op": "add", "path":"/foo", "value": "bar"}]' | http PATCH :8080/users/ar6ej4mkj5lfl688d8lg If-Match:'"1234567890123456789012345678901234567890"' \
Content-Type: application/json-patch+json \
Prefer: return=minimal
HTTP/1.1 204 No Content
```

### Content-Type

The Content-Type of the request body. Most HTTP methods only support `"aplication/json"` by default, but `PUT` requests also allow `"application/json-patch+json"`.

## HTTP Request Methods

Following HTTP Methods are currently supported by rest-layer.

### OPTIONS

Used to tell the client which HTTP Methods are supported for any given path.

### HEAD

The same as `GET`, except it includes only headers in the response.

### GET

Used to retrieve a (projected[#field-selection]) resource document by specifying it's `ID` in the path, or to retrieve a paginated view of documents matching a [query](#quering).

### POST

Used to create new resource document when the `ID` can be generated by the server. Field default values are set for omitted fields, and `OnCreate` field hooks are issued.

### PUT

Used to create or update a single resource document by specifying it's `ID` in the path. Field default values are set for omitted fields. If the document did not previously exist `OnCreate` field hooks are issued, otherwise `OnUpdate` field hooks are issued.

`If-Match` [concurrency protection](#data-integrity-and-concurrency-control) could be used if relevant.

### PATCH

Used to create or patch a single resource document by specifying it's `ID` in the path. `OnUpdate` field hooks are issued.

REST Layer supports two PATCH protocols, that can be specified via the `Content-Type` header.

- Simple filed replacement [RFC-5789](http://tools.ietf.org/html/rfc5789) - this protocol will update only supplied top level fields, and will leave other fields in the document intact. This means that this protocol can't delete fields. Using this protocol is specified with `Content-Type: application/json` HTTP Request header.

- [JSON-Patch/RFC-6902](https://tools.ietf.org/html/rfc6902) - When patching deeply nested documents, it is more convenient to use protocol designed especially for this. Using this protocol is specified with `Content-Type: application/json-patch+json` HTTP Request header.

`If-Match` [concurrency protection](#data-integrity-and-concurrency-control) could be used if relevant.

Example JSON Patch Request where we utilize concurrency control ask for the response body to be omitted:

```sh
$ echo '[{"op": "add", "path":"/foo", "value": "bar"}]' | http PATCH :8080/users/ar6ej4mkj5lfl688d8lg If-Match:'"1234567890123456789012345678901234567890"' \
Content-Type: application/json-patch+json \
Prefer: return=minimal
HTTP/1.1 204 No Content
```

### DELETE

Used to delete single resource document given its `ID`, or multiple documents matching a [query](#quering).

## Querying

When supplying query parameters be sure to honor URL encoding scheme. If you need to include `+` sign, use `%2B`, etc.

### Filtering

To filter resources, you use the `filter` query-string parameter. The format of the parameter is inspired by the [MongoDB query format](http://docs.mongodb.org/manual/tutorial/query-documents/). The `filter` parameter can be used with `GET` and `DELETE` methods on resource URLs.

To use a resource field with the `filter` parameter, the field must be defined on the resource and the `Filterable` field property must be set to `true`. You may want to ensure the backend database has this field indexed when enabled.

To specify equality condition, use the query `{<field>: <value>}` to select all items with `<field>` equal `<value>`. REST Layer will complain with a `422` HTTP error if any field queried is not defined in the resource schema or is using an operator incompatible with field type (i.e.: `$lt` on a string field).

A query can specify conditions for more than one field. Implicitly, a logical `AND` conjunction connects the clauses so that the query selects the items that match all the conditions.

It is also possible to use an explicit `$and` operator to join each clause with a logical `AND`. There are sometimes good use-cases for this, such as when joining two independent `$or` queries that must both match, or when programmatically merging multiple queries with potentially overlapping fields.

```js
{$and: [
  {$or: [{quantity: {$gt: 100}}, {price: {$lt: 9.95}}]},
  {$or: [{length: {$lt: 1000}}, {width: {$lt: 1000}}
]}
```

Using the the `$or` operator, you can specify a compound query that joins each clause with a logical `OR` conjunction so that the query selects the items that match at least one condition.

In the following example, the query document selects all items in the collection where the field `quantity` has a value greater than (`$gt`) `100` or the value of the `price` field is less than (`$lt`) `9.95`:

```js
{
  $or: [{ quantity: { $gt: 100 } }, { price: { $lt: 9.95 } }];
}
```

Match on sub-fields is performed through field path separated by dots. This example shows an exact match on the sub-fields `country` and `city` of the `address` sub-document:

```js
{address.country: "France", address.city: "Paris"}
```

Some operators can change the type of match. For instance `$in` can be used to match a field against several values. For instance, to select all items with the `type` field equal either `food` or `snacks`, use the following query:

```js
{
  type: {
    $in: ["food", "snacks"];
  }
}
```

The opposite `$nin` is also available.

The following numeric comparisons operators are supported: `$lt`, `$lte`, `$gt`, `$gte`.

The `$exists` operator matches documents containing the field, even if this field is `null`.

```js
{
  type: {
    $exists: true;
  }
}
```

You can invert the operator by passing `false`.

There is also a `$regex` operator that matches documents containing the field given as a regular expression. In general, the syntax of the regular expressions accepted is the same general syntax used by Perl, Python, and other languages.
More precisely, it is the syntax accepted by RE2 and described at [https://golang.org/s/re2syntax](https://golang.org/s/re2syntax), except for `\C`.

Flags are supported for more control over regular expressions. Flag syntax is xyz (set) or -xyz (clear) or xy-z (set xy, clear z).
The flags are:

| Flag | Mode                                                                        | Default |
| ---- | --------------------------------------------------------------------------- | ------- |
| `i`  | case-insensitive                                                            | false   |
| `m`  | multi-line mode: ^ and $ match begin/end line in addition to begin/end text | false   |
| `s`  | let . match \n                                                              | false   |
| `U`  | non-greedy: swap meaning of x\* and x\*?, x+ and x+?, etc                   | false   |

For example the following regular expression would match any document with a field `type` and its value `rest-layer`.

```js
{
  type: {
    $regex: "re[s]{1}t-la.+r";
  }
}
```

The same example with flags:

```js
{
  type: {
    $regex: "(?i)re[s]{1}t-LAYER";
  }
}
```

However, keep in mind that Storers have to support regular expression and depending on the implementation of the storage handler the accepted syntax may vary.
An error of `ErrNotImplemented` will be returned for those storage back-ends which do not support the `$regex` operator.

The `$elemMatch` operator matches documents that contain an array field with at least one element that matches all the specified query criteria.

```go
            "telephones": schema.Field{
                Filterable: true,
                Validator: &schema.Array{
                    Values: schema.Field{
                        Validator:  &schema.Object{Schema: &Telephone},
                    },
                },
            },
```

Matching documents that contain specific values within array objects can be done with `$elemMatch`:

```js
{telephones: {$elemMatch: {name: "John Snow", active: true}}}
```

The snippet above will return all documents, which `telephones` array field contains objects that have `name` **_AND_** `active` fields matching queried values.

> Note that documents returned may contain other objects in `telephones` that don't match the query above, but at least one object will do. Further filtering could be needed on the API client side.

#### _$elemMatch_ Limitation

`$elemMatch` will work only for arrays of objects for now. Later it could be extended to work on plain arrays e.g:

```js
{
  numbers: {
    $elemMatch: {
      $gt: 20;
    }
  }
}
```

#### Filter operators

| Operator     | Usage                           | Description                                                                           |
| ------------ | ------------------------------- | ------------------------------------------------------------------------------------- |
| `$or`        | `{$or: [{a: "b"}, {a: "c"}]}`   | Join two clauses with a logical `OR` conjunction.                                     |
| `$and`       | `{$and: [{a: "b"}, {b: "c"}]}`  | Join two clauses with a logical `AND` conjunction.                                    |
| `$in`        | `{a: {$in: ["b", "c"]}}`        | Match a field against several values.                                                 |
| `$nin`       | `{a: {$nin: ["b", "c"]}}`       | Opposite of `$in`.                                                                    |
| `$lt`        | `{a: {$lt: 10}}`                | Fields value is lower than specified number.                                          |
| `$lte`       | `{a: {$lte: 10}}`               | Fields value is lower than or equal to the specified number.                          |
| `$gt`        | `{a: {$gt: 10}}`                | Fields value is greater than specified number.                                        |
| `$gte`       | `{a: {$gte: 10}}`               | Fields value is greater than or equal to the specified number.                        |
| `$exists`    | `{a: {$exists: true}}`          | Match if the field is present (or not if set to `false`) in the item, event if `nil`. |
| `$regex`     | `{a: {$regex: "fo[o]{1}"}}`     | Match regular expression on a field's value.                                          |
| `$elemMatch` | `{a: {$elemMatch: {b: "foo"}}}` | Match array items against multiple query criteria.                                    |

_Some storage handlers may not support all operators. Refer to the storage handler's documentation for more info._

### Sorting

Sorting of resource items is defined through the `sort` query-string parameter. The `sort` value is a list of resource's fields separated by comas (`,`). To invert a field's sort, you can prefix its name with a minus (`-`) character. The `sort` parameter can be used with `GET` and `DELETE` methods on resource URLs.

To use a resource field with the `sort` parameter, the field must be defined on the resource and the `Sortable` field property must be set to `true`. You may want to ensure the backend database has this field indexed when enabled.

Here we sort the result by ascending quantity and descending create time:

```txt
/posts?sort=quantity,-created
```

### Field Selection

REST APIs tend to grow over time. Resources get more and more fields to fulfill the needs for new features. But each time fields are added, all existing API clients automatically get the additional cost. This tend to lead to huge waste of bandwidth and added latency due to the transfer of unnecessary data. As a workaround, the `field` parameter can be used to minimize and customize the response body from requests with a `GET`, `POST`, `PUT` or `PATCH` method on resource URLs.

REST Layer provides a powerful fields selection (also named projection) system. If you provide the `fields` parameter with a list of fields for the resource you are interested in separated by commas, only those fields will be returned in the document:

```sh
$ http -b :8080/api/users/ar6eimekj5lfktka9mt0 fields=='id,name'
{
    "id": "ar6eimekj5lfktka9mt0",
    "name": "John Doe"
}
```

If your document has sub-fields, you can use brackets to select sub-fields:

```sh
$ http -b :8080/api/users/ar6eimekj5lfktka9mt0/posts fields=='meta{title,body}'
[
    {
        "_etag": "ar6eimukj5lfl07r0uv0",
        "meta": {
            "title": "test",
            "body": "example"
        }
    }
]
```

Also `all fields` expansion is supported:

```sh
$ http -b :8080/api/users/ar6eimekj5lfktka9mt0/posts fields=='*,user{*}'
[
    {
        "_etag": "ar6eimukj5lfl07r0uv0",
        "id": "ar6eimukj5lfl07r0ugz",
        "created": "2015-07-27T21:46:55.355857401+02:00",
        "updated": "2015-07-27T21:46:55.355857989+02:00",
        "user": {
          "id": "ar6eimukj5lfl07gzb0b",
          "created": "2015-07-24T21:46:55.355857401+02:00",
          "updated": "2015-07-24T21:46:55.355857989+02:00",
          "name": "John Snow",
        },
        "meta": {
            "title": "test",
            "body": "example"
        }
    }
]
```

#### Field Aliasing

It's also possible to rename fields in the response using aliasing. To create an alias, prefix the field name by the wanted alias separated by a colon (`:`):

```sh
$ http -b :8080/api/users/ar6eimekj5lfktka9mt0 fields=='id,name,n:name'
{
    "id": "ar6eimekj5lfktka9mt0",
    "n": "John Doe",
    "name": "John Doe"
}
```

As you see, you can specify the same field several times. It doesn't seem useful in this example, but with [fields parameters](#field-parameters), it becomes very powerful (see below).

Aliasing works with sub-fields as well:

```sh
$ http -b :8080/api/users/ar6eimekj5lfktka9mt0/posts fields=='meta{title,b:body}'
[
    {
        "_etag": "ar6eimukj5lfl07r0uv0",
        "meta": {
            "title": "test",
            "b": "example"
        }
    }
]
```

#### Field Parameters

Field parameters are used to apply a transformation on the value of a field using custom logic.

For instance, if you are using an on demand dynamic image resizer, you may want to expose the capability of this service, without requiring from the client to learn another URL based API. Wouldn't it be better if we could just ask the API to return the `thumbnail_url` dynamically transformed with the desired dimensions?

By combining field aliasing and field parameters, we can expose this resizer API as follow:

```sh
$ http -b :8080/api/videos fields=='id,
                                    thumb_small_url:thumbnail_url(width:80,height:60),
                                    thumb_large_url:thumbnail_url(width:800,height:600)'
[
    {
        "_etag": "ar6eimukj5lfl07r0uv0",
        "thumb_small_url": "http://cdn.com/path/to/image-80w60h.jpg",
        "thumb_large_url": "http://cdn.com/path/to/image-800w600h.jpg"
    }
]
```

The example above show the same field represented twice but with some useful value transformations.

To add parameters on a field, use the `Params` property of the `schema.Field` type as follow:

```go
schema.Schema{
    Fields: schema.Fields{
        "field": {
            Params: schema.Params{
                "width": {
                    Description: "Change the width of the thumbnail to the value in pixels",
                    Validator: schema.Integer{}
                },
                "height": {
                    Description: "Change the width of the thumbnail to the value in pixels",
                    Validator: schema.Integer{},
                },
            },
            Handler: func(ctx context.Context, value interface{}, params map[string]interface{}) (interface{}, error) {
                // your transformation logic here
                return value, nil
            },
        },
    },
}
```

Only parameters listed in the `Params` field will be accepted. You `Handler` function is called with the current value of the field and parameters sent by the user if any. Your function can apply wanted transformations on the value and return it. If an error is returned, a `422` error will be triggered with your error message associated to the field.

#### Embedding

With sub-fields notation you can also request referenced resources or connections (sub-resources). REST Layer will recognize them automatically and fetch the associated resources in order embed their data in the response. This can save a lot of unnecessary sequential round-trips:

```sh
$ http -b :8080/api/users/ar6eimekj5lfktka9mt0/posts \
  fields=='meta{title},user{id,name},comments(sort:"-created",limit:10){user{id,name},body}'
  [
    {
        "_etag": "ar6eimukj5lfl07r0uv0",
        "meta": {
            "title": "test"
        },
        "user": {
            "id": "ar6eimul07lfae7r4b5l",
            "name": "John Doe"
        },
        "comments": [
            {
                "user": {
                    "id": "ar6emul0kj5lfae7reimu",
                    "name": "Paul Wolf"
                },
                "body": "That's awesome!"
            },
            ...
        ]
    },
    ...
]
```

In the above example, the `user` field is a reference on the `users` resource. REST Layer did fetch the user referenced by the post and embedded the requested sub-fields (`id` and `name`). Same for `comments`: `comments` is set as a sub-resource of the `posts` resource. With this syntax, it's easy to get the last 10 comments on the post in the same REST request. For each of those comment, we asked to embed the `user` field referenced resource with `id` and `name` fields again.

Notice the `sort` and `limit` parameters passed to the `comments` field. Those are field parameter automatically exposed by connections to let you control the embedded list order, filter and pagination. You can use `sort`, `filter`, `skip`, `page` and `limit` parameters with those field with the same syntax as their top level query-string parameter counterpart.

Such request can quickly generate a lot of queries on the storage handler. To ensure a fast response time, REST layer tries to coalesce those storage requests and to execute them concurrently whenever possible.

### Pagination

Pagination is supported on collection URLs using the `page` and `limit` query-string parameters and can be used for resource list view URLs with request method `GET` and `DELETE`. If you don't define a default pagination limit using `PaginationDefaultLimit` resource configuration parameter, the resource won't be paginated for list `GET` requests until you provide the `limit` query-string parameter. The `PaginationDefaultLimit` does not apply to list `DELETE` requests, but the `limit` and `page` parameters may still be used to delete a subset of items.

If your collections are large enough, failing to define a reasonable `PaginationDefaultLimit` parameter may quickly render your API unusable.

### Skipping

Skipping of resource items is defined through the `skip` query-string parameter. The `skip` value is a positive integer defining the number of items to skip when querying for items, and can be applied for requests with method `GET` or `DELETE`.

Skip the first 10 items of the result:

```txt
/posts?skip=10
```

Return the first 2 items after skipping the first 10 of the result:

```txt
/posts?skip=10&limit=2
```

The `skip` parameter can be used in conjunction with the `page` parameter. You may want them both when for instance, you show the first N elements of a list and then allow to paginate the remaining items:

Show the first 2 elements:

```txt
/posts?limit=2
```

Paginate the rest of the list:

```txt
/posts?skip=2&page=1&limit=10
```

## Authentication and Authorization

REST Layer doesn't provide any kind of support for authentication. Identifying the user is out of the scope of a REST API, it should be performed by an OAuth server. The OAuth endpoints could be either hosted on the same code base as your API or live in a different app. The recommended way to integrate OAuth or any other kind of authentication with REST Layer is through a signed token like [JWT](https://jwt.io).

In this schema, the authentication service identifies the user and stores data relevant to the user's identification in a JWT token. This token is sent to the API client as a [bearer token](https://tools.ietf.org/html/rfc6750), through the `access-token` query-string parameter or the `Authorization` HTTP header. A http middleware then decodes and verifies this token, extracts user's info from it and stores it into the context. In REST layer, user info is now accessible from your [resource hooks](#hooks) so you can change the query lookup or ensure mutated objects are owned by the user in order to handle the authorization part.

See the [JWT auth example](https://github.com/clarify/rested/blob/master/examples/auth-jwt/main.go) for more info.

## Conditional Requests

Each stored resource provides information on the last time it was updated (`Last-Modified`), along with a hash value computed on the representation itself (`ETag`). These headers allow clients to perform conditional requests by using the `If-Modified-Since` header:

```sh
$ http :8080/users/ar6ej4mkj5lfl688d8lg If-Modified-Since:'Wed, 05 Dec 2012 09:53:07 GMT'
HTTP/1.1 304 Not Modified
```

or the `If-None-Match` header:

```sh
$ http :8080/users/ar6ej4mkj5lfl688d8lg If-None-Match:'"1234567890123456789012345678901234567890"'
HTTP/1.1 304 Not Modified
```

## Data Integrity and Concurrency Control

API responses include a `ETag` header which also allows for proper concurrency control. An `ETag` is a hash value representing the current state of the resource on the server. Clients may choose to ensure they update (`PATCH` or `PUT`) or delete (`DELETE`) a resource in the state they know it by providing the last known `ETag` for that resource. This prevents overwriting items with obsolete data.

Consider the following workflow:

```sh
$ http PATCH :8080/users/ar6ej4mkj5lfl688d8lg If-Match:'"1234567890123456789012345678901234567890"' \
    name='John Doe'
HTTP/1.1 412 Precondition Failed
```

What went wrong? We provided a `If-Match` header with the last known `ETag`, but its value did not match the current `ETag` of the item currently stored on the server, so we got a 412 Precondition Failed.

When this happens, it's up to the client to decide whether to inform the user of the error and/or re-fetch the latest version of the document to get the latest `ETag` before retrying the operation.

```sh
$ http PATCH :8080/users/ar6ej4mkj5lfl688d8lg If-Match:'"80b81f314712932a4d4ea75ab0b76a4eea613012"' \
    name='John Doe'
HTTP/1.1 200 OK
Etag: "7bb7a71b0f66197aa07c4c8fc9564616"
Last-Modified: Mon, 27 Jul 2015 19:36:19 GMT
```

This time the update operation was accepted and we got a new `ETag` for the updated resource.

Concurrency control header `If-Match` can be used with all mutation methods on item URLs: `PATCH` (update), `PUT` (replace) and `DELETE` (delete).

## Data Validation

Data validation is provided out-of-the-box. Your configuration includes a schema definition for every resource managed by the API. Data sent to the API to be inserted/updated will be validated against the schema, and a resource will only be updated if validation passes. See [Field Definition](#field-definition) section to know more about how to configure your validators.

```sh
$ http  :8080/api/users name:=1 foo=bar
HTTP/1.1 422 status code 422
Content-Length: 110
Content-Type: application/json
Date: Thu, 30 Jul 2015 21:56:39 GMT
Vary: Origin

{
    "code": 422,
    "message": "Document contains error(s)",
    "issues": {
        "foo": [
            "invalid field"
        ],
        "name": [
            "not a string"
        ]
    }
}
```

In the example above, the document did not validate so the request was rejected with description of the errors for each fields.

### Nullable Values

To allow `null` value in addition the field type, you can use [schema.AnyOf](https://pkg.go.dev/github.com/clarify/rested/schema#AnyOf) validator:

```go
"nullable_field": {
    Validator: schema.AnyOf{
        schema.String{},
        schema.Null{},
    },
}
```

### Extensible Data Validation

It is very easy to add new validators. You just need to implement the [schema.FieldValidator](https://pkg.go.dev/github.com/clarify/rested/schema#FieldValidator):

```go
type FieldValidator interface {
    Validate(value interface{}) (interface{}, error)
}
```

The `Validate` method takes the value as argument and must either return the value back with some eventual transformation or an `error` if the validation failed.

Your validator may also implement the optional [schema.Compiler](https://pkg.go.dev/github.com/clarify/rested/schema#Compiler) interface:

```go
type Compiler interface {
    Compile() error
}
```

When a field validator implements this interface, the `Compile` method is called at the server initialization. It's a good place to pre-compute some data (i.e.: compile regexp) and verify validator configuration. If validator configuration contains issues, the `Compile` method must return an error, so the initialization of the resource will generate a fatal error.

A validator may implement some advanced serialization or transformation of the data to optimize its storage. In order to read this data back and put it in a format suitable for JSON representation, a validator can implement the [schema.FieldSerializer](https://pkg.go.dev/github.com/clarify/rested/schema#FieldSerializer) interface:

```go
type FieldSerializer interface {
    Serialize(value interface{}) (interface{}, error)
}
```

When a validator implements this interface, the method is called with the field's value just before JSON marshaling. You should return an error if the format stored in the db is invalid and can't be converted back into a suitable representation.

See [schema.IP](https://pkg.go.dev/github.com/clarify/rested/schema#IP) validator for an implementation example.

## Timeout and Request Cancellation

REST Layer respects [context](https://pkg.go.dev/context) deadline from end to end. Timeout and request cancellation are thus handled through `context`. Since Go 1.8, context is cancelled automatically if the user closes the connection.

When a request is stopped because the client closed the connection (context cancelled), the response HTTP status is set to `499 Client Closed Request` (for logging purpose). When a timeout is set and the request has reached this timeout, the response HTTP status is set to `509 Gateway Timeout`.

## Logging

You can customize REST Layer logger by changing the `resource.Logger` function to call any logging framework you want.

We recommend using [zerolog](https://github.com/rs/zerolog). To configure REST Layer with `zerolog`, proceed as follow:

```go
// Init an alice handler chain (use your preferred one)
c := alice.New()

// Install a logger
c = c.Append(hlog.NewHandler(log.With().Logger()))

// Log API accesses
c = c.Append(hlog.AccessHandler(func(r *http.Request, status, size int, duration time.Duration) {
    hlog.FromRequest(r).Info().
        Str("method", r.Method).
        Str("url", r.URL.String()).
        Int("status", status).
        Int("size", size).
        Dur("duration", duration).
        Msg("")
}))

// Add some fields to per-request logger context
c = c.Append(hlog.RequestHandler("req"))
c = c.Append(hlog.RemoteAddrHandler("ip"))
c = c.Append(hlog.UserAgentHandler("ua"))
c = c.Append(hlog.RefererHandler("ref"))
c = c.Append(hlog.RequestIDHandler("req_id", "Request-Id"))

// Install zerolog/rest-layer adapter
resource.LoggerLevel = resource.LogLevelDebug
resource.Logger = func(ctx context.Context, level resource.LogLevel, msg string, fields map[string]interface{}) {
    zerolog.Ctx(ctx).WithLevel(zerolog.Level(level)).Fields(fields).Msg(msg)
}

```

See [zerolog](https://github.com/rs/zerolog) documentation for more info.

## CORS

REST Layer doesn't support CORS internally but relies on an external middleware to do so. You may use the [CORS](http://github.com/rs/cors) middleware to add CORS support to REST Layer if needed. Here is a basic example:

```go
package main

import (
    "log"
    "net/http"

    "github.com/rs/cors"
    "github.com/clarify/rested/resource"
    "github.com/clarify/rested/rest"
)

func main() {
    index := resource.NewIndex()

    // configure your resources

    api, err := rest.NewHandler(index)
    if err != nil {
        log.Fatalf("Invalid API configuration: %s", err)
    }

    handler := cors.Default().Handler(api)
    log.Fatal(http.ListenAndServe(":8080", handler))
}
```

## JSONP

In general you don’t really want to add JSONP when you can use CORS instead:

> There have been some criticisms raised about JSONP. Cross-origin resource sharing (CORS) is a more recent method of getting data from a server in a different domain, which addresses some of those criticisms. All modern browsers now support CORS making it a viable cross-browser alternative (source.)
> There are circumstances however when you do need JSONP, like when you have to support legacy software (IE6 anyone?)

As for CORS, REST Layer doesn't support JSONP directly but rely on an external middleware. Such a middleware is very easy to write. Here is an example:

```go
package main

import (
    "log"
    "net/http"

    "github.com/clarify/rested/resource"
    "github.com/clarify/rested/rest"
)

func main() {
    index := resource.NewIndex()

    // configure your resources

    api, err := rest.NewHandler(index)
    if err != nil {
        log.Fatalf("Invalid API configuration: %s", err)
    }

    handler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        fn := r.URL.Query().Get("callback")
        if fn != "" {
            w.Header().Set("Content-Type", "application/javascript")
        _, _ = w.Write([]byte(";fn("))
        }
        api.ServeHTTP(w, r)
        if fn != "" {
            _, _ = w.Write([]byte(");"))
        }
    })
    log.Fatal(http.ListenAndServe(":8080", handler))
}
```

## Data Storage Handler

REST Layer doesn't handle storage of resources directly. A [mem.MemoryHandler](https://pkg.go.dev/github.com/clarify/rested/resource/testing/mem#MemoryHandler) is provided as an example but should be used for testing only.

A resource storage handler is easy to write though. Some handlers for [popular databases are available](#main-storage-handlers), but you may want to write your own to put an API in front of anything you want. It is very easy to write a data storage handler, you just need to implement the [resource.Storer](https://pkg.go.dev/github.com/clarify/rested/resource#Storer) interface:

```go
type Storer interface {
    Find(ctx context.Context, q *query.Query) (*ItemList, error)
    Insert(ctx context.Context, items []*Item) error
    Update(ctx context.Context, item *Item, original *Item) error
    Delete(ctx context.Context, item *Item) error
    Clear(ctx context.Context, q *query.Query) (int, error)
}
```

Mutation methods like `Update` and `Delete` must ensure they are atomically mutating the same item as specified in argument by checking their `ETag` (the stored `ETag` must match the `ETag` of the provided item). In case the handler can't guarantee that, the storage must be left untouched and a [resource.ErrConflict](https://pkg.go.dev/github.com/clarify/rested/resource#pkg-variables) must be returned.

If the operation is not immediate, the method must listen for cancellation on the passed `ctx`. If the operation is stopped due to context cancellation, the function must return the result of the [ctx.Err()](https://pkg.go.dev/golang.org/x/net/context#Context) method. See [this blog post](https://blog.golang.org/context) for more information about how `context` works.

If the backend storage is able to efficiently fetch multiple document by their id, it can implement the optional [resource.MultiGetter](https://pkg.go.dev/github.com/clarify/rested/resource#MultiGetter) interface. REST Layer will automatically use it whenever possible.

See [resource.Storer](https://pkg.go.dev/github.com/clarify/rested/resource#Storer) documentation for more information on resource storage handler implementation details.

## Custom Response Formatter / Sender

REST Layer lets you extend or replace the default response formatter and sender. To write a new response format, you need to implement the [rest.ResponseFormatter](https://pkg.go.dev/github.com/clarify/rested/rest#ResponseFormatter) interface:

```go
// ResponseFormatter defines an interface responsible for formatting a the different types of response objects
type ResponseFormatter interface {
    // FormatItem formats a single item in a format ready to be serialized by the ResponseSender
    FormatItem(ctx context.Context, headers http.Header, i *resource.Item, skipBody bool) (context.Context, interface{})
    // FormatList formats a list of items in a format ready to be serialized by the ResponseSender
    FormatList(ctx context.Context, headers http.Header, l *resource.ItemList, skipBody bool) (context.Context, interface{})
    // FormatError formats a REST formated error or a simple error in a format ready to be serialized by the ResponseSender
    FormatError(ctx context.Context, headers http.Header, err error, skipBody bool) (context.Context, interface{})
}
```

You can also customize the response sender responsible for the serialization of the formatted payload:

```go
// ResponseSender defines an interface responsible for serializing and sending the response
// to the http.ResponseWriter.
type ResponseSender interface {
    // Send serialize the body, sets the given headers and write everything to the provided response writer
    Send(ctx context.Context, w http.ResponseWriter, status int, headers http.Header, body interface{})
}
```

Then set your response formatter and sender on the REST Layer HTTP handler like this:

```go
api, _ := rest.NewHandler(index)
api.ResponseFormatter = &myResponseFormatter{}
api.ResponseSender = &myResponseSender{}
```

You may also extend the [DefaultResponseFormatter](https://pkg.go.dev/github.com/clarify/rested/rest#DefaultResponseFormatter) and/or [DefaultResponseSender](https://pkg.go.dev/github.com/clarify/rested/rest#DefaultResponseSender) if you just want to wrap or slightly modify the default behavior:

```go
type myResponseFormatter struct {
    rest.DefaultResponseFormatter
}

// Add a wrapper around the list with pagination info
func (r myResponseFormatter) FormatList(ctx context.Context, headers http.Header, l *resource.ItemList, skipBody bool) (context.Context, interface{}) {
    ctx, data := r.DefaultResponseFormatter.FormatList(ctx, headers, l, skipBody)
    return ctx, map[string]interface{}{
        "meta": map[string]int{
            "offset": l.Offset,
            "total":  l.Total,
        },
        "list": data,
    }
}
```

## JSONSchema

It is possible to convert a schema to [JSON Schema](http://json-schema.org/) with some limitations for certain schema fields. Currently, we implement JSON Schema Draft 4 [core](https://tools.ietf.org/html/draft-zyp-json-schema-04) and [validation](https://tools.ietf.org/html/draft-fge-json-schema-validation-00) specifications. In addition, we have implemented "readOnly" from the less commonly used [hyper-schema](https://tools.ietf.org/html/draft-luff-json-hyper-schema-00#section-4.4) specification.

Example usage:

```go
import "github.com/clarify/rested/schema/encoding/jsonschema"

b := new(bytes.Buffer)
enc := jsonschema.NewEncoder(b)
if err := enc.Encode(aSchema); err != nil {
  return err
}
fmt.Println(b.String()) // Valid JSON Document describing the schema.
```

### Custom FieldValidators

For a custom `FieldValidator` to support encoding to JSON Schema, it must implement the `jsonschema.Builder` interface:

```go
// The Builder interface should be implemented by custom schema.FieldValidator implementations to allow JSON Schema
// serialization.
type Builder interface {
    // BuildJSONSchema should return a map containing JSON Schema Draft 4 properties that can be set based on
    // FieldValidator data. Application specific properties can be added as well, but should not conflict with any
    // legal JSON Schema keys.
    BuildJSONSchema() (map[string]interface{}, error)
}
```

To easier extend a `FieldValidator` from the `schema` package, you can call `ValidatorBuilder` inside `BuildJSONSchema()`:

```go
type Email struct {
    schema.String
}

func (e Email) BuildJSONSchema() (map[string]interface{}, error) {
    parentBuilder, _ = jsonschema.ValidatorBuilder(e.String)
    m, err := parentBuilder.BuildJSONSchema()
    if err != nil {
        return nil, err
    }
    m["format"] = "email"
    return m, nil
}
```

### Sub-schema Limitation

Sub-schemas only get converted to JSON Schema, if you specify a sub-schema via setting a Field's `Validator` attribute to a `schema.Object` instance. Use of the Field's `Schema` field is _not_ supported.

### schema.Dict Limitations

`schema.Dict` only supports `nil` and `schema.String` as `KeysValidator` values. Note that some less common combinations of `schema.String` attributes will lead to usage of an `allOf` construct with duplicated schemas for values. This is to avoid usage of regular expression expansions that only a subset of implementations actually support.

The limitation in `KeysValidator` values arise because JSON Schema draft 4 (and draft 5) support for key validation is limited to [properties, patternProperties and additionalProperties](https://tools.ietf.org/html/draft-fge-json-schema-validation-00#section-5.4.4). This essentially means that there can be no JSON Schema object supplied for key validation, but that we need to rely on exact match (properties), regular expressions (patternProperties) or no key validation (additionalProperties).

### schema.Reference Provisional Support

The support for `schema.Reference` is purely provisional and simply returns an empty object `{}`, meaning it does not give any hint as to which validation the server might use.

With a potential later implantation of the [OpenAPI Specification](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/2.0.md) (a.k.a. the Swagger 2.0 Specification), the goal is to refer to the ID field of the linked resource via an object `{"$ref": "#/definitions/<unique schema title>/id"}`. This is tracked via issue [#36](https://github.com/clarify/rested/issues/36).

### schema.URL Limitations

The current serialization of `schema.URL` always returns a schema `{"type": "string", "format": "uri"}`, ignoring any struct attributes that affect the actual validation within rest-layer. The JSON Schema is thus not completely accurate for this validator.

Note that JSON Schema draft 5 adds [uriref](https://tools.ietf.org/html/draft-wright-json-schema-validation-00#section-7.3.7), which could allow us to at least document whether `AllowRelative` is `true` or `false`. JSON Schema also allow application specific additional formats to be defined, but it's not practical to create a custom format for any possible struct attribute combination.

## Licenses

All source code is licensed under the [MIT License](https://raw.github.com/clarify/rested/master/LICENSE).
