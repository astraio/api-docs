Astra API Docs (v0.2)
=====================

[Astra](https://astra.io/) is a modern cloud platform for storing, transforming, and delivering media. The system includes:

* key-value storage for static files, with type info (image, generic blob, etc.);
* type-specific transforms (e.g., image resizing, video cutting), which can be applied on the fly;
* content delivery via direct link or built-in CDN; and
* flat rates for disk space and bandwidth (no tiers or additional API fees).

The infrastructure behind Astra has powered [PhotoShelter](http://www.photoshelter.com/) for the last decade. Our distributed, redundant storage system currently handles billions of objects, comprising several PBs of data.

#### Contact

* For more info, including rates, check out: https://astra.io/
* To sign up for a **60-day free trial**, visit: https://astra.io/trial/
* For support, feature requests, or bugs, e-mail: [hello@astra.io](mailto:hello@astra.io)

#### Future Work

While tested and stable, this version of Astra is a **preview release** which we'll be actively expanding.

We have plans for implementing usage endpoints, cache rulesets, auth tokens, query ranges/paging, and more. If there's an object type, transform, or other feature you need, please [contact us](mailto:hello@astra.io).

#### Contents

* [API Overview](#api-overview)
    * [Versioning](#versioning)
* [Private Requests](#private-requests)
    * [Base URL](#base-url)
    * [Secret-Based Authentication](#secret-based-authentication)
    * [Responses](#responses)
* [Public Requests](#public-requests)
    * [Protocol](#protocol)
    * [Requests via CDN](#requests-via-cdn)
    * [HMAC Signatures](#hmac-signatures)
* [Entities](#entities)
    * [Buckets](#buckets)
    * [Objects](#objects)
* [Endpoints](#endpoints)
    * [`root`](#root)
    * [`bucket`](#bucket)
    * [`object`](#object)
    * [`stream`](#stream)
    * [`public`](#public)
* [Transforms](#transforms)
    * [`blob` Object](#blob-object)
    * [`image` Object](#image-object)
    * [`video` Object](#video-object)
* [Errors](#errors)
    * [`ProtocolErr`](#protocolerr)
    * [`AuthErr`](#autherr)
    * [`FormErr`](#formerr)
    * [`BucketErr`](#bucketerr)
    * [`ObjectErr`](#objecterr)
    * [`InternalErr`](#internalerr)
* [Changelog](#changelog)
    * [v0](#v0)

API Overview
------------

Astra organizes data using a bucket-object model. You can create unlimited buckets (which do not nest) in your account, provided they are uniquely named. Each bucket can contain any number of objects whose names are unique to it. Together, a bucket-object name tuple uniquely identifies a file.

Objects in Astra are typed, with a generic `blob` type available as a catch-all. Types are required and determine what metadata the API computes for objects, as well as how they can be transformed.

The RESTful API supports standard CRUD ops, authentication via shared secret or HMAC signature, and serves data directly or via CDN. Except when streaming, the API returns JSON exclusively.

We charge for storage and bandwidth only; API calls are unlimited, except to guard against abuse.

### Versioning

API releases are indicated by major and minor version numbers. Specifying only the major version number always calls the most recent minor release. So right now `v0` points to `v0.2`.

Your client can check the API version programmatically by calling the [`root`](#root) endpoint.

Minor version updates may add fixes or small features but will always be backward-compatible. Any changes that break compatibility will be released as new major versions, on an opt-in basis.

Private Requests
----------------

The simplest API requests you can make are private. Private requests use:

* [HTTPS](https://en.wikipedia.org/wiki/HTTP_Secure); and
* authentication via [shared secrets](https://en.wikipedia.org/wiki/Shared_secret) ([pre-shared keys](https://en.wikipedia.org/wiki/Pre-shared_key)).

Private requests are most appropriate for server-side API clients. You should **not** make private requests that would expose your account's secrets (over plain HTTP, in client-side code, and so on).

### Base URL

The base URL for private requests, which are served directly, is currently:

```bash
https://api.astra.io/v0/
```

(You can substitute `v0.2` for `v0` to target the minor version specifically.)

### Secret-Based Authentication

Shared secrets are 32-character, [base64url](https://tools.ietf.org/html/rfc4648#page-7)-encoded (i.e., from the set `[A-Za-z0-9\-_]`) strings. The API authenticates private requests via secrets passed in the `Astra-Secret` HTTP request header:

```bash
$ curl https://api.astra.io/v0/... \
    -H 'Astra-Secret: 3jaX4Bls9rxCiqSYfv5FaRMbfqff2Vh7' \
    ...
```

Future versions of the API may allow you to authenticate requests using tokens.

You can maintain your account's secrets by logging into the [control panel](https://astra.io/panel/). In addition to adding and deleting secrets, you can also toggle their active statuses in order to disable them temporarily.

### Responses

Except when streaming binary data, the Astra API returns JSON exclusively. All JSON responses, regardless of the resource, contain an `ok` boolean to say whether the request succeeded.

Requests for single resources return JSON objects under the `data` field:

```json
{
    "ok": true,
    "data": { ... }
}
```

When the API returns multiple resources, `data` is an array of objects:

```json
{
    "ok": true,
    "data": [
        ...,
        { ... },
        ...
    ]
}
```

In both cases, the API will also set the response's HTTP status code to `200 OK`.

If any request fails, the `ok` field will be `false`, and the API will return an `error` object instead of `data`. Clients should always check the `ok` field first when handling responses.

Public Requests
---------------

(For specific examples of signing URLs, see the section on [`public`](#public) endpoint requests.)

In addition to private requests, the API also supports public requests. These requests are:

* aliases for certain [`object`](#object) and [`stream`](#stream) endpoints;
* served over vanilla HTTP (versus HTTPS);
* authenticated by calculating and appending [HMAC](https://en.wikipedia.org/wiki/Hash-based_message_authentication_code) signatures;
* often temporary, with lifetimes enforced using `expires` parameters; and
* optionally delivered via built-in CDN.

Public requests are appropriate for client-side use: serving images on web pages, handling user uploads, etc. In these cases, you would build URLs server-side and then pass them to the front end.

Authentication using one-time tokens may be supported in future API versions.

### Protocol

When using the [`public`](#public) endpoint, change the standard [base URL](#base-url)'s protocol to plain `http`:

```bash
http://api.astra.io/v0/public/...
```

### Requests via CDN

To serve public requests via CDN, you can substitute `cdn` for `api` in the subdomain:

```bash
http://cdn.astra.io/v0/public/...
```

The API supports delivery via CDN for public streaming only; metadata must be accessed directly.

### HMAC Signatures

HMAC signatures are used to verify the sources and integrity of public requests. Since changing URLs invalidates their signatures, signing "locks in" parameters like `expires` fields and [transforms](#transforms).

The API's method for constructing [HMAC-SHA1](http://tools.ietf.org/html/rfc2104) signatures is (in PHP-style pseudocode):

```php
function sign_request($http_method, $path, $percent_encoded_query_str=null, $secret)
{
    if (isset($percent_encoded_query_str)) {
        $sorted_qs = sort_by_field_name($percent_encoded_query_str); // lexicographical order
    }

    $str = $http_method . ':' . $path . (empty($sorted_qs) ? '' : ('?' . $sorted_qs));
    $sha1 = hash_hmac('sha1', $str, $secret, true); // as raw binary data

    return rtrim(strtr(base64_encode($sha1), '+/', '-_'), '='); // unpadded base64url sig
}
```

You can then pass signatures with requests by adding `hmac={signature}` to their query strings.

Entities
--------

Buckets and objects are the two entities you'll manipulate using the API's current endpoints. The list of entities will likely grow as we add support for rulesets, account configuration endpoints, etc.

Entities have two standard JSON representations, called long and short forms:

* Long forms list all attributes and are usually returned when interacting with a single entity.
* Short forms are truncated versions, which are more appropriate for nesting (e.g., listing objects within buckets) or inclusion in lists of multiple entities, like query results.

A few things to keep in mind when dealing with entity fields:

* The API serves and accepts timestamp strings in [RFC 3339](http://www.ietf.org/rfc/rfc3339.txt) format only. These timestamps must be UTC-offset, with second-level precision (e.g., `2014-08-30T14:28:56Z`).
* All integers are potentially 64-bit, which may cause problems for JavaScript clients.
* Buckets and objects both have `status` fields which are always `ready`. We may create values in later API versions to handle async processing, so filtering for `ready` is smart future-proofing.

### Buckets

Buckets are containers for objects. They cannot nest, and must be uniquely named within your account. Deleting buckets from Astra also deletes all the objects they contain.

#### Long Form

| Field     | Type      | Description                                                   |
|:----------|:----------|:--------------------------------------------------------------|
| `name`    | string    | 1-256 chars from `[A-Za-z0-9.\-_]` (can't begin/end with `.`) |
| `size`    | integer   | total of all objects (in bytes)                               |
| `status`  | string    | `ready` (reserved for future use)                             |
| `objects` | array     | list of short-form objects                                    |
| `ctime`   | timestamp | creation time                                                 |
| `mtime`   | timestamp | modification time (doesn't inherit from objects)              |

```json
{
    "name": "someBucket",
    "size": 437716,
    "status": "ready",
    "objects": [
        {"name": "fooObject.gif"},
        {"name": "barObject.js"},
        {"name": "bazObject.pdf"}
    ],
    "ctime": "2014-08-30T14:28:56Z",
    "mtime": "2014-10-07T18:35:19Z"
}
```

#### Short Form

Short-form buckets replace the `objects` array with an integer count:

```json
{
    "name": "someBucket",
    "size": 437716,
    "status": "ready",
    "objects": 3,
    "ctime": "2014-08-30T14:28:56Z",
    "mtime": "2014-10-07T18:35:19Z"
}
```

### Objects

Objects sit inside buckets and represent data you've uploaded, plus metadata the API has parsed. Types determine objects' metadata and [transforms](#transforms). A generic `blob` type can be used as a catch-all.

You can identify objects in your account uniquely using bucket-object name tuples.

#### Long Form

| Field    | Type      | Description                                                      |
|:---------|:----------|:-----------------------------------------------------------------|
| `name`   | string    | 1-2048 chars from `[A-Za-z0-9.\-_]` (can't begin/end with `.`)   |
| `bucket` | string    | name of parent bucket                                            |
| `hash`   | string    | [SHA-1](https://en.wikipedia.org/wiki/SHA-1) hash (40 hex chars) |
| `size`   | integer   | size (in bytes)                                                  |
| `type`   | string    | currently `blob`, `image`, or `video`                            |
|  *...*   | *...*     | *type-specific fields*                                           |
| `status` | string    | `ready` (reserved for future use)                                |
| `ctime`  | timestamp | creation time                                                    |
| `mtime`  | timestamp | modification time                                                |

##### `blob` Objects

Type-specific fields:

| Field     | Type   | Description                                                      |
|:----------|:-------|:-----------------------------------------------------------------|
| `content` | string | `Content-Type` (`[A-Za-z0-9\-]+\/[A-Za-z0-9\-]+`) (may be empty) |

```json
{
    "name": "blobObject.pdf",
    "bucket": "someBucket",
    "hash": "1a5699e83a1d4aaf7a23ddb5e9f0a8979d0007d8",
    "size": 9381224,
    "type": "blob",
    "status": "ready",
    "content": "application/pdf",
    "ctime": "2014-07-28T14:42:27Z",
    "mtime": "2014-07-28T14:42:27Z"
}
```

##### `image` Objects

Type-specific fields:

| Field    | Type    | Description                       |
|:---------|:--------|:----------------------------------|
| `format` | string  | currently `gif`, `jpeg`, or `png` |
| `width`  | integer | image width (in pixels)           |
| `height` | integer | image height (in pixels)          |

```json
{
    "name": "imageObject.jpg",
    "bucket": "someBucket",
    "hash": "dc53baf6000a0b7d3ef55afe86a7f48fc16c5373",
    "size": 1845778,
    "type": "image",
    "status": "ready",
    "format": "jpeg",
    "width": 3264,
    "height": 2448,
    "ctime": "2014-06-03T10:56:08Z",
    "mtime": "2014-06-14T02:17:13Z"
}
```

##### `video` Objects

Type-specific fields:

| Field      | Type    | Description                       |
|:-----------|:--------|:----------------------------------|
| `format`   | string  | currently `mp4`, `ogg`, or `webm` |
| `width`    | integer | video width (in pixels)           |
| `height`   | integer | video height (in pixels)          |
| `start`    | float   | video start (in seconds)          |
| `duration` | float   | video duration (in seconds)       |

```json
{
    "name": "videoObject.mp4",
    "bucket": "someBucket",
    "hash": "06e786af81f20f7d31ba34fb0a5e92b0e7c4ff19",
    "size": 99476300,
    "type": "video",
    "status": "ready",
    "format": "mp4",
    "width": 1280,
    "height": 720,
    "start": 0,
    "duration": 112.831667,
    "ctime": "2014-11-16T20:22:58Z",
    "mtime": "2014-11-16T20:22:58Z"
}
```

#### Short Form

Regardless of type, short-form objects consist of a `name` field only:

```json
{"name": "blobObject.pdf"}
```

```json
{"name": "imageObject.jpg"}
```

```json
{"name": "videoObject.mp4"}
```

Endpoints
---------

The Astra API uses a [RESTful](https://en.wikipedia.org/wiki/Representational_state_transfer) structure, organizing [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete) operations on entities around [HTTP methods](http://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html). (Absent a true `PATCH` verb, the API uses `POST` for both creating and updating entities.)

You can pass required or optional parameters as multipart form data (`multipart/form-data`). Query strings are used for setting request-specific info, like transform details or [HMAC](https://en.wikipedia.org/wiki/Hash-based_message_authentication_code) signatures.

Currently, the API supports only ASCII values in form data, and requires timestamps to be in [RFC 3339](http://www.ietf.org/rfc/rfc3339.txt) format (relative to UTC) with second-level precision (e.g., `2014-08-30T14:28:56Z`). 

Any values you pass to the API in query string fields should be [percent-encoded](http://tools.ietf.org/html/rfc3986#section-2.1).

### `root`

The `root` pseudo-endpoint is located at the API base and returns info about the API itself. Right now, `root` indicates only the version of the API you're accessing. We may expand this in the future.

#### Scheme

| Action             | Method | Path |
|:-------------------|:-------|:-----|
| [Read](#root-read) | `GET`  | `/`  |

#### `root` Read

* Method: `GET`
* Path: `/`

**Example**

Fetch the API version programmatically, which is useful if you're specifying only major revisions:

```bash
$ curl https://api.astra.io/v0/ \
    -H 'Astra-Secret: 3jaX4Bls9rxCiqSYfv5FaRMbfqff2Vh7' \
    -X GET
```

=>

```json
{
    "ok": true,
    "data": {
        "version": {
            "string": "0.2",
            "major": 0,
            "minor": 2
        }
    }
}
```

### `bucket`

The `bucket` endpoint lets you maintain buckets. Note that deleting buckets deletes their contents.

#### Scheme

| Action                   | Method   | Path                 |
|:-------------------------|:---------|:---------------------|
| [Create](#bucket-create) | `POST`   | `/bucket`            |
| [Query](#bucket-query)   | `GET`    | `/bucket`            |
| [Read](#bucket-read)     | `GET`    | `/bucket/{bkt_name}` |
| [Update](#bucket-update) | `POST`   | `/bucket/{bkt_name}` |
| [Delete](#bucket-delete) | `DELETE` | `/bucket/{bkt_name}` |

#### `bucket` Create

* Method: `POST`
* Path: `/bucket`
* Form fields:
    * `name` - bucket name

**Example**

Make a new bucket called `fooBucket`:

```bash
$ curl https://api.astra.io/v0/bucket \
    -H 'Astra-Secret: 3jaX4Bls9rxCiqSYfv5FaRMbfqff2Vh7' \
    -X POST \
    -F name=fooBucket
```

=>

```json
{
    "ok": true,
    "data": {
        "name": "fooBucket",
        "size": 0,
        "status": "ready",
        "objects": [],
        "ctime": "2014-09-02T10:11:34Z",
        "mtime": "2014-09-02T10:11:34Z"
    }
}
```

#### `bucket` Query

* Method: `GET`
* Path: `/bucket`

**Example**

List all the buckets in your account:

```bash
$ curl https://api.astra.io/v0/bucket \
    -H 'Astra-Secret: 3jaX4Bls9rxCiqSYfv5FaRMbfqff2Vh7' \
    -X GET
```

=>

```json
{
    "ok": true,
    "data": [
        ...,
        {
            "name": "uploads",
            "size": 479236,
            "status": "ready",
            "objects": 2,
            "ctime": "2014-08-27T16:22:32Z",
            "mtime": "2014-09-01T05:36:45Z"
        },
        {
            "name": "fooBucket",
            "size": 0,
            "status": "ready",
            "objects": 0,
            "ctime": "2014-09-02T10:11:34Z",
            "mtime": "2014-09-02T10:11:34Z"
        },
        ...
    ]
}
```

#### `bucket` Read

* Method: `GET`
* Path: `/bucket/{bkt_name}`

**Example**

Fetch a bucket named `uploads`:

```bash
$ curl https://api.astra.io/v0/bucket/uploads \
    -H 'Astra-Secret: 3jaX4Bls9rxCiqSYfv5FaRMbfqff2Vh7' \
    -X GET
```

=>

```json
{
    "ok": true,
    "data": {
        "name": "uploads",
        "size": 1479236,
        "status": "ready",
        "objects": [
            {"name": "upload-0.jpg"},
            {"name": "upload-1.png"}
        ],
        "ctime": "2014-08-27T16:22:32Z",
        "mtime": "2014-09-01T05:36:45Z"
    }
}
```

#### `bucket` Update

* Method: `POST`
* Path: `/bucket/{bkt_name}`
* Form fields:
    * [`name`] - new bucket name (optional)

**Example**

Rename `fooBucket` to `barBucket`:

```bash
$ curl https://api.astra.io/v0/bucket/fooBucket \
    -H 'Astra-Secret: 3jaX4Bls9rxCiqSYfv5FaRMbfqff2Vh7' \
    -X POST \
    -F name=barBucket
```

=>

```json
{
    "ok": true,
    "data": {
        "name": "barBucket",
        "size": 0,
        "status": "ready",
        "objects": [],
        "ctime": "2014-09-02T10:11:34Z",
        "mtime": "2014-09-04T12:01:27Z"
    }
}
```

#### `bucket` Delete

* Method: `DELETE`
* Path: `/bucket/{bkt_name}`

**Example**

Remove `barBucket` (including any objects inside) from your account:

```bash
$ curl https://api.astra.io/v0/bucket/barBucket \
    -H 'Astra-Secret: 3jaX4Bls9rxCiqSYfv5FaRMbfqff2Vh7' \
    -X DELETE
```

=>

```json
{"ok": true}
```

### `object`

The `object` endpoint lets you perform CRUD operations on objects. The parameters you pass to this endpoint may differ from object type to object type (when creating an object, for example).

Use this endpoint to act on objects as entities. If you need to stream them, use the [`stream`](#stream) endpoint.

#### Scheme

| Action                   | Method   | Path                                   |
|:-------------------------|:---------|:---------------------------------------|
| [Create](#object-create) | `POST`   | `/bucket/{bkt_name}/object`            |
| [Query](#object-query)   | `GET`    | `/bucket/{bkt_name}/object`            |
| [Read](#object-read)     | `GET`    | `/bucket/{bkt_name}/object/{obj_name}` |
| [Update](#object-update) | `POST`   | `/bucket/{bkt_name}/object/{obj_name}` |
| [Delete](#object-delete) | `DELETE` | `/bucket/{bkt_name}/object/{obj_name}` |

#### `object` Create

* Method: `POST`
* Path: `/bucket/{bkt_name}/object`
* Form fields:
    * `name` - object name
    * `type` - object type (currently `blob` or `image`)
    * `file` - binary data
    * if `blob`:
        * [`content`] - `Content-Type` for streaming (optional)

**Example 1**

Upload a new `blob` object named `static.js` to `testBucket`:

```bash
$ curl https://api.astra.io/v0/bucket/testBucket/object \
    -H 'Astra-Secret: 3jaX4Bls9rxCiqSYfv5FaRMbfqff2Vh7' \
    -X POST \
    -F name=static.js \
    -F type=blob \
    -F file=@/path/to/local/file/static-local.js \
    -F content=application/javascript
```

=>

```json
{
    "ok": true,
    "data": {
        "name": "static.js",
        "bucket": "testBucket",
        "hash": "1d0204069fa14db15adf1ac0e925ed166b9f2a51",
        "size": 83968,
        "type": "blob",
        "status": "ready",
        "content": "application/javascript",
        "ctime": "2014-06-03T18:11:51Z",
        "mtime": "2014-06-03T18:11:51Z"
    }
}
```

**Example 2**

Make a new `image` object called `hawaii.jpg` in `testBucket`:

```bash
$ curl https://api.astra.io/v0/bucket/testBucket/object \
    -H 'Astra-Secret: 3jaX4Bls9rxCiqSYfv5FaRMbfqff2Vh7' \
    -X POST \
    -F name=hawaii.jpg \
    -F type=image \
    -F file=@/path/to/local/file/hawaii-local.jpg
```

=>

```json
{
    "ok": true,
    "data": {
        "name": "hawaii.jpg",
        "bucket": "testBucket",
        "hash": "8b9f3c27fc466cf8a1c6627166b6227385e45886",
        "size": 686480,
        "type": "image",
        "status": "ready",
        "format": "jpeg",
        "width": 1600,
        "height": 1200,
        "ctime": "2014-06-05T07:42:24Z",
        "mtime": "2014-06-05T07:42:24Z"
    }
}
```

#### `object` Query

* Method: `GET`
* Path: `/bucket/{bkt_name}/object`

**Example**

List all the objects in `testBucket`:

```bash
$ curl https://api.astra.io/v0/bucket/testBucket/object \
    -H 'Astra-Secret: 3jaX4Bls9rxCiqSYfv5FaRMbfqff2Vh7' \
    -X GET
```

=>

```json
{
    "ok": true,
    "data": [
        ...,
        {
            "name": "static.js",
            "bucket": "testBucket",
            "hash": "1d0204069fa14db15adf1ac0e925ed166b9f2a51",
            "size": 83968,
            "type": "blob",
            "status": "ready",
            "content": "application/javascript",
            "ctime": "2014-06-03T18:11:51Z",
            "mtime": "2014-06-03T18:11:51Z"
        },
        {
            "name": "hawaii.jpg",
            "bucket": "testBucket",
            "hash": "8b9f3c27fc466cf8a1c6627166b6227385e45886",
            "size": 686480,
            "type": "image",
            "status": "ready",
            "format": "jpeg",
            "width": 1600,
            "height": 1200,
            "ctime": "2014-06-05T07:42:24Z",
            "mtime": "2014-06-05T07:42:24Z"
        },
        ...
    ]
}
```

#### `object` Read

* Method: `GET`
* Path: `/bucket/{bkt_name}/object/{obj_name}`

**Example**

Fetch an `image` object called `hawaii.jpg` from `testBucket`:

```bash
$ curl https://api.astra.io/v0/bucket/testBucket/object/hawaii.jpg \
    -H 'Astra-Secret: 3jaX4Bls9rxCiqSYfv5FaRMbfqff2Vh7' \
    -X GET
```

=>

```json
{
    "ok": true,
    "data": {
        "name": "hawaii.jpg",
        "bucket": "testBucket",
        "hash": "8b9f3c27fc466cf8a1c6627166b6227385e45886",
        "size": 686480,
        "type": "image",
        "status": "ready",
        "format": "jpeg",
        "width": 1600,
        "height": 1200,
        "ctime": "2014-06-05T07:42:24Z",
        "mtime": "2014-06-05T07:42:24Z"
    }
}
```

#### `object` Update

* Method: `POST`
* Path: `/bucket/{bkt_name}/object/{obj_name}`
* Form fields:
    * [`bucket`] - move object to new bucket (optional)
    * [`name`] - new object name (optional)
    * [`type`] - new object type (optional)
    * [`file`] - new binary data (required if new `type`; otherwise optional)
    * if `blob`:
        * [`content`] - new `Content-Type` for streaming (optional)

**Example 1**

Update a `blob` object named `static.js` in `testBucket`:

```bash
$ curl https://api.astra.io/v0/bucket/testBucket/object/static.js \
    -H 'Astra-Secret: 3jaX4Bls9rxCiqSYfv5FaRMbfqff2Vh7' \
    -X POST \
    -F name=static.min.js \
    -F file=@/path/to/local/file/static-local.min.js
```

=>

```json
{
    "ok": true,
    "data": {
        "name": "static.min.js",
        "bucket": "testBucket",
        "hash": "c79b0a1cbc9707b58a8a4a8638df4a32ca507087",
        "size": 48312,
        "type": "blob",
        "status": "ready",
        "content": "application/javascript",
        "ctime": "2014-06-03T18:11:51Z",
        "mtime": "2014-06-03T22:05:48Z"
    }
}
```

**Example 2**

Upload a new `image` to replace `hawaii.jpg` in `testBucket`:

```bash
$ curl https://api.astra.io/v0/bucket/testBucket/object/hawaii.jpg \
    -H 'Astra-Secret: 3jaX4Bls9rxCiqSYfv5FaRMbfqff2Vh7' \
    -X POST \
    -F file=@/path/to/local/file/kauai-local.jpg
```

=>

```json
{
    "ok": true,
    "data": {
        "name": "hawaii.jpg",
        "bucket": "testBucket",
        "hash": "7892bec20a397645bb6a5a6777933b2265303082",
        "size": 512191,
        "type": "image",
        "status": "ready",
        "format": "jpeg",
        "width": 1200,
        "height": 800,
        "ctime": "2014-06-05T07:42:24Z",
        "mtime": "2014-06-05T19:27:11Z"
    }
}
```

#### `object` Delete

* Method: `DELETE`
* Path: `/bucket/{bkt_name}/object/{obj_name}`

**Example**

Remove `blob` object `static.min.js` from `testBucket`:

```bash
$ curl https://api.astra.io/v0/bucket/testBucket/object/static.min.js \
    -H 'Astra-Secret: 3jaX4Bls9rxCiqSYfv5FaRMbfqff2Vh7' \
    -X DELETE
```

=>

```json
{"ok": true}
```

### `stream`

The `stream` endpoint delivers objects as binary data. To act on objects, use the [`object`](#object) endpoint.

You can apply [transforms](#transforms) when streaming by adding optional parameters via query string.

#### Scheme

| Action               | Method | Path                                   |
|:---------------------|:-------|:---------------------------------------|
| [Read](#stream-read) | `GET`  | `/bucket/{bkt_name}/stream/{obj_name}` |

#### `stream` Read

* Method: `GET`
* Path: `/bucket/{bkt_name}/stream/{obj_name}`
* Query string:
    * *type-specific transforms*

(Refer to the section on [transforms](#transforms) for examples of on-the-fly modifications.)

**Example**

Serve an image called `foo.png` directly, without applying any transforms:

```bash
$ curl 'https://api.astra.io/v0/bucket/content/stream/foo.png' \
    -H 'Astra-Secret: 3jaX4Bls9rxCiqSYfv5FaRMbfqff2Vh7' \
    -X GET
```

=> *image streams directly as `Content-Type: image/png`*

### `public`

The `public` endpoint aliases certain [`object`](#object) and [`stream`](#stream) endpoints to deliver objects publicly. You can use this endpoint to build self-contained, signed requests suitable for embedding, etc.

* Form fields:
    * *form fields are forwarded to aliased endpoints*
* Query string:
    * *query strings are forwarded to aliased endpoints*
    * `hmac` - HMAC-SHA1 signature
    * [`expires`] - percent-encoded [RFC 3339](http://www.ietf.org/rfc/rfc3339.txt) timestamp (optional)

(Refer to the section on [HMAC signatures](#hmac-signatures) to learn the algorithm for signing messages.)

#### Scheme

| Action                   | Method   | Path                                         |
|:-------------------------|:---------|:---------------------------------------------|
| [Create](#public-create) | `POST`   | `/public/{acct_label}/{bkt_name}`            |
| [Query](#public-query)   | `GET`    | `/public/{acct_label}/{bkt_name}`            |
| [Read](#public-read)     | `GET`    | `/public/{acct_label}/{bkt_name}/{obj_name}` |
| [Update](#public-update) | `POST`   | `/public/{acct_label}/{bkt_name}/{obj_name}` |
| [Delete](#public-delete) | `DELETE` | `/public/{acct_label}/{bkt_name}/{obj_name}` |

#### `public` Create

(This aliases the [`object` create](#object-create) endpoint.)

#### `public` Query

(This aliases the [`object` query](#object-query) endpoint.)

#### `public` Read

(This aliases the [`stream` read](#stream-read) endpoint; appending `?metadata=true` aliases [`object` read](#object-read). As an alias to the [`stream` read](#stream-read) endpoint, you can sign in on-the-fly [transforms](#transforms) via query string.)

**Example 1**

Resize and serve `otis-04.jpg` from bucket `assets` (in account `pics`) via CDN:

```php
$http_method = 'GET';
$path = '/v0/public/pics/assets/otis-04.jpg';
$sorted_qs = 'height=400&width=600';
$secret = '3jaX4Bls9rxCiqSYfv5FaRMbfqff2Vh7';

$str = $http_method . ':' . $path . (empty($sorted_qs) ? '' : ('?' . $sorted_qs));
// => 'GET:/v0/public/pics/assets/otis-04.jpg?height=400&width=600'

$sha1 = hash_hmac('sha1', $str, $secret, true);

$sig = rtrim(strtr(base64_encode($sha1), '+/', '-_'), '=');
// => 'Ezh1DtfZNp0_vgu85UURWlnTyko'
```
=>

```bash
$ curl 'http://cdn.astra.io/v0/public/pics/assets/otis-04.jpg?width=600&height=400&hmac=Ezh1DtfZNp0_vgu85UURWlnTyko' \
    -X GET
```

=> *image streams via CDN as `Content-Type: image/jpeg`*

**Example 2**

Stream `client.js` directly from bucket `js` (in account `code`) with expiration:

```php
$http_method = 'GET';
$path = '/v0/public/code/js/client.js';
$sorted_qs = 'expires=2014-06-01T12%3A00%3A00Z'; // percent-encoded '2014-06-01T12:00:00Z'
$secret = '3jaX4Bls9rxCiqSYfv5FaRMbfqff2Vh7';

$str = $http_method . ':' . $path . (empty($sorted_qs) ? '' : ('?' . $sorted_qs));
// => 'GET:/v0/public/code/js/client.js?expires=2014-06-01T12%3A00%3A00Z'

$sha1 = hash_hmac('sha1', $str, $secret, true);

$sig = rtrim(strtr(base64_encode($sha1), '+/', '-_'), '=');
// => 'iE16Op-GwtTm_urfx6od-mQV_5A'
```

=>

```bash
$ curl 'http://api.astra.io//v0/public/code/js/client.js?expires=2014-06-01T12%3A00%3A00Z&hmac=iE16Op-GwtTm_urfx6od-mQV_5A' \
    -X GET
```

=> *blob streams directly as `Content-Type: application/javascript`*

#### `public` Update

(This aliases [`object` update](#object-update) endpoint.)

#### `public` Delete

(This aliases [`object` delete](#object-delete) endpoint.)

Transforms
----------

Transforms are object manipulations (image resizing, video cutting, etc.) that the API can perform on the fly. Transformed objects are cached, so subsequent calls with the same parameters incur no cost.

You can apply transforms by modifying the query strings you pass to the [`stream`](#stream) and [`public`](#public) endpoints. Note that transforms never modify the original objects you've uploaded.

Right now transforms cannot be combined. In the future, you'll be able to apply transforms in series.

### `blob` Object

*n/a*

### `image` Object

#### Resize Up & Down

| Field    | Type               | Description                             |
|:---------|:-------------------|:----------------------------------------|
| `width`  | integer            | image width (in pixels)                 |
| `height` | integer            | image height (in pixels)                |
| `mode`   | string (optional)  | `fill` or `fit` sizing                  |
| `orient` | boolean (optional) | apply EXIF orientation to `jpeg` images |

Both `width` and `height` are required if `mode` is not set (arbitrary resize), or if `mode` is set to `fill`. If `mode` is `fit`, then either `width` or `height` must be set (and `width` takes precedence if both).

The `orient` flag can be used with any resize operation on `jpeg` images. It's particularly useful when dealing with media uploaded from mobile devices, whose cameras often don't rotate images.

Dimensions for scaling up images are currently capped at 10,000 px each.

**Example 1**

Resize an image arbitrarily and serve directly:

```bash
$ curl 'https://api.astra.io/v0/bucket/content/stream/lol.gif?width=320&height=260' \
    -H 'Astra-Secret: 3jaX4Bls9rxCiqSYfv5FaRMbfqff2Vh7' \
    -X GET
```

=> *image streams directly as `Content-Type: image/gif`*

**Example 2**

Correctly orient an image named `iPhone.jpg`, which has rotation EXIF data:

```bash
$ curl https://api.astra.io/v0/bucket/content/stream/iPhone.jpg?orient=true \
    -H 'Astra-Secret: 3jaX4Bls9rxCiqSYfv5FaRMbfqff2Vh7' \
    -X GET
```

=> *image streams directly as `Content-Type: image/jpeg`*

**Example 3**

Orient an image (in account `biz`) and resize to completely fill a 400x400 bounding box:

```bash
$ curl 'http://cdn.astra.io/v0/public/biz/dogpics/zoe2.jpg?orient=true&mode=fill&width=400&height=400&expires=2015-01-14T16%3A30%3A00Z&hmac=yuEreJMqExv0A96Iu-HYAzMVo60' \
    -X GET
```

=> *image streams via CDN as `Content-Type: image/jpeg` until `2015-01-14T16:30:00Z`*

**Example 4**

Resize an image to fit a bounding box 320 px wide, keeping its aspect ratio, and download it:

```bash
$ curl 'https://api.astra.io/v0/bucket/bikes/stream/fixie08.jpg?mode=fit&width=320' \
    -H 'Astra-Secret: 3jaX4Bls9rxCiqSYfv5FaRMbfqff2Vh7' \
    -X GET \
    > /path/to/local/file/fixie08-resized.jpg
```

=> *image saves locally as `fixie08-resized.jpg`*

### `video` Object

#### Resize Down

| Field    | Type    | Description              |
|:---------|:--------|:-------------------------|
| `width`  | integer | video width (in pixels)  |
| `height` | integer | video height (in pixels) |

Both `width` and `height` parameters are required, and the maximum downsizing factor is 60x in both dimensions (e.g., a video 600 px wide can be scaled down no smaller than 10 px wide). Also, the `width` and `height` for `mp4`-format videos must be a multiple of 2 based on the encoder used.

**Example 1**

Resize a large video to 640x480 and download it:

```bash
$ curl 'https://api.astra.io/v0/bucket/doge/stream/wow.mp4?width=640&height=480' \
    -H 'Astra-Secret: 3jaX4Bls9rxCiqSYfv5FaRMbfqff2Vh7' \
    -X GET \
    > /path/to/local/file/wow-resized.mp4
```

=> *video saves locally as `wow-resized.mp4`*

**Example 2**

Resize a video and stream it publicly (from account `media`) with an expiration time:

```bash
$ curl 'http://api.astra.io/v0/public/media/web/X9qAkz.ogg?width=150&height=75&expires=2015-02-18T12%3A00%3A00Z&hmac=PhdpwhJ1uYdgVFXvw5v89yab2XM' \
    -X GET
```

=> *video streams directly as `Content-Type: video/webm`* until `2015-02-18T12:00:00Z`

#### Cut Segment

| Field   | Type  | Description                     |
|:--------|:------|:--------------------------------|
| `start` | float | segment start time (in seconds) |
| `end`   | float | segment end time (in seconds)   |

The `start` and `end` parameters default to 0 and the duration of the video, respectively, if omitted.

**Example 1**

Cut a 28.25-second segment from a video (in account `biz`) and stream it publicly via CDN:

```bash
$ curl 'http://cdn.astra.io/v0/public/biz/footage/interview3.webm?start=2&end=30.25&hmac=3syfFCpZR-PgoyJ_QN7kITd7Fyk' \
    -X GET
```

=> *video streams via CDN as `Content-Type: video/webm`*

**Example 2**

Trim 3.5 seconds from the beginning of a video and download it:

```bash
$ curl https://api.astra.io/v0/bucket/ocean-uncut/stream/waves.ogg?start=3.5 \
    -H 'Astra-Secret: 3jaX4Bls9rxCiqSYfv5FaRMbfqff2Vh7' \
    -X GET \
    > /path/to/local/file/waves-cut.ogg
```

=> *video saves locally as `waves-cut.ogg`*

Errors
------

If any request fails, the API will return an `error` object instead of `data`:

```json
{
    "ok": false,
    "error": {
        "type": ...,
        "code": ...,
        "message": ...
    }
}
```

The API will also set the response's [HTTP status code](http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html) accordingly.

### `ProtocolErr`

| Type                       | Code  | Message                   |
|:---------------------------|:------|:--------------------------|
| `ProtocolHTTPRequiredErr`  | `403` | `protocol http required`  |
| `ProtocolHTTPSRequiredErr` | `403` | `protocol https required` |

### `AuthErr`

| Type                   | Code  | Message                          |
|:-----------------------|:------|:---------------------------------|
| `AuthSecretMissingErr` | `401` | `request header requires secret` |
| `AuthSecretInvalidErr` | `401` | `invalid or expired secret`      |
| `AuthHMACErr`          | `401` | `invalid hmac signature`         |
| `AuthExpiredErr`       | `401` | `expired link`                   |

### `FormErr`

| Type           | Code  | Message                                            |
|:---------------|:------|:---------------------------------------------------|
| `FormFieldErr` | `400` | `field '{field_name}' required`                    |
| `FormValueErr` | `401` | `value '{value}' invalid for field '{field_name}'` |
| `FormFileErr`  | `400` | `field '{field_name}' expects input file`          |

(A `FormErr` indicates a problem with the HTML form data sent to an endpoint.)

### `BucketErr`

| Type                     | Code  | Message                              |
|:-------------------------|:------|:-------------------------------------|
| `BucketNotFoundErr`      | `404` | `bucket '{bkt_name}' not found`      |
| `BucketAlreadyExistsErr` | `409` | `bucket '{bkt_name}' already exists` |

### `ObjectErr`

| Type                     | Code  | Message                                                     |
|:-------------------------|:------|:------------------------------------------------------------|
| `ObjectNotFoundErr`      | `404` | `object '{obj_name}' not found in bucket '{bkt_name}'`      |
| `ObjectAlreadyExistsErr` | `409` | `object '{obj_name}' already exists in bucket '{bkt_name}'` |

#### `ObjectBlobErr`

*n/a*

#### `ObjectImageErr`

| Type                   | Code  | Message                                   |
|:-----------------------|:------|:------------------------------------------|
| `ObjectImageFormatErr` | `400` | `image type does not support this format` |
| `ObjectImageEmptyErr`  | `400` | `file empty or image has no width/height` |

#### `ObjectVideoErr`

| Type                   | Code  | Message                                                |
|:-----------------------|:------|:-------------------------------------------------------|
| `ObjectImageFormatErr` | `400` | `video type does not support this format`              |
| `ObjectImageEmptyErr`  | `400` | `file empty or first video stream has no width/height` |

### `InternalErr`

| Type          | Code  | Message                 |
|:--------------|:------|:------------------------|
| `InternalErr` | `500` | `internal server error` |

(An `InternalErr` signifies a server-side problem, which we'll investigate.)

Changelog
---------

### v0

#### v0.2

* 2014-11-10 - Note that update can be used to move objects
* 2014-11-03 - Add back vanilla object stream example
* 2014-11-03 - Bump version in root; revise long-form object types
* 2014-11-03 - Break out transforms; move section on versioning
* 2014-10-22 - Add video type support; orient flag for images

#### v0.1

* 2014-08-06 - Fix blob content field regex; other cleanups
* 2014-08-04 - Update header and link to control panel
* 2014-08-04 - Clarify HMAC code and add ProtocolErr type
* 2014-08-01 - Note CDN for public streams only
* 2014-08-01 - Clarify period in bucket/object names
* 2014-08-01 - Add TOC link to public endpoint
* 2014-05-13 - Correct direct streaming example
* 2014-05-13 - Fix missing name in object update path
* 2014-05-12 - Initial API release
