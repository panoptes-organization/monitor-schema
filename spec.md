# Panoptes Monitor Specification

## Table of Contents

- [Overview](#overview)
  - [Introduction](#introduction)
  - [Definitions](#defintions)
- [Notational Conventions](#notational-conventions)
- [Conformance](#conformance)
  - [Determining Support](#determining-support)
  - [Endpoint Requirements](#endpoint-requirements)
  - [Response Details](#response-details)
  - [Endpoint Details](#endpoint-details)
	1. [Service Info](#service-info)
	2. [Create Workflow](#create-workflow)
	3. [Update Workflow](#update-workflow)


## Overview

### Introduction

The **Panoptes Monitor Specification** (the "monitor spec") defines an API protocol 
to standardize the requests and responses for clients and servers to communicate about
some application status. In other words, the spec facilitates monitoring.

### Definitions

The following terms are used commonly in this document, and a list of definitions is provided for reference:

- **Server**: a service that provides the endpoints defined in this spec
- **Client**: an application or tool that interacts with a Server.

## Notational Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" are to be interpreted as described in [RFC 2119](http://tools.ietf.org/html/rfc2119) (Bradner, S., "Key words for use in RFCs to Indicate Requirement Levels", BCP 14, RFC 2119, March 1997).


## Conformance

Currently, we don't have any tools to test conformance, and the requirements are outlined here. 

### Determining Support

To check whether or not the server implements the monitor spec, the client SHOULD 
perform a `GET` request to the `/m1/` (service info) endpoint.
If the response is `200 OK`, then the server implements the spec. This particular endpoint
MAY be used for authentication, however authentication is outside of the scope of this spec.

For example, given a url prefix of `http://127.0.0.0:5000` the client would issue a `GET`
request to:

```
http://127.0.0.1:5000/m1/
```

And see [the service info](#service-info) section for more details on this request and response.
**Note** this prefix can be up for debate, but it should be unique enough to be easy to
integrate into a web application, and not have any conflicts. In practice the client should
always be able to go to `/m1/` and be guaranteed to find either instruction to use
a different endpoint, a redirect, or a successful response.

All of the following would be valid:

```
https://workflows-for-the-win.org/m1/
http://workflows-for-the-win.org/m1/
http://localhost/m1/
http://127.0.0.1/m1/
http://127.0.0.1:8282/m1/
```

And for each of the above, the client implementing the spec would be provided the url
before `/m1/` (e.g., https://workflows-for-the-win.org/) and then use that to assemble
all the endpoints.

### Endpoint Requirements

Servers conforming to the Monitor spec must provide the following endpoints: 

The most basic requirements for a server include endpoints:
 
1. **Service Info** (/m1/) endpoint with a 200 or 30* response.
2. **Create New Workflow** (/m1/workflow/create/) to create a workflow
3. **Update Workflow** (/m1/workflow/update/) to update an existing workflow

### Response Details

For all error responses, the server can (OPTIONAL) return in the body a nested structure of errors,
each including a message and error code. For example:

```HTTP
{
    "errors": [
        {
            "code": "<error code>",
            "message": "<error message>",
            "detail": ...
        },
        ...
    ]
}
```

Currently we don't have a namespace for errors, but this can be developed if/when needed.
For now, the code can be a standard server error code.

### Endpoint Details

#### Service Info

`GET /m1/`

This particular Endpoint exists to check the status of a running monitor service. It replaces the previously
implemented `service-info` to be the root of the namespace for the Monitor Spec. The client
should issue a `GET` request to this endpoint without any data, and the response should be any of the following:

- [404](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/404): not implemented
- [200](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/200): success (indicates running)
- [503](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/503): service not available
- [302](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/302): found, change namespace
- [301](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/301): redirect

As the initial entrypoint, this endpoint also can communicate back to the client that the prefix (m1)
has changed (e.g., response 302 with a Location header). More detail about the use case for each return code is provided below.
For each of the above, the minimal response returned should include in the body a status message
and a version, both strings:

```json
{"status": "success", "version": "1.0.0"}
```

##### 404

In the case of a 404 response, it means that the server does not implement the monitor spec.
The client should stop, and then respond appropriately (e.g., giving an error message or warning to the user).

```json
{"status": "not implemented", "version": "1.0.0"}
```
##### 200

A 200 is a successful response, meaning that the endpoint was found, and is running.

```json
{"status": "running", "version": "1.0.0"}
```

##### 503

If the service exists but is not running, a 503 is returned. The client should respond in the same
way as the 404, except perhaps trying later.

```json
{"status": "service not available", "version": "1.0.0"}
```

##### 302

A 302 is a special status intended to support version changes in a server. For example,
let's say that an updated spec is served at `/m2/` and by default, a client knows to
send a request to `/m1/`. To give the client instruction to use `/m2/` for all further
interactions, the server would return a 302 response

```json
{"status": "multiple choices", "version": "1.0.0"}
```

with a [location](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Location) 
header to indicate the updated url prefix:

```
Location: /m2/
```

And the client would update all further prefixes accordingly.

##### 301

A 301 is a more traditional redirect that is intended for one off redirects, but
not necessarily indicatig to change the entire client namespace. For example,
if the server wanted the client to redirect `/m1/` to be `/service-info/` (but only
for this one case) the response would be:

```
{"status": "multiple choices", "version": "1.0.0"}
```

With a location header for just this request:

```
Location: /service-info/
```

For each of the above, if the server does not return a Location header, the client
should issue an error.

#### Create Workflow

`GET /m1/workflow/create/`

A request to create a workflow should come as a `POST` from the client. The id would then be returned to the client
to track the workflow further. The server can respond in any of the following ways:

- [404](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/404): not implemented
- [201](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/200): success (workflow created)

##### 404

A 404 response indicates the endpoint is not found. The client should not proceed.

##### 201

A 201 is a success response to indicate that the workflow was created. As data,
an id should be returned in the body:

```
{"id": "123456"}
```

and (OPTIONALLY) a Location header to indicate a URL to browse to in order to see
progress or metadata about the workflow:

```
Location: /workflows/123456/
```

The receiving server can create whatever internal database models are required to keep
track of the workflow. This is up to the server implementation.

#### Update Workflow

`POST /m1/workflow/update/`

An update workflow request is done after workflow creation, intended to be sent from the client
to add metadata to a workflow or otherwise update it. The client should send the following
data as a `POST` request:

```json
{
  "message": <message>,
  "timestamp": <timestamp>,
  "id": <workflow-id>
}
```

Where `<message>` is a json dictionary of metadata about the updated workflow.
This minimally `MUST` have a jobid (a workflow can have one or more jobs), 
however the following other fields can be provided (OPTIONAL). These
are derived from the snakemake logging utility:


```json
...message : {
     "jobid": "123456",
     "level": "info",
     "name": "register brainmap",
     "input": ["brain.niigz", "MNI152.nii.gz"],
     "output": ["registered-brain.nii.gz"],
     "log": "This is a longer message..."
}
```

 - **level**: is an optional verbosity level, in case the interface / server can filter based on it.
 - **jobid**: the unique id of the job run.
 - **name**: the name of the job
 - **input**: a list of inputs 
 - **output**: a list of expected or existing outputs
 - **log**: content that would go to logging

The server and client can accept other OPTIONAL key value pairs, but they aren't required here.
As an example, snakemake has **is_checkpoint**.

The following responses can be returned:

- [404](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/404): not implemented or found
- [202](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/202): success (workflow updated)


##### 404

A 404 response indicates that the id is not found, or the endpoint is not
implemented.

##### 202

A 202 response indicates that the server accepted the updates (success) and that the workflow was
updated. A Location header (OPTIONAL) can be returned again with a web address for the workflow.

```
Location: <location>
```
