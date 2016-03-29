Collection Service API (v1.0)
=============================

Table of contents:

1. [Data formats](#data-formats)
2. [Authentication](#authentication)
3. [Endpoints](#endpoints)
	1. [Return codes](#endpoints/response-codes)
	2. [Example request](#endpoints/example-request)
4. [Leads](#leads)
	1. [Submitting new lead](#leads/submit)
	2. [Getting list of leads](#leads/get)
	3. [Removing lead and its data](#leads/delete)
	4. [Getting all data associated with lead](#leads/get_data)
5. [Call information](#calls)
	1. [Submitting new phone call](#calls/submit)
	2. [Getting list of phone calls](#calls/get)
	3. [Removing phone call](#calls/delete)
6. [Contact forms](#forms)
	1. [Submitting new contact form](#forms/submit)
	2. [Getting list of contact forms](#forms/get)
	3. [Removing contact form](#forms/delete)


## <a name="data-formats">Data formats</a>

_Serval_ accounting service uses [Protocol Buffers][protobuff] for internal communication with puppeteer service and collection service. Communication with public API is done with JSON objects through standard HTTP(S) protocol. All data is stored and used in UTF32 encoding.

For `POST` methods, both request and response body is in JSON format. `GET` and `DELETE` use standard NVP query strings for request, response body is then returned in JSON format. _All NVP parameter values must be URL encoded._

Service can be extended to accept different formats through different protocols by adapting existing or creating new service extensions.

For internal communication [Salsa20][salsa20] encryption algorithm is used. Connections are stream-based and can be used for multiple requests. Upon opening new connection client must send 32 bytes long initialization vector along with client id. This IV will be used in encryption process for the currently active session. Keys used in encryption are never exchanged through the network and are considered known at the time of encryption.


[protobuff]: https://developers.google.com/protocol-buffers
[salsa20]: https://en.wikipedia.org/wiki/Salsa20


## <a name="authentication">Authentication</a>

For service to accept data, client must provide identifying information. To avoid integration complexity authentication data is sent per-request. Service does not support any sort of session based authentication.

Authentication is done through basic HTTP authentication. Authentication is done by specifying `Authorization` header with value set to `Basic <code>`. Where code is BASE64 encoded pair of `username:password`.

Example:

	Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQK

For internal communication authentication is not required as keys are considered private and trusted. Identifying information for client is exchanged in first encrypted message.


## <a name="endpoints">End points</a>

Structure of API endpoint path for version 1.0 follows strict format (`/<version>/json/<object>`).  Objects are always included as a part of path and are case sensitive. They represent object of operation while request method (`GET`, `POST`, `DELETE`) specifies operation itself.

For example, endpoint for storing new phone call would be `/v1/json/calls` while using `POST` method. Call information is transfered as request body in form of JSON object. Some objects support additional operations in their endpoint in format `/v1/json/<object>/<action>`.


### <a name="endpoints/response-codes">Response codes</a>

All service operations return standard HTTP `200` code on successful execution and `500` when error occurs with exception to `403` on failed authentication and `501` when endpoint path is malformed. Message following the code will give more insight in what kind of error occurred.


### <a name="endpoints/example-request">Example request</a>

The following is a complete service request with headers and authentication:

```
POST /v1/json/leads HTTP/1.1
Host: ?????
Content-Type: application/json; charset=UTF-8
Content-Length: ???
Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQK
Connection: close

{
	"name": "Joe Danger",
	"status": 0
}
```


## <a name="leads">Leads</a>

Working with leads is done through `/v1/json/leads` endpoint. While system supports directly storing leads it will try to automatically assign other data types (calls, contact forms, etc.) with a lead.

System recognizes the following actions, along with specified methods:

- [Submitting new lead - `POST /v1/json/leads`](#leads/submit)
- [Getting list of leads - `GET /v1/json/leads`](#leads/get)
- [Removing lead and its data - `DELETE /v1/json/leads`](#leads/delete)
- [Getting all data associated with lead - `GET /v1/json/leads/data`](#leads/get_data)

<a name="leads/data-structure">Data structure:</a>

- `id` - Unique id of lead;
- `name` - Full name of the lead;
- `status` - Numerical representation of lead status;
- `initial_source` - Origin of first submission (client id);
- `timestamp` - Date and time of first submission in format `YYYY-MM-DD HH:MM:SS`.

Only the following data can be changed: `name`, `status`.


### <a name="leads/submit">Submitting new lead</a>

This endpoint is used to submit new lead or update existing information. By specifying lead `id` in your request body data will be modified instead of inserted. Only [specific fields](#leads/data-structure) can be modified. Others will be ignored.

Method: `POST`  
Endpoint: `/v1/json/leads`

Request body:

```json
{
	"name": "Joe Danger",
	"status": 0
}
```

Response body:

```json
{
	"id": 0,
	"name": "Joe Danger",
	"status": 0,
	"initial_source": 0,
	"timestamp": "2014-12-31 16:00:04"
}
```


### <a name="leads/get">Getting list of leads</a>

This endpoint gives a list of one or more leads. If lead `id` is specified as a single number or comma separated list of numbers, only those leads will be matched against other parameters. In case requested parameters didn't match any leads empty list is returned. All fields in request query string are optional. Omitting all of them will return all leads.

Method: `GET`  
Endpoint: `/v1/json/leads`

Additional parameters:

- `from` - Starting timestamp. If time is omitted it defaults to first second of the day;
- `to` - Ending timestamp. If time is omitted it defaults to last second of the day.

Query string:

```
id=0,2
name=Joe*
status=0
initial_source=0
from=2014-01-30 20:00:00
to=2016-02-10 10:00:00
```

Response body:

```json
[
	{
		"id": 0,
		"name": "Joe Danger",
		"status": 0,
		"initial_source": 0,
		"timestamp": "2014-12-31 16:00:04"
	},
	{
		"id": 2,
		"name": "Joel Dagger",
		"status": 1,
		"initial_source": 0,
		"timestamp": "2014-12-31 16:00:04"
	}
]
```


### <a name="leads/delete">Removing lead and its data</a>

This endpoint will mark specified lead as removed. System doesn't not support permanent removal of data in order to prevent undesired behavior and removal by mistake. This API endpoint only accepts single lead id or comma separated list of ids to be removed. Response body is always an empty JSON object. Result of data removal is returned through response code and message.

Method: `DELETE`  
Endpoint: `/v1/json/leads`

Query string:

	id=0,2

Response body:

	{}


### <a name="#leads/get_data">Getting all data associated with lead</a>


## <a name="calls">Call information</a>

Working with phone calls is done through `/v1/json/calls` endpoint. Phone calls can not be deleted only marked as such. Deleted phone calls are excluded from reports and statistics.

System recognizes the following actions, along with specified methods:

- [Submitting new phone call - `POST /v1/json/calls`](#calls/submit)
- [Getting list of phone calls - `GET /v1/json/calls`](#calls/get)
- [Removing phone call - `DELETE /v1/json/calls`](#calls/delete)

<a name="calls/data-structure">Data structure:</a>

- `id` - Unique id of phone call;
- `lead` - Unique id of associated lead;
- `source` - Origin of submission (client id);
- `audio_url` - URL of audio file;
- `duration` - Total duration of phone call in seconds;
- `ring_time` - Time before phone was picked up in seconds;
- `talk_time` - Total time spent in conversation in seconds;
- `caller_number` - Phone number of caller;
- `receiving_number` - Phone number that received the call;
- `tags` - Comma-separated list of custom tags;
- `timestamp` - Date and time of first submission in format `YYYY-MM-DD HH:MM:SS`;
- `status` - Integer denoting status of phone call.

Only the following data can be changed: `tags`, `status`, `lead`.


### <a name="calls/submit">Submitting new phone call</a>

This endpoint is used to submit new phone call. By specifying call `id` in your request body data will be modified instead of inserted. Only [specific fields](#calls/data-structure) can be posted. Others will be ignored.

Method: `POST`  
Endpoint: `/v1/json/calls`

Additional parameters:

- `lead` - Id of lead to associate phone call with;
- `name` - Caller name used to automatically associate with existing lead;
- `email` - Caller email used to automatically associate with existing lead.

Request body:

```json
{
	"audio_url": "http://somesite.com/data/something.mp3",
	"duration": 100,
	"ring_time": 5,
	"talk_time": 95,
	"caller_number": "+1-955-555-1234",
	"receiving_number": "+1-955-555-000",
	"tags": "ctm, google, sale"
}
```

Response body:

```json
{
	"id": 0,
	"lead": 1,
	"source": 2
}
```


### <a name="#calls/get">Getting list of phone calls</a>

This endpoint gives a list of one or more phone calls. If call `id` is specified as a single number or comma separated list of numbers, only those calls will be matched against other parameters. In case requested parameters didn't match any leads empty list is returned. All fields in request query string are optional. Omitting all of them will return all leads. If not otherwise specified API will return only calls that are not marked as deleted.

Method: `GET`  
Endpoint: `/v1/json/calls`

Additional parameters:

- `from` - Starting timestamp. If time is omitted it defaults to first second of the day;
- `to` - Ending timestamp. If time is omitted it defaults to last second of the day;
- `lead` - Lead id to associate data with.

Query string:

	id=1,2
	from=2014-01-30 20:00:00
	to=2016-02-10 10:00:00

Response body:

```json
[
	{
		"id": 1,
		"lead": 1,
		"source": 5,
		"audio_url": "http://somesite.com/data/something.mp3",
		"duration": 100,
		"ring_time": 10,
		"talk_time": 90,
		"caller_number": "+1-555-555",
		"receiving_number": "+1-555-1234",
		"tags": "sale,solicitor",
		"timestamp": "2014-12-31 16:00:04",
		"status": 0
	},
	{
		"id": 2,
		"lead": 1,
		"source": 5,
		"audio_url": "http://somesite.com/data/something.mp3",
		"duration": 100,
		"ring_time": 10,
		"talk_time": 90,
		"caller_number": "+1-555-555",
		"receiving_number": "+1-555-1234",
		"tags": "sale,solicitor",
		"timestamp": "2014-12-31 16:00:04",
		"status": 0
	}
]
```


### <a name="#calls/delete">Removing phone call</a>

This endpoint will mark specified phone call as removed. System doesn't not support permanent removal of data in order to prevent undesired behavior and removal by mistake. This API endpoint only accepts single call id or comma separated list of ids to be removed. Response body is always an empty JSON object. Result of data removal is returned through response code and message.

Method: `DELETE`  
Endpoint: `/v1/json/calls`

Query string:

	id=0,2

Response body:

	{}


## <a name="forms">Contact forms</a>

Working with contact forms is done through `/v1/json/forms` endpoint. Contact form data can not be deleted only marked as such. Deleted contact form data is excluded from reports and statistics.

System recognizes the following actions, along with specified methods:

- [Submitting new contact form - `POST /v1/json/forms`](#forms/submit)
- [Getting list of contact forms - `GET /v1/json/forms`](#forms/get)
- [Removing contact form submission - `DELETE /v1/json/forms`](#forms/delete)

<a name="forms/data-structure">Data structure:</a>

- `id` - Unique id of phone call;
- `lead` - Unique id of associated lead;
- `source` - Origin of submission (client id);
- `email` - Contact email address;
- `phone` - Contact phone number;
- `timestamp` - Date and time of first submission in format `YYYY-MM-DD HH:MM:SS`;
- `status` - Integer value denoting status of contact form submission.

Only the following data can be changed: `status`, `lead`.


### <a name="#forms/submit">Submitting new contact form</a>

This endpoint is used to submit new contact form data. By specifying call `id` in your request body data will be modified instead of inserted. Only [specific fields](#calls/data-structure) can be posted. Others will be ignored.

Method: `POST`  
Endpoint: `/v1/json/forms`

Additional parameters:

- `lead` - Id of lead to associate form data with;
- `name` - Caller name used to automatically associate with existing lead;
- `email` - Caller email used to automatically associate with existing lead;
- `phone_number` - Phone number used to automatically associate with existing lead.

Request body:

```json
{
	"email": "joe.danger@site.com",
	"phone": "+1-955-555-1234"
}
```

Response body:

```json
{
	"id": 0,
	"lead": 1,
	"source": 2
}
```


### <a name="#forms/get">Getting list of contact forms</a>

This endpoint gives a list of one or more contact forms. If form `id` is specified as a single number or comma separated list of numbers, only those form data will be matched against other parameters. In case requested parameters didn't match any leads empty list is returned. All fields in request query string are optional. Omitting all of them will return all contact form data. If not otherwise specified API will return only data that is not marked as deleted.

Method: `GET`  
Endpoint: `/v1/json/forms`

Additional parameters:

- `from` - Starting timestamp. If time is omitted it defaults to first second of the day;
- `to` - Ending timestamp. If time is omitted it defaults to last second of the day;
- `lead` - Lead id to associate data with.

Query string:

	id=1,2
	from=2014-01-30 20:00:00
	to=2016-02-10 10:00:00

Response body:

```json
[
	{
		"id": 1,
		"lead": 1,
		"source": 5,
		"email": "joe.danger@site.com",
		"phone": "+1-955-555-1234",
		"timestamp": "2014-12-31 16:00:04",
		"status": 0
	},
	{
		"id": 2,
		"lead": 1,
		"source": 5,
		"email": "even.more.joe.dangerous@site.com",
		"phone": "+1-955-555-1234",
		"timestamp": "2014-12-31 16:00:04",
		"status": 0
	}
]
```


### <a name="#forms/delete">Removing contact form</a>

This endpoint will mark specified contact form as removed. System doesn't not support permanent removal of data in order to prevent undesired behavior and removal by mistake. This API endpoint only accepts single contact form id or comma separated list of ids to be removed. Response body is always an empty JSON object.  Result of data removal is returned through response code and message.

Method: `DELETE`  
Endpoint: `/v1/json/forms`

Query string:

	id=0,2

Response body:

	{}

