Accounting Service API (v1.0)
=============================

Table of contents:

1. [Data formats](#data-formats)
3. [Public API](#api)
	1. [Authentication](#api/authentication)
	2. [End points](#api/endpoints)
		1. [Response codes](#api/endpoints/response-codes)
		2. [Example request](#api/endpoints/example-request)
	3. [Organizations](#api/organization)
		1. [Retrieving organization information](#api/organization/get)
		2. [Adding new organization](#api/organization/add)
		3. [Changing organization related data](#api/organization/change)
		4. [Removing organization](#api/organization/remove)
			- [Scheduling organization removal](#api/organization/remove-schedule)
			- [Canceling organization removal](#api/organization/remove-cancel)
		5. [Listing users with access to organization](#api/organization/users)
		6. [Assigning users to organization](#api/organization/assign-user)
		7. [Changing user access to organization](#api/organization/change-access)
		8. [Removing users from organization](#api/organization/remove-user)
	4. [User accounts](#api/users)
		1. [Getting user information](#api/users/get)
		2. [Getting list of user accounts](#api/users/get-list)
		3. [Creating new user account](#api/users/add)
		4. [Verifying user account](#api/users/verify)
		5. [Changing user information](#api/users/change)
		6. [Changing user password](#api/users/set-password)
		7. [Removing user from the system](#api/users/remove)
	5. [Collection services](#api/collection-services)
		1. [Getting collection service information](#api/collection-services/get)
		2. [Creating new collection service](#api/collection-services/add)
		3. [Changing service details](#api/collection-services/change)
		4. [Removing collection service](#api/collection-services/remove)
		5. [Restarting and stopping service](#api/collection-services/status/change)
		6. [Listing users associated with service](#api/collection-services/get-user)
		7. [Adding service user](#api/collection-services/add-user)
		8. [Changing user data](#api/collection-services/change-user)
		9. [Removing user from the service](#api/collection-services/remove-user)
4. [Internal communication](#ipc)


## <a name="data-formats">Data formats</a>

_Serval_ accounting service uses [Protocol Buffers][protobuff] for internal communication with controller service and collection service. Communication with public API is done with JSON objects through standard HTTP(S) protocol. All data is stored and used in UTF32 encoding.

When communicating with public API through `POST` methods, both request and response body is in JSON form. `GET` and `DELETE` use standard NVP query strings for request, response body is then returned in JSON format. _All NVP parameter values must be URL encoded._

Service can be extended to accept different formats through different protocols by adapting existing or creating new service extensions.

For internal communication [Salsa20][salsa20] encryption algorithm is used. Connections are stream-based and can be used for multiple requests. Upon opening new connection client must send 32 bytes long initialization vector along with client id. This IV will be used in encryption process for the currently active session. Keys used in encryption are never exchanged through the network and are considered known at the time of encryption.

[protobuff]: https://developers.google.com/protocol-buffers
[salsa20]: https://en.wikipedia.org/wiki/Salsa20


## <a name="api">Public API</a>

Public API is primarily used by the Serval web application. However this API can be used to automate integration tasks and other use cases.

It is important to note that accounting service is separate from other services to ensure robustness of the infrastructure. Accounting service is designed to be used as a proxy to other services and to make operations on them easier. To retrieve actual data collected client has to connect to API of other services and directly request data from them.

Internal communication between services is done through secure and encrypted protocol.


### <a name="api/authentication">Authentication</a>

For service to accept data, client must provide identifying information. To avoid integration complexity authentication data is sent per-request. Service does not support any sort of session based authentication.

Authentication for web application is done through basic HTTP authentication. Authentication is done by specifying `Authorization` header with value set to `Basic <code>`. Where code is BASE64 encoded pair of `username:password`.

Example:

	Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQK

For internal communication authentication is not required as keys are considered private and trusted. Identifying information for client is exchanged in first encrypted message.


### <a name="api/endpoints">End points</a>

Structure of API endpoint path for version 1.0 follows strict format (`/<version>/json/<object>`).  Objects are always included as a part of path and are case sensitive. They represent object of operation while request method (`GET`, `POST`, `DELETE`) specifies operation itself.

For example, endpoint for creating new organization would be `/v1/json/organization` while using `POST` method. Organization information is transferred as request body in form of JSON object. Some objects support additional operations in their endpoint in format `/v1/json/<object>/<action>`.


#### <a name="api/endpoints/response-codes">Response codes</a>

All service operations return standard HTTP `200` code on successful execution and `500` when error occurs with exception to `403` on failed authentication and `501` when endpoint path is malformed. Message following the code will give more insight in what kind of error occurred.


#### <a name="api/endpoints/example-request">Example request</a>

The following is a complete service request with headers and authentication:

```
POST /v1/json/organization HTTP/1.1
Host: ?????
Content-Type: application/json; charset=UTF-8
Content-Length: ???
Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQK
Connection: close

{
	"name": "Weirdo Inc.",
	"web_site": "http://something.com"
}
```

### <a name="api/organization">Organizations</a>

Working with organizations is done through `/v1/json/organization`, `/v1/json/organization-remove` and `/v1/json/organization-access` endpoints. Organizations are entities used to organize and group data collection and users. They do not necessarily represent legal entities. System allows for user interface to be customized with organization colors and logo image.

Business plans are applied and billed per organization.

System recognizes the following actions, along with specified methods:

- [Retrieving organization information - `GET /v1/json/organization`](#api/organization/get)
- [Adding new organization - `POST /v1/json/organization`](#api/organization/add)
- [Changing organization related data - `PATCH /v1/json/organization`](#api/organization/change)
- [Removing organization - `DELETE /v1/json/organization`](#api/organization/remove)
	- [Scheduling organization removal - `POST /v1/json/organization-remove`](#api/organization/remove-schedule)
	- [Canceling organization removal - `DELETE /v1/json/organization-remove`](#api/organization/remove-cancel)
- [Listing users with access to organization - `GET /v1/json/organization-access`](#api/organization/users)
- [Assigning users to organization - `POST /v1/json/organization-access`](#api/organization/assign-user)
- [Changing user access to organization - `PATCH /v1/json/organization-access`](#api/organization/change-access)
- [Removing users from organization - `DELETE /v1/json/organization-access`](#api/organization/remove-user)

<a name="api/organization/data-structure">Data structure:</a>

- `id` - Unique id of organization;
- `name` - Full name of the organization;
- `web_site` - URL to the company site without protocol;
- `phone_number` - Contact phone number;
- `address` - Office street address;
- `city` - City where offices are located;
- `zip` - ZIP code for city where offices are located;
- `state` - State where organization is located;
- `country` - Country where organization is located;
- `colors` - Coma separated colors used for stylizing user interface;
- `logo_url` - URL to organization logo.

All data, except for `id`, can be changed.

<a name="api/organization/access-data-structure">Access list data structure:</a>

- `organization` - Unique id of the organization;
- `service` - Unique id of service;
- `user` - Unique user id;
- `type` - Type of access.

Either of the two (`organization` and `service`) can be specified for access. If user is assigned access to `organization` alone it will have access to all services on the system. Specifying `service` limits access to that service alone. When configuring user can be given access only to services and organization currently logged in user has `owner` access.

System recognizes the following types of access:

- `agent` - Minimal level of access to your organization and services. Can only post data to specified services;
- `user` - Regular access to data without ability to change system configuration;
- `owner` - Maximum level acces. Allows configuration of organization and services.


#### <a name="api/organization/add">Adding new organization</a>

This endpoint is used to create new organization associated with currently logged in user account. Response contains data for the newly created organization.

Method: `POST`  
Endpoint: `/v1/json/organization`

Request body:

```json
{
	"name": "Company Inc.",
	"web_site": "somesite.com",
	"phone_number": "000-CALL-NOW",
	"address": "Bruce Partington rd. 221b",
	"city": "London",
	"zip": "1000",
	"state": "",
	"country": "UK",
	"colors": "#330033,white,#ff00ff",
	"logo_url": "somesite.com/images/logo.png"
}
```

Response body:

```json
{
	"id": 213,
	"result": true
}
```


#### <a name="api/organization/change">Changing organization related data</a>

This endpoint is used to change specified organization data. Only [fields](#api/organization/data-structure) specified in request will be modified. Fields which can't be modified will be silently ignored.

Response body contains `id` of affected organization and `result` of operation. If operation fails, `id` will be set to 0.

Method: `PATCH`  
Endpoint: `/v1/json/organization`

Request body:

```json
{
	"id": 213,
	"name": "New Name Inc."
}
```

Response body:

```json
{
	"id": 213,
	"result": true
}
```

#### <a name="api/organization/remove">Removing organization</a>

Removes organization and all of its services permanently. User accounts associated with the organization will not be affected. Prior to calling this endpoint organization must be [scheduled for removal](#api/organization/remove-schedule). After one month since organization was marked for removal calling this endpoint will cause all the data associated with organization to be **permanently** removed.

During waiting period services assigned to organization will continue to operate normally and can be transferred to other organizations. Owners can [cancel removal](#api/organization/remove-cancel) process at any time.

If this request is made before waiting period is over, system will return `false` and a date when after which organization is no longer protected from removal.


Method: `DELETE`  
Endpoint: `/v1/json/organization`

Request parameters:

```
organization=213
```

Response body:

```json
{
	"scheduled": "2016-10-10",
	"result": true
}
```


##### <a name="api/organization/remove-schedule">Scheduling organization removal</a>

Schedules organization and its services for removal. Before any organization can be removed there's a mandatory waiting period of one month. This is to prevent accidental removals and malicious intents. Scheduled removals [can be canceled](#api/organization/remove-cancel) by any account with _owner_ access.

If this request is made and organization is already scheduled for removal no changes to initial date are made. Response will contain `true` and original date of removal.


Method: `POST`  
Endpoint: `/v1/json/organization-remove`

Request body:

```json
{
	"organization": 213
}
```

Response body:

```json
{
	"scheduled": "2016-10-10",
	"result": true
}
```


##### <a name="api/organization/remove-cancel">Canceling organization removal</a>

Cancel removal of organization and its services. Organization removal can be canceled by any account with _owner_ access before removal is confirmed. If organization and its services were removed by the system they can **not** be recovered.

Return value for this call denotes if cancellation was successful.

Method: `DELETE`  
Endpoint: `/v1/json/organization-remove`

```
organization=213
```

Response body:

```json
{
	"scheduled": "2016-10-10",
	"result": true
}
```


#### <a name="api/organization/users">Listing users with access to organization</a>

Get list of all users with access to `organization` specified in request parameter. This endpoint is only available to users which have ability to modify user access in organization.

Method: `GET`  
Endpoint: `/v1/json/organization-access`

Request parameters:

```
organization=213
```

Response body:

```json
{
	"users": [11, 1450, 234],
	"result": true
}
```


#### <a name="api/organization/assign-user">Assigning users to organization</a>

Endpoint used to allow existing users to access organization and its data with specified restrictions. This endpoint can not be used to create new user account.

Response object contains `result` of the operation.

Method: `POST`  
Endpoint: `/v1/json/organization-access`

Request body:

```json
{
	"organization": 213,
	"user": 12,
	"access": 0
}
```

Response body:

```json
{
	"result": true
}
```


#### <a name="api/organization/change-access">Changing user access to organization</a>

Change access for user in organization. This endpoint allows you to promote or demote users withing organization. It is also used in ownership transfers. It's important to note that organization can have more than one owner account and in fact an advisable situation to be in as it will make sure [bus factor](https://en.wikipedia.org/wiki/Bus_factor) is higher. When configuring user can be given access only to services and organization currently logged in user has `owner` access.

Response object contains `result` of the operation.

Method: `PATCH`  
Endpoint: `/v1/json/organization-access`

Request body:

```json
{
	"organization": 213,
	"user": 12,
	"access": 0
}
```

Response body:

```json
{
	"result": true
}
```


#### <a name="api/organization/remove-user">Removing users from organization</a>

Remove user from the organization. This operation does not affect data associated with the user.

Response object contains `result` of the operation.

Method: `DELETE`  
Endpoint: `/v1/json/organization-access`

Request body:

```json
{
	"organization": 213,
	"user": 12
}
```

Response body:

```json
{
	"result": true
}
```


### <a name="api/users">User accounts</a>

Working with user accounts is done through `/v1/json/users` and `/v1/json/user-session` endpoints. Account can be used with multiple organizations with different access levels to each.

System recognizes the following actions, along with specified methods:
- [Getting user information - `GET /v1/json/users`](#api/users/get)
- [Getting list of user accounts - `GET /v1/json/users/all](#api/users/get-list)
- [Creating new user account - `POST /v1/json/users`](#api/users/add)
- [Verifying user account - `POST|GET /v1/json/users/verify`](#api/users/verify)
- [Changing user information - `PATCH /v1/json/users`](#api/users/change)
- [Changing user password - `PATCH /v1/json/users/password`](#api/users/set-password)
- [Removing user from the system - `DELETE /v1/json/users`](#api/users/remove)

<a name="api/users/data-structure">Data structure:</a>

- `id` - Unique id of user;
- `first_name` - First name;
- `last_name` - Last name;
- `phone_number` - Contact phone number;
- `email` - Email address;
- `username` - Username used to log in to system;
- `password` - Hashed and salted password (not available to public);
- `salt` - Salt used for password hash (not available to public);
- `verified` - State of account;
- `oauth_service` - OAuth service used (not available to public);
- `oauth_token` - Token for OAuth service (not available to public);
- `profile_image` - URL to the profile image.

All fields except `id`, `password`, `salt`, `oauth_service`, `oauth_token` and `verified` can be directly changed.


#### <a name="api/users/get">Getting user information</a>

This endpoint is called to retrieve information associated with authenticated user account. Web application makes this request in order to check if entered username and password are correct.

Method: `GET`  
Endpoint: `/v1/json/users`

Request parameters:
_There are no request parameters supported by this endpoint._

Response body:

```json
{
	"id": 213,
	"first_name": "John",
	"last_name": "Doe",
	"phone_number": "",
	"email": "",
	"username": "jon-doe",
	"verified": false,
	"profile_image": "http://somesite.com/profile-image.jpg"
}
```


#### <a name="api/users/get-list">Getting list of user accounts</a>

Returns list of all user accounts matching information provided. This endpoint is used to list accounts associated with organizations or collection service.

Request body contains elements of the user account to match with extra `organization` or `service` id parameter to limit result list only to accounts associated with specified entity. System matches names approximately.

Response is a list of JSON objects containing all the matched user accounts with excluded private information (`username`, `verified`, `password`, `salt`, `oauth_service`, `oauth_token`, `email`, `phone_number`) in order to protect privacy of users.

Method: `GET`  
Endpoint: `/v1/json/users/all`

Request parameters:

```
name=Joe&
organization=12
```

Response body:

```json
[
	{
		"id": 213,
		"first_name": "Joe",
		"last_name": "Remul",
		"profile_image": "http://somesite.com/profile-image.jpg"
	},
	{
		"id": 2,
		"first_name": "Joen",
		"last_name": "",
		"profile_image": "http://someothersite.com/profile-image.jpg"
	}
]
```


#### <a name="api/users/add">Creating new user account</a>

Creates new user account with specified data. Account is not usable before verification, therefore response will only contain `id` of the newly created entry. In order to create account email, username and password must be provided. In case either of these fields is empty error is raised.

Once new account is created verification code is generated and sent to provided email. Account is not usable until email address is verified.

Method: `POST`  
Endpoint: `/v1/json/users`

Request body:

```json
{
	"first_name": "John",
	"last_name": "Doe",
	"email": "someone@site.com",
	"password": "dadada"
}
```

Response body:

```json
{
	"id": 213,
	"result": true
}
```


#### <a name="api/users/verify">Verifying user account</a>

Before account can be used email address needs to be verified. Upon creating new account system will send an email with verification code and URL. Unique code is randomly generated id and associated with specific account.

This endpoint allows both `POST` and `GET` methods to be used to allow users to verify account simply by clicking on a URL provided in verification email. This also means parameters can be provided as JSON object or as request parameters.

Response is always in form of JSON object containing `result` of the verification process. Upon successful verification code is removed from the account.

Method: `POST` or `GET`  
Endpoint: `/v1/json/users/verify`

Request body:

```
id=123&code=123457890ABCDEF
```

Or:

```json
{
	"id": 123,
	"code": "123457890ABCDEF"
}
```

Response body:
```json
{
	"result": true
}
```


#### <a name="api/users/change">Changing user information</a>

Used to change currently authenticated user account information. Only data specified in request body will be changed. Data which are read-only or nonexistent keys will be silently ignored.

Response body contains object with complete user account data.

Method: `PATCH`  
Endpoint: `/v1/json/users`

Request body:
```json
{
	"first_name": "Marco",
	"last_name": "Polo"
}
```

Response body:

```json
{
	"id": 213,
	"first_name": "Marco",
	"last_name": "Polo",
	"phone_number": "",
	"email": "",
	"username": "jon-doe",
	"verified": false,
	"profile_image": "http://somesite.com/profile-image.jpg"
}
```

#### <a name="api/users/set-password">Changing user password</a>

Change currently logged in user password. This endpoint expects JSON object with single property containing new password for the currently logged in user. Password change is immediate and following requests need to use it in order to get response from the server.

Response object contains single `result` boolean value denoting success of the operation.

Method: `PATCH`  
Endpoint: `/v1/json/users/password`

Request body:

```json
{
	"password": "tickle tickle little star"
}
```

Response body:

```json
{
	"result": true
}
```


#### <a name="api/users/remove">Removing user from the system</a>

Perform removal of currently logged in user from the Serval system. User account can not be removed if it is a single owner in any organization. In such cases another user account needs to be given owner permissions on organization or remove organization before removing user account. This endpoint doesn't require any additional parameters.

Response body contains single boolean value denoting success of the operation. Upon removal user account is immediately rendered unusable.

Method: `DELETE`  
Endpoint: `/v1/json/users`

Response body:

```json
{
	"result": true
}
```


### <a name="api/colleciton-services">Collection services</a>

Working with collection services is done through `/v1/json/collection-service` and `/v1/json/collection-service/user` endpoints. These services are physically separate locations where you data is stored. They can be spread apart on different servers and even different countries. This approach allows for better redundancy, uptime and ease of maintenance.

Accounting server, and with it this API, allows for controlling service status, user access and creating of new services. Working with data stored on the service requires connecting to the service itself. Connection information is provided with service listing.

Service status is sent by the service periodically to simplify operations on large scale.

System recognizes the following actions, along with specified methods:
- [Getting collection service information - `GET /v1/json/collection-service`](#api/collection-services/get)
- [Creating new collection service - `POST /v1/json/collection-service`](#api/collection-services/add)
- [Changing service details - `PATCH /v1/json/collection-service`](#api/collection-services/change)
- [Removing collection service - `DELETE /v1/json/collection-service`](#api/collection-services/remove)
- [Restarting and stopping service - `PATCH /v1/json/collection-service/status`](#api/collection-services/status)
- [Listing users associated with service - `GET /v1/json/collection-service/user`](#api/collection-services/get-user)
- [Adding service user - `POST /v1/json/collection-service/user`](#api/collection-services/add-user)
- [Changing user data - `PATCH /v1/json/collection-service/user`](#api/collection-services/change-user)
- [Removing user from the service - `DELETE /v1/json/collection-service/user`](#api/collection-services/remove-user)

<a name="api/collection-services/data-structure">Data structure:</a>

- `id` - Unique id of collection service in accounting;
- `name` - Custom name for easier tracking;
- `status` - Last reported status of a service;
- `status_timestamp` - Time when status was last reported;
- `organization` - Organization service belongs to;
- `address_v4` - IPv4 address where service is located;
- `address_v6` - IPv6 address where service is located;
- `controller` - Controller service in charge.

Only `name` and `organization` fields can be changed through API.


#### <a name="api/collection-services/get">Getting collection service information</a>

Retrieve information about specified collection service. Only services currently logged in user has access to can be retrieved. If no service `id` is specified all services user has access to will be returned.

Response body is a list containing JSON objects representing collection services.

Status attribute can have the following values:

- `running` - service is running and accepting new data;
- `stopped` - service is in stopped mode and is only accepting IPC requests;
- `disabled` - service is disabled due to maintenance or some accounting error.

Method: `GET`  
Endpoint: `/v1/json/collection-service`

Request parameters: 
```
id=11
```

Response body:
```json
[
	{
		"id": "11",
		"name": "Blog",
		"status": "running",
		"status_timestamp": 1467902960,
		"organization": 213,
		"address_v4": "192.168.0.1",
		"address_v6": "2001:0db8:0000:0042:0000:8a2e:0370:7334",
		"controller": 15
	}
]
```


#### <a name="api/collection-services/add">Creating new collection service</a>

Schedule creation of new collection service with specified `name`. Optionally desired `country` where service will be hosted can be requested. If user has _owner_ access to more than one organization additional parameter (`organization`) can be submitted. Newly created collection service will be assigned to either organization provided as a parameter or to organization currently logged in user has _owner_ access to. In cases where user is owner in multiple organizations and parameter is not specified to denote which organization new service belongs to system will avoid creating the service to avoid bad assignment.

Method: `POST`  
Endpoint: `/v1/json/collection-service`

Request body:
```json
{
	"name": "Blog",
	"organization": 11,
	"country": "de"
}
```

Response body:
```json
{
	"result": true,
	"id": 1000,
	"name": "Blog",
	"organization": 11
```


#### <a name="api/collection-services/change">Changing service details</a>

Change collection service information. This endpoint can only be used to change some of the information. Changing service status, for example, is done through separate endpoint.

Response is in form of JSON object containing `result` of the operation and _changed_ collection service data.

Method: `PATCH`  
Endpoint: `/v1/json/collection-service`

Request body:
```json
{
	"id": 11,
	"name": "Blog"
}
```

Response body:
```json
{
	"result": true,
	"id": 1000,
	"name": "Blog",
	"organization": 11
```


#### <a name="api/collection-services/remove">Removing collection service</a>

Schedule collection service removal with controller in charge. As opposed to organization, collection services do not require waiting period and can be scheduled for removal immediately. There is usually very limited waiting period as service removal is performed by controller. Calling this end point with service `id` as a parameter will tell controller to stop the service and perform removal of service and its backups.

Response body contains JSON object with only one attribute (`result`) denoting outcome of operation.

Method: `DELETE`  
Endpoint: `/v1/json/collection-service`

Request parameters:
```
id=11
```

Response body:
```json
{
	"result": true
}
```


#### <a name="api/collection-services/status/change">Restarting and stopping service</a>

This end point allows control of collection service status. It will allow user to stop, restart and resume operations of any collection service belonging to organization user has _owner_ access. Request object must contain service `id` and desired `status` of the service. Value of `status` attribute is not the same as value returned by getting service status. These values are intermediate and only serve to set desired status which other methods return.

Response body contains JSON object with `result` of the operation and target service `status`. If process of setting new status fails `status` attribute will contain last know status of the service.

Recognized intermediate status values:

- `start`;
- `restart`;
- `stop`.

Method: `PATCH`  
Endpoint: `/v1/json/collection-service/status`

Request body:
```json
{
	"id": 11,
	"status": "stopped"
}
```

Response body:
```json
{
	"result": true,
	"status": "stopped"
}
```


#### <a name="api/collection-services/get-user">Listing users associated with service</a>

Returns a list of credentials which have access to specified collection service. These user accounts are not related to user accounts which have access to web application. By design access to collection service is limited per-service and in certain way. This provides better security and easier integration.

Collection service credentials are only available to accounts with _owner_ access to organization service belongs to. Response body contains list of JSON objects. In cases there are no credentials associated with collection service empty list is returned.

Method: `GET`  
Endpoint: `/v1/json/collection-service/user`

Request parameters:
```
service=212
```

Response body:
```json
[
	{
		"id": 1,
		"name": "Web JavaScript",
		"uid": "123ffee123",
		"last_access": 1467902960,
		"type": 0
	}
]
```

