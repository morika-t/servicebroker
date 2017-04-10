# Table of Contents
  - [API Release Notes](#release-notes)
  - [Changes](#changes)
    - [Change Policy](#change-policy)
    - [Changes Since v2.10](#since-v2.10)
  - [API Overview](#api-overview)
  - [API Version Header](#version-header)
  - [Authentication](#authentication)
  - [Synchronous and Asynchronous Operations](#synchronous-asynchronous)
    - [Synchronous Operations](#synchronous-operations)
    - [Asynchronous Operations](#asynchronous-operations)
    - [Blocking Operations](#blocking-operations)
  - [Polling Last Operation](#polling)
    - [Polling Interval and Duration](#polling-interval-and-duration)
  - [Binding](#binding)
    - [Types of Binding](#types-of-binding)
  - [Unbinding](#unbinding)
  - [Deprovisioning](#deprovisioning)
  - [Broker Errors](#broker-errors)
  - [Orphans](#orphans)

## Change

### Change Policy

* Existing endpoints and fields will not be removed or renamed.
* New optional endpoints, or new HTTP methods for existing endpoints, may be
added to enable support for new features.
* New fields may be added to existing request/response messages.
These fields must be optional and should be ignored by clients and servers
that do not understand them.

### Changes Since v2.10

* Add `bindable` field to plans, to allow services to have both bindable and non-bindable plans.

## API Overview

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

The marketplace must authenticate with the service broker using HTTP basic authentication (the `Authorization:` header) on every request. The broker is responsible for validating the username and password and returning a `401 Unauthorized` message if credentials are invalid. It is recommended that brokers support secure communication from platform marketplaces over TLS.

## Synchronous and Asynchronous Operations

Broker clients expect prompt responses to all API requests in order to provide users with fast feedback. Service broker authors should implement their brokers to respond promptly to all requests but must decide whether to implement synchronous or asynchronous responses. Brokers that can guarantee completion of the requested operation with the response should return the synchronous response. Brokers that cannot guarantee completion of the operation with the response should implement the asynchronous response.

Providing a synchronous response for a provision, update, or bind operation before actual completion causes confusion for users as their service may not be usable and they have no way to find out when it will be. Asynchronous responses set expectations for users that an operation is in progress and can also provide updates on the status of the operation.

Support for synchronous or asynchronous responses may vary by service offering, even by service plan.

### Synchronous Operations

To execute a request synchronously, the broker need only return the usual status codes: `201 CREATED` for provision and bind, and `200 OK` for update, unbind, and deprovision.

Brokers that support sychronous responses for provision, update, and delete can ignore the `accepts_incomplete=true` query parameter if it is provided by the client.

### Asynchronous Operations

**Note:** Asynchronous operations are currently supported only for provision, update, and deprovision.

For a broker to return an asynchronous response, the query parameter `accepts_incomplete=true` must be included the request. If the parameter is not included or is set to `false`, and the broker cannot fulfill the request synchronously (guaranteeing that the operation is complete on response), then the broker should reject the request with the status code `422 UNPROCESSABLE ENTITY` and the following body:

```
{
  "error": "AsyncRequired",
  "description": "This service plan requires client support for asynchronous service operations."
}
```

If the query parameter described above is present, and the broker executes the request asynchronously, the broker must return the asynchronous response `202 ACCEPTED`. The response body should be the same as if the broker were serving the request synchronously.

An asynchronous response triggers the platform marketplace to poll the endpoint `GET /v2/service_instances/:guid/last_operation` until the broker indicates that the requested operation has succeeded or failed. Brokers may include a status message with each response for the `last_operation` endpoint that provides visibility to end users as to the progress of the operation.

### Blocking Operations

The marketplace must ensure that service brokers do not receive requests for an instance while an asynchronous operation is in progress. For example, if a broker is in the process of provisioning an instance asynchronously, the marketplace must not allow any update, bind, unbind, or deprovision requests to be made through the platform. A user who attempts to perform one of these actions while an operation is already in progress must receive an HTTP 400 response with the error message: `Another operation for this service instance is in progress`.

## Polling Last Operation

When a broker returns status code `202 ACCEPTED` for provision, update or deprovision, the platform will begin polling the `/v2/service_instances/:guid/last_operation` endpoint to obtain the state of the last requested operation. The broker response must contain the field `state` and an optional field `description`.

Valid values for `state` are `in progress`, `succeeded`, and `failed`. The platform will poll the `last_operation` endpoint as long as the broker returns `"state": "in progress"`. Returning `"state": "succeeded"` or `"state": "failed"` will cause the platform to cease polling. The value provided for `description` will be passed through to the platform API client and can be used to provide additional detail for users about the progress of the operation.


### Polling Interval and Duration

The frequency and maximum duration of polling may vary by platform client. If a platform has a max polling duration and this limit is reached, the platform will cease polling and the operation state will be considered `failed`.

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
