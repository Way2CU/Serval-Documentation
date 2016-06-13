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
		1. [Creating new user account](#api/users/add)
		2. [Verifying user account](#api/users/verify)
		3. [Changing user information](api/users/change)
		4. [Changing user password](#api/users/set-password)
		5. [Removing user from the system](api/users/remove)
		6. [Logging user in/starting a new session](#api/users/login)
		7. [Logging user out/closting existing session](#api/users/logout)
	5. [Collection services](#api/collection-services)
		1. [Creating new collection service](#api/collection-services/add)
		2. [Changing service details](#api/collection-services/change)
		3. [Restarting and stopping service](#api/collection-services/status)
		4. [Removing collection service](#api/collection-services/remove)
		5. [Adding service user](#api/collection-services/add-user)
		6. [Changing user data](#api/collection-services/change-user)
		7. [Removing user from the service](#api/collection-services/remove-user)
4. [Internal communication](#ipc)


## <a name="data-formats">Data formats</a>

_Serval_ accounting service uses [Protocol Buffers][protobuff] for internal communication with puppeteer service and collection service. Communication with public API is done with JSON objects through standard HTTP(S) protocol. All data is stored and used in UTF32 encoding.

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

Request body:

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

Request body:

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

Change access for user in organization. This endpoint allows you to promote or demote users withing organization. It is also used in ownership transfers. It's important to note that organization can have more than one owner account and in fact an advisable situation to be in as it will make sure [bus factor](https://en.wikipedia.org/wiki/Bus_factor) is higher.

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
- `oauth_token` - Token for OAuth service (not available to public).

All fields except `id`, `password`, `salt` and `verified` can be changed.


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
