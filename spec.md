#Table of Contents#
  - [API Release Notes](#release-notes)
  - [Changes](#changes)
    - [Change Policy](#change-policy)
    - [Changes Since v2.10](#since-v2.10)
  - [API Overview](#api-overview)
  - [API Version Header](#version-header)
  - [Authentication](#authentication)
  - [Catalog Management](#catalog-mgmt)
    - [Adding a Broker to the Platform](#adding-broker-platform)
  - [Synchronous and Asynchronous Operations](#synchronous-asynchronous)
    - [Synchronous Operations](#synchronous-operations)
    - [Asynchronous Operations](#asynchronous-operations)
  - [Polling Last Operation](#polling)
    - [Polling Interval and Duration](#polling-interval-and-duration)
  - [Provisioning](#provisioning)
  - [Updating a Service Instance](#updating_service_instance)
  - [Binding](#binding)
    - [Types of Binding](#types-of-binding)
  - [Unbinding](#unbinding)
  - [Deprovisioning](#deprovisioning)
  - [Broker Errors](#broker-errors)
  - [Orphans](#orphans)

## Change

###Change Policy

* Existing endpoints and fields will not be removed or renamed.
* New optional endpoints, or new HTTP methods for existing endpoints, may be
added to enable support for new features.
* New fields may be added to existing request/response messages.
These fields must be optional and should be ignored by clients and servers
that do not understand them.

##Changes Since v2.10

* Add <tt>bindable</tt> field to [Plan Object](#PObject) to allow services to have both bindable and non-bindable plans.

##API Overview

The Service Broker API defines an HTTP interface between the services marketplace of a platform and service brokers.

The service broker is the component of the service that implements the Service Broker API, for which a platform's marketplace is a client. Service brokers are responsible for advertising a catalog of service offerings and service plans to the marketplace, and acting on requests from the marketplace for provisioning, binding, unbinding, and deprovisioning.

In general, provisioning reserves a resource on a service; we call this reserved resource a service instance. What a service instance represents can vary by service. Examples include a single database on a multi-tenant server, a dedicated cluster, or an account on a web application.

What a binding represents may also vary by service. In general creation of a binding either generates credentials necessary for accessing the resource or provides the service instance with information for a configuration change.

A platform marketplace may expose services from one or many service brokers, and an individual service broker may support one or many platform marketplaces using different URL prefixes and credentials.

## API Version Header

Requests from the platform to the service broker must contain a header that declares the version number of the Service Broker API that the marketplace will use:

`X-Broker-Api-Version: 2.11`

The version numbers are in the format `MAJOR.MINOR`, using semantic versioning such that 2.10 comes before 2.11.

This header allows brokers to reject requests from marketplaces for versions they do not support. While minor API revisions will always be additive, it is possible that brokers depend on a feature from a newer version of the API that is supported by the platform. In this scenario the broker may reject the request with `412 Precondition Failed` and provide a message that informs the operator of the required API version.

## Authentication

The marketplace must authenticate with the service broker using HTTP
basic authentication (the `Authorization:` header) on every request. The broker is responsible for validating the username and password and returning a `401 Unauthorized` message if credentials are invalid. It is recommended that brokers support secure communication from platform marketplaces over TLS.

## Adding a Broker to the Platform

After implementing the catalog endpoint (`GET /v2/catalog`), you must register the service broker with your platform to make your services and plans available to end users.

## Synchronous and Asynchronous Operations

Broker clients expect prompt responses to all API requests in order to provide users with fast feedback. Service broker authors should implement their brokers to respond promptly to all requests but must decide whether to implement synchronous or asynchronous responses. Brokers that can guarantee completion of the requested operation with the response should return the synchronous response. Brokers that cannot guarantee completion of the operation with the response should implement the asynchronous response.

Providing a synchronous response for a provision, update, or bind operation before actual completion causes confusion for users as their service may not be usable and they have no way to find out when it will be. Asynchronous responses set expectations for users that an operation is in progress and can also provide updates on the status of the operation.

Support for synchronous or asynchronous responses may vary by service offering, even by service plan.

### Synchronous Operations

To execute a request synchronously, the broker need only return the usual status codes: `201 CREATED` for provision and bind, and `200 OK` for update, unbind, and deprovision.

Brokers that support sychronous responses for provision, update, and delete can ignore the `accepts_incomplete=true` query parameter if it is provided by the client.

### Asynchronous Operations

<p class='note'><strong>Note:</strong> Asynchronous operations are currently supported only for provision, update, and deprovision.</p>

For a broker to return an asynchronous response, the query parameter `accepts_incomplete=true` must be included the request. If the parameter is not included or is set to `false`, and the broker cannot fulfill the request synchronously (guaranteeing that the operation is complete on response), then the broker should reject the request with the status code `422 UNPROCESSABLE ENTITY` and the following body:

<pre class="terminal">
{
  "error": "AsyncRequired",
  "description": "This service plan requires client support for asynchronous service operations."
}
</pre>

If the query parameter described above is present, and the broker executes the request asynchronously, the broker must return the asynchronous response `202 ACCEPTED`. The response body should be the same as if the broker were serving the request synchronously.

An asynchronous response triggers the platform marketplace to poll the endpoint `GET /v2/service_instances/:guid/last_operation` until the broker indicates that the requested operation has succeeded or failed. Brokers may include a status message with each response for the `last_operation` endpoint that provides visibility to end users as to the progress of the operation.

#### Blocking Operations ####

The marketplace must ensure that service brokers do not receive requests for an instance while an asynchronous operation is in progress. For example, if a broker is in the process of provisioning an instance asynchronously, the marketplace must not allow any update, bind, unbind, or deprovision requests to be made through the platform. A user who attempts to perform one of these actions while an operation is already in progress must receive an HTTP 400 response with the error message: `Another operation for this service instance is in progress`.

## Polling Last Operation

When a broker returns status code `202 ACCEPTED` for [provision](#provisioning), [update](#updating_service_instance), or [deprovision](#deprovisioning), the platform will begin polling the `/v2/service_instances/:guid/last_operation` endpoint to obtain the state of the last requested operation. The broker response must contain the field `state` and an optional field `description`.

Valid values for `state` are `in progress`, `succeeded`, and `failed`. The platform will poll the `last_operation` endpoint as long as the broker returns `"state": "in progress"`. Returning `"state": "succeeded"` or `"state": "failed"` will cause the platform to cease polling. The value provided for `description` will be passed through to the platform API client and can be used to provide additional detail for users about the progress of the operation.

### Request ###

##### Route #####
`GET /v2/service_instances/:instance_id/last_operation`

##### Parameters #####

The request provides these query string parameters as useful hints for brokers.

|  Query-String Field | Type  | Description  |
|---|---|---|
| service_id  |  string | ID of the service from the catalog.  |
| plan_id  | string  | ID of the plan from the catalog.  |
| operation  |  string | A broker-provided identifier for the operation. When a value for <code>operation</code> is included with asynchronous responses for [Provision](#provisioning), [Update](#updating_service_instance), and [Deprovision](#deprovisioning) requests, the broker client should provide the same value using this query parameter as a URL-encoded string.  |

<p class="note"><strong>Note:</strong> Although the request query parameters <code>service_id</code> and <code>plan_id</code> are not required, the platform should include them on all <code>last_operation</code> requests it makes to service brokers.</p>

##### cURL #####
<pre class="terminal">
$ curl http://username:password@broker-url/v2/service_instances/:instance_id/last_operation
</pre>

### Response ###

| Status Code  |  Description |
|---|---|
| 200 OK |  The expected response body is below. |
|  410 GONE | Appropriate only for asynchronous delete operations. The platform should consider this response a success and remove the resource from its database. The expected response body is <code>{}</code>. Returning this while the platform is polling for create or update operations should be interpreted as an invalid response and the platform should continue polling.  |

Responses with any other status code should be interpreted as an error or invalid response. The platform should continue polling until the broker returns a valid response or the [maximum polling duration](#polling-interval-and-duration) is reached. Brokers may use the `description` field to expose user-facing error messages about the operation state; for more info see [Broker Errors](#broker-errors).

##### Body #####

All response bodies must be a valid JSON Object (`{}`). This is for future compatibility; it will be easier to add fields in the future if JSON is expected rather than to support the cases when a JSON body may or may not be returned.

For success responses, the following fields are valid.

|  Response field | Type  | Description  |
|---|---|---|
| state*  | string  | Valid values are <code>in progress</code>, <code>succeeded</code>, and <code>failed</code>. While <code>"state": "in progress"</code>, the platform should continue polling. A response with <code>"state": "succeeded"</code> or <code>"state": "failed"</code> should cause the platform to cease polling.  |
|  description |  string | Optional field. A user-facing message displayed to the platform API client. Can be used to tell the user details about the status of the operation.  |

\* Fields with an asterisk are required.

<pre class="terminal">
{
  "state": "in progress",
  "description": "Creating service (10% complete)."
}
</pre>

### Polling Interval and Duration

The frequency and maximum duration of polling may vary by platform client. If a platform has a max polling duration and this limit is reached, the platform will cease polling and the operation state will be considered `failed`.

## Broker Errors

Broker failures beyond the scope of the well-defined HTTP response codes defined in the specification should return an appropriate HTTP response code (chosen to accurately reflect the nature of the failure) and a body containing a valid JSON Object, possibly including a `description` field (string) containing a meaningful error message explaining why the request failed.

## Orphans

The platform marketplace is the source of truth for service instances and bindings. Service brokers are expected to have successfully provisioned all the instances and bindings that the marketplace knows about, and none that it doesn't.

Orphans can result if the broker does not return a response before a request from the marketplace times out (typically 60 seconds). For example, if a broker does not return a response to a provision request before the request times out, the broker might eventually succeed in provisioning an instance after the marketplace considers the request a failure. This results in an orphan instance on the broker's side.

To mitigate orphan instances and bindings, the marketplace should attempt to delete resources it cannot be sure were successfully created, and should keep trying to delete them until the broker responds with a success.

Platforms should initiate orphan mitigation in the following scenarios:

| Status code of broker response | Platform interpretation of response | Platform initiates orphan mitigation? |
|---|---|---|
| 200 | Success | No |
| 200 with malformed response |  Failure | No |
| 201 | Success | No |
| 201 with malformed response | Failure | Yes |
| All other 2xx | Failure | Yes |
| 408 | Failure due to timeout | Yes |
| All other 4xx | Broker rejected request | No |
| 5xx | Broker error | Yes |
| Timeout | Failure | Yes |

If the platform marketplace encounters an internal error provisioning an instance or binding (for example, saving to the database fails), then it should at least send a single delete or unbind request to the service broker to prevent creation of an orphan.

## Binding

If `bindable:true` is declared for a service or plan in the cataolog endpoint, broker clients may request generation of a service binding.

**Note**: Not all services must be bindable. Some deliver value just from being provisioned. Brokers that offer services that are bindable should declare them as such using `bindable: true` in the Catalog. Brokers that do not offer any bindable services do not need to implement the endpoint for bind requests.

### Types of Binding

#### Credentials ####

Credentials are a set of information used by an application or a user to utilize the service instance. If the broker supports generation of credentials it should return `credentials` in the response for a request to create a service binding. Credentials should be unique whenever possible, so access can be revoked for each binding without affecting consumers of other bindings for the service instance.

#### Log Drain ####

There are a class of service offerings that provide aggregation, indexing, and analysis of log data. To utilize these services an application that generates logs needs information for the location to which it should stream logs. If a broker represents one of these services, it may optionally return a `syslog_drain_url` in the response for a request to create a service binding, to which logs may be streamed.

The `requires` field in the catalog endpoint enables a platform marketplace to validate a response for create binding that includes a `syslog_drain_url`. Platform marketplaces should consider a broker's response invalid if it includes a `syslog_drain_url` and `"requires":["syslog_drain"]` is not present in the catalog endpoint.

#### Route Services

There are a class of service offerings that intermediate requests to applications, performing functions such as rate limiting or authorization. To configure a service instance with behavior specific to an application's routable address, a broker client may send the address along with the request to create a binding using `"bind_resource":{"route":"some-address.com"}`.

Some platforms may support proxying of application requests to service instances. In this case the platform needs to know where to send application requests; to facilitate this, the broker may return a `route_service_url` in the response for a request to create a binding. Not all services of this type expect to receive requests proxied by the platform; some services will have been configured out-of-band to intermediate requests to applications. In this case, the broker will not return `route_service_url` in response to the create binding request. By sending `bind-resource` as described above, the platform enables dynamic configuration of a service instance already in the application request path for the route, requiring no change in the platform routing tier.

The `requires` field in the catalog endpoint enables a platform marketplace to validate requests to create bindings. A platform may opt to reject requests to create bindings when a broker has declared `"requires":["route_forwarding"]` for a service in the catalog endpoint.

#### Volume Services ####

There are a class of services that provide network storage to applications via volume mounts in the application container. A service broker may return data required for this configuration with `volume_mount` in response to the request to create a binding.

The `requires` field in the catalog endpoint enables a platform marketplace to validate a response for create binding that includes a `volume_mounts`. Platform marketplaces should consider a broker's response invalid if it includes a `volume_mounts` and `"requires":["volume_mount"]` is not present in the catalog endpoint.


----

### Request ###

##### Route #####
`PUT /v2/service_instances/:instance_id/service_bindings/:binding_id`

The `:instance_id` is the ID of a previously-provisioned service instance. The `:binding_id` is also provided by the platform. This ID will be used for future unbind requests, so the broker must use it to correlate
the resource it creates.

##### Body #####

| Request Field  | Type  | Description  |
|---|---|---|
| service_id*  | string  | ID of the service from the catalog.  |
| plan_id*  | string  | ID of the plan from the catalog.  |
| app_guid  | string  | Deprecated in favor of <code>bind\_resource.app\_guid</code>. GUID of an application associated with the binding to be created.  |
| bind_resource  | JSON object  | A JSON object that contains data for platform resources associated with the binding to be created. Current valid values include <code>app\_guid</code> for [credentials](#types-of-binding) and <code>route</code> for [route services](#route_services).  |
| parameters | JSON object  |  Configuration options for the service binding. An opaque object, controller treats this as a blob.  |   |

\* Fields with an asterisk are required.

<pre class="terminal">
{
  "service_id": "service-guid-here",
  "plan_id": "plan-guid-here",
  "bind_resource": {
    "app_guid": "app-guid-here"
  },
  "parameters": {
    "parameter1-name-here": 1,
    "parameter2-name-here": "parameter2-value-here"
  }
}
</pre>


##### cURL #####
<pre class="terminal">
$ curl http://username:password@broker-url/v2/service_instances/:instance_id/service_bindings/:binding_id -d '{
  "service_id": "service-guid-here",
  "plan_id": "plan-guid-here",
  "bind_resource": {
    "app_guid": "app-guid-here"
  },
  "parameters": {
    "parameter1-name-here": 1,
    "parameter2-name-here": "parameter2-value-here"
  }
}' -X PUT
</pre>

### Response ###

| Status Code  | Description  |
|---|---|
| 201 Created  |  Binding has been created. The expected response body is below. |
| 200 OK  |  May be returned if the binding already exists and the requested parameters are identical to the existing binding. The expected response body is below. |
| 409 Conflict  | Should be returned if the requested binding already exists. The expected response body is <code>{}</code>, though the description field can be used to return a user-facing error message, as described in [Broker Errors](#broker-errors).  |
| 422 Unprocessable Entity  | Should be returned if the broker requires that <code>app_guid</code> be included in the request body. The expected response body is: <code>{ "error": "RequiresApp", "description": "This service supports generation of credentials through binding an application only." }</code>  |

Responses with any other status code will be interpreted as a failure and an unbind request will be sent to the broker to prevent an orphan being created on the broker. Brokers can include a user-facing message in the `description` field; for details see [Broker Errors](#broker-errors).

##### Body #####

All response bodies must be a valid JSON Object (`{}`). This is for future compatibility; it will be easier to add fields in the future if JSON is expected rather than to support the cases when a JSON body may or may not be returned.

For success responses, the following fields are supported. Others will be ignored. For error responses, see [Broker Errors](#broker-errors).

|  Response Field | Type  | Description  |
|---|---|---|
| credentials  | object  |  A free-form hash of credentials that may be used by applications or users to access the service. |
| syslog\_drain_url  | string  | A URL to which logs may be streamed. <code>"requires":["syslog\_drain"]</code> must be declared in the [Catalog](#catalog-mgmt) endpoint or the platform should consider the response invalid.  |
| route\_service_url  | string  |  A URL to which the platform may proxy requests for the address sent with <code>bind\_resource.route</code> in the request body. <code>"requires":["route\_forwarding"]</code> must be declared in the [Catalog](#catalog-mgmt) endpoint or the platform should consider the response invalid. |
| volume\_mounts  | array-of-objects  | An array of configuration for mounting volumes. <code>"requires":["volume_mount"]</code> must be declared in the [Catalog](#catalog-mgmt) endpoint or the platform should consider the response invalid.  |

<pre class="terminal">
    {
      "credentials": {
        "uri": "mysql://mysqluser:pass@mysqlhost:3306/dbname",
        "username": "mysqluser",
        "password": "pass",
        "host": "mysqlhost",
        "port": 3306,
        "database": "dbname"
      }
    }
</pre>

## Unbinding

<p class="note"><strong>Note</strong>: Brokers that do not provide any bindable services or plans do not need to implement this endpoint.</p>

When a broker receives an unbind request from the marketplace, it should delete any resources associated with the binding. In the case where credentials were generated, this may result in requests to the service instance failing to authenticate.

### Request ###

##### Route #####
`DELETE /v2/service_instances/:instance_id/service_bindings/:binding_id`

The `:instance_id` is the ID of a previously-provisioned service instance. The `:binding_id` is the ID of a previously provisioned binding for that instance.

##### Parameters #####

The request provides these query string parameters as useful hints for brokers.

| Query-String Field  | Type  | Description |
|---|---|---|
| service_id*  | string | ID of the service from the catalog. |
| plan_id*  | string  | ID of the plan from the catalog. |

\* Query parameters with an asterisk are required.

##### cURL #####
<pre class="terminal">
$ curl 'http://username:password@broker-url/v2/service_instances/:instance_id/
  service_bindings/:binding_id?service_id=service-id-here&plan_id=plan-id-here' -X DELETE -H "X-Broker-API-Version: 2.11"
</pre>

### Response ###

| Status Code  | Description  |
|---|---|
| 200 OK  | Binding was deleted. The expected response body is <code>{}</code>.  |
| 410 Gone  | Should be returned if the binding does not exist. The expected response body is <code>{}</code>.  |

Responses with any other status code will be interpreted as a failure and the binding will remain in the marketplace database. Brokers can include a user-facing message in the `description` field; for details see [Broker Errors](#broker-errors).

##### Body #####

All response bodies must be a valid JSON Object (`{}`). This is for future compatibility; it will be easier to add fields in the future if JSON is expected rather than to support the cases when a JSON body may or may not be returned.

For a success response, the expected response body is `{}`.

## Deprovisioning

When a broker receives a deprovision request from the marketplace, it should
delete any resources it created during the provision.
Usually this means that all resources are immediately reclaimed for future
provisions.

