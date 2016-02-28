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
		1. [Adding new organization](#api/organization/add)
		2. [Changing organization related data](#api/organization/change)
		3. [Transfering organization ownership](#api/organization/transfer-ownership)
		4. [Assigning users to organization](#api/organization/assign-user)
		6. [Changing user access to organization](#api/organization/access)
		5. [Removing organization](#api/organization/remove)
	4. [User accounts](#api/users)
		1. [Adding new user to collection service](#api/users/add)
		2. [Changing user information](api/users/change)
		3. [Changing user password](#api/users/set-password)
		4. [Removing user from the system](api/users/remove)
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

For example, endpoint for storing new phone call would be `/v1/json/organization` while using `POST` method. Call information is transfered as request body in form of JSON object. Some objects support additional operations in their endpoint in format `/v1/json/<object>/<action>`.


#### <a name="api/endpoints/response-codes">Response codes</a>

All service operations return standard HTTP `200` code on successful execution and `500` when error occurs with exception to `403` on failed authentication and `501` when endpoint path is malformed. Message following the code will give more insight in what kind of error occurred.


#### <a name="api/endpoints/example-request">Example request</a>

The following is a complete service request with headers and authentication:

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


### <a name="api/organization">Organizations</a>

Working with organizations is done through `/v1/json/organization` endpoint. Organizations are entities used to organize and group data collection and users. They do not necessarily represent legal entities. System allows for user interface to be customized with organization colors and logo image.

System recognizes the following actions, along with specified methods:

- [Adding new organization - `POST /v1/json/organization`](#api/organization/add)

<a name="api/organization/data-structure">Data structure:</a>

- `id` - Unique id of organization;
- `name` - Full name of the organization;
- `web_site` - URL to the company site;
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
