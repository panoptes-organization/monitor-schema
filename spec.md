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
	2. [Status](#status)
	3. [Create Workflow](#create-workflow)
	4. [Update Workflow](#update-workflow)
	5. [Get Workflow](#get-workflow)
	6. [Delete Workflow](#delete-workflow)
	7. [Rename Workflow](#rename-workflow)
	8. [Get Workflows](#workflows)
	9. [Delete Workflows](#delete-workflows)
	10. [Workflow Jobs](#workflow-jobs)
	11. [Get a Workflow Job](#get-workflow-job)

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
 
1. **Service Info** (`GET /m1/`) endpoint with a 200 or 30* response.
2. **Create New Workflow** (`GET /m1/workflow/create/`) to create a workflow
3. **Update Workflow** (`POST /m1/workflow/<workflow-id>/`) to update an existing workflow
4. **Get Workflow** (`GET /m1/workflow/<workflow-id>/`)

Extra (but not required) endpoints include:

1. **Statuses**: (`GET /m1/statuses/`) to see all known workflow statuses. If not implemented, assume the default.
2. **Workflows** (`GET /m1/workflows/`) to see all workflows known to a server.
3. **Workflow Jobs** (`GET /m1/workflow/<workflow_id>/jobs/`) to see all jobs that belong to a workflow.
4. **Get Workflow Job**: (`GET /m1/workflow/<workflow-id>/job/<job-id>/`) to get a particular job belonging to a workflow
5. **Delete Workflow** (`DELETE /m1/workflow/<workflow-id>/`) to delete a workflow
6. **Rename Workflow**: (`PUT /m1/workflow/<workflow-id>`) to update the name of a workflow (rename)
7. **Delete Workflows** (`DELETE /m1/workflows/`) to delete all workflows

### Response Details

#### Errors

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

#### Timestamps

For all fields that return a timestamp, we are tentatively going to use the stringified
version of a `datetime.now()`, which looks like this:

```
2020-12-15 11:43:24.811860
```

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

```
{"status": "running", "version": "1.0.0"}
```

##### 404

In the case of a 404 response, it means that the server does not implement the monitor spec.
The client should stop, and then respond appropriately (e.g., giving an error message or warning to the user).

```
{"status": "not implemented", "version": "1.0.0"}
```
##### 200

A 200 is a successful response, meaning that the endpoint was found, and is running.

```
{"status": "running", "version": "1.0.0"}
```

##### 503

If the service exists but is not running, a 503 is returned. The client should respond in the same
way as the 404, except perhaps trying later.

```
{"status": "service not available", "version": "1.0.0"}
```

##### 302

A 302 is a special status intended to support version changes in a server. For example,
let's say that an updated spec is served at `/m2/` and by default, a client knows to
send a request to `/m1/`. To give the client instruction to use `/m2/` for all further
interactions, the server would return a 302 response

```
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

#### Status

`GET /m1/statuses`

The status endpoint exists for a client to retrieve a complete list of possible
statuses of a workflow that can be returned, along with descriptionif needed. 
At a minimum, the Monitor Schema expects:

 - **running**: a workflow is currently running
 - **pending**: a working has not started
 - **error**: the job exited or completed with error
 - **completed**: a workflow has finished running.

For the general statuses above, completed could indicate a full completion with
success, failure, or a partial completion with an exit of error. The client
would then inspect the workflow for more details about the status. Other
statuses are allowed to be added by the server, as long as they are described here.
For example, here is what the response might look like:

```
{"statuses": [{
      "name": "running",
      "description": "The workflow is currently running" ,
    },{
      "name": "pending",
      "description": "The workflow has not started, and is not scheduled." ,
    },{
      "name": "scheduled",
      "description": "The workflow has not started, and is scheduled." ,
    },{
      "name": "error",
      "description": "The workflow exited with an error." ,
    },{
      "name": "completed",
      "description": "The workflow has finished." ,
    },
}
```

Notice how we've added a "scheduled" status to indicate a more complex system
that can have jobs submit but not scheduled (pending) and submit and
scheduled to run (scheduled).

#### Create Workflow

`GET /m1/workflow/create/`

A request to create a workflow should come as a `POST` from the client. The id would then be returned to the client
to track the workflow further. The server can respond in any of the following ways:

- [404](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/404): not implemented
- [201](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/201): success (workflow created)

##### 404

A 404 response indicates the endpoint is not found. The client should not proceed.

##### 201

A 201 is a success response to indicate that the workflow was created. As data,
an id should be returned in the body:

```
{"id": "123456"}
```

and (OPTIONAL) a Location header to indicate a URL to browse to in order to see
progress or metadata about the workflow:

```
Location: /workflow/123456/
```

This URL typically responds to the API view to get the workflow.
The receiving server can create whatever internal database models are required to keep
track of the workflow. This is up to the server implementation.

#### Update Workflow

`POST /m1/workflow/<workflow-id>/`

An update workflow request is done after workflow creation, intended to be sent from the client
to add metadata to a workflow or otherwise update it. The client should send the following
data as a `POST` request:

```
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


```
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

#### Get Workflow

`GET /m1/workflow/<workflow-id>/`

This endpoint returns metadata about a particular workflow, retrieved based on the id.

- [404](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/404): not implemented
- [200](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/200): success

The highest level of the response looks like this:

```
{"workflow": ...}
```

Where the workflow object is a single (workflow item)[#workflow-item], akin to one returned in the
listing at `/m1/workflows/`.

##### 200

A 200 response returns the serialized data above, and indicates that the request
was successful.

##### 404 

A 404 response indicates that the endpoint is not implemented.


##### Workflow Item

A workflow item is a dictionary that has the following attributes:

 - **id**: the id of the workflow, as understood by the monitoring server
 - **name**: the name of the workflow
 - **status**: the status of the workflow, must be one listed from the `/m1/statuses` endpoint.
 - **started_at**: the time that the workflow was started
 - **completed_at**: the time that the workflow was completed (or errored and stopped)

The following fields are OPTIONAL, and useful if the workflow is more complex
and might be broken into jobs.

 - **jobs_total**: the total number of jobs in the workflow
 - **jobs_done**: the number of jobs completed at the time of the request
                }

#### Delete Workflow

`DELETE /m1/workflow/<workflow-id>/`

A DELETE request to the workflow endpoint will delete the workflow from the database.
It's advised for this view to be protected with authentication, although it's not required.
Responses include:

- [404](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/404): not found or not implemented
- [403](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/403): permission denied
- [204](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/204): successful delete
- [507](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/507): not enough information


##### 404

A 404 response indicates that the endpoint is not implemented, or the workflow was not
found based on the id.

##### 403

A 403 generally means permission denied, and can result from finding a workflow running
(we do not allow deleting a running workflow) or not passing authentication. Although
not part of the spec, typically a 401 would be returned if the user does the request
without authentication and it's required. Authentication, however, is outside the scope
of this spec.

##### 204

A 204 response indicates that the workflow was successfully deleted.

##### 507

A 507 response indicates that "the server is unable to store the representation needed to complete the delete request."


#### Rename Workflow

`PUT /m1/workflow/<workflow_id>/`

This is a scoped update for a workflow, meaning that we only allow changing the name of the
workflow. Responses include:

- [404](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/404): not found or not implemented
- [400](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/400): bad request
- [200](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/200): successful rename
- [500](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/500): there was an error in renaming.

Generally, a correct body should provide the name in the data.

```
{"name": "New Workflow Name"}
```

##### 404

A 404 response indicates that the workflow was not found, or the endpoint does not exist.

##### 400

A 400 response indicates that the request body was malformed, empty, or missing.

##### 200

A 200 response indicates a successful rename, and we return the same serialized
data as "Get Workflow" `/m1/workflow/<workflow-id/` with the updated information.

##### 500

A 500 response returns [errors](#errors).

#### Workflows

`GET /m1/workflows/`

The workflows endpoint returns a list of serialized workflows with high
level metadata and timing information. The following responses can be returned:

- [404](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/404): not implemented
- [200](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/200): success

The highest level of the response looks like this:

```
{"workflows": [], count: 0}
```

Where workflows is a list of [workflow items](#workflow-item), and count is the number of workflows
in the list. 

##### 200

A 200 response returns the serialized data above, and indicates that the request
was successful.

##### 404 

A 404 response indicates that the endpoint is not implemented.


#### Delete Workflows

`DELETE /m1/workflows/`

This endpoint should be implemented with caution (OPTIONAL) as it would delete
all workflows in the database. It is also advised to have authentication here,
although it's not officially part of the Monitor spec. The following responses
can be returned

- [404](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/404): not implemented
- [410](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/410): gone
- [200](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/200): successful delete
- [507](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/507): unable to complete

##### 404

A 404 response indicates that the method is not implemented.

##### 200

A 200 is a successful response, indicating that all workflows were deleted.

##### 410

A 410 response indicates that the table is already empty. 

##### 407

A 407 response indicates that "the server is unable to store the representation needed to complete the delete request."


#### Workflow Jobs

`GET /m1/workflow/<workflow_id>/jobs/`

A workflow can optionally be broken into jobs. This endpoint is OPTIONAL, and will
return a listing of jobs that belong to a workflow. The following responses can be returned:

- [404](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/404): not implemented
- [200](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/200): success

The highest level of the response looks like this:

```
{"jobs": [], count: 0}
```

Where jobs is a list of [job items](#job-item), and count is the number of jobs
in the list. 

##### Job Item

A job item is a dictionary (data) with the following attributes:

 - **jobid**: the id of the job, if defined
 - **workflow_id**: the id of the parent workflow
 - **name**: the name of the job step
 - **input**: a list of inputs for the step
 - **output**: a list of outputs for the step
 - **status**: the job status, one returned from `/m1/statuses/` or the default
 - **started_at**: the time the job started
 - **completed_at**: the time the job was completed or errored and stopped
 - **log**: content from logging


And any extra parameters (e.g., for an implementation like Snakemake) are not
assumed to exist, and should be checked for existence before indexing on:

 - **message**: an error or output message from the job
 - **wildcards**: wildcards specific to the job (e.g., snakemake)
 - **is_checkpoint**: is the job step a checkpoint?

##### 200

A 200 response returns the serialized data above, and indicates that the request
was successful.

##### 404 

A 404 response indicates that the endpoint is not implemented.

#### Get Workflow Job

`GET /m1/workflow/<workflow_id>/job/<job_id>/`

A single, specific job can be retrieved via this endpoint. Responses include:

- [404](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/404): not implemented
- [200](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/200): success

And given a successful response, the same list of [job items](#job-item) is returned
as described in [workflow jobs](#workflow-jobs). The only difference is that the
jobs key is a list of one item. E.g.,:

```
{"jobs": [], count: 1}
```

##### 200

A 200 response returns the single job in a list above.


##### 404

A 404 response can mean that the job was not found based on the id, or that
the endpoint is not implemented.
