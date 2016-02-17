Accounting Service API (v1.0)
=============================

Table of contents:

1. [Data formats](#data-formats)
1. [Authentication](#authentication)


## <a name="data-formats">Data formats</a>

_Serval_ accounting service uses [Protocol Buffers][protobuff] for internal communication with puppeteer
service and collection service. Communication with public API is done with JSON objects through
standard HTTP(S) protocol. All data is stored and used in UTF32 encoding.

When communicating with public API through `POST` methods, both request and response body is in
JSON form. `GET` and `DELETE` use standard NVP query strings for request, response body is then returned
in JSON format. _All NVP parameter values must be URL encoded._

Service can be extended to accept different formats through different protocols by adapting
existing or creating new service extensions.

For internal communication [Salsa20][salsa20] encryption algorithm is used. Connections are
stream-based and can be used for multiple requests. Upon opening new connection client must
send 32 bytes long initialization vector along with client id. This IV will be used in encryption
process for the currently active session. Keys used in encryption are never exchanged through the
network and are considered known at the time of encryption.


## <a name="authentication">Authentication</a>

For service to accept data, client must provide identifying information. To avoid integration
complexity authentication data is sent per-request. Service does not support any sort of session
based authentication.

Authentication for web application is done through basic HTTP authentication. Authentication is
done by specifying `Authorization` header with value set to `Basic <code>`. Where code is BASE64
encoded pair of `username:password`.

Example:

	Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQK

For internal communication authentication is not required as keys are considered private and
trusted. Identifying information for client is exchanged in first encrypted message.


[protobuff]: https://developers.google.com/protocol-buffers
[salsa20]: https://en.wikipedia.org/wiki/Salsa20


## <a name="api">Public API</a>
### <a name="api/users">Managing users</a>
### <a name="api/users/add">Adding new user to collection service</a>
### <a name="api/users/set-password">Adding new user to collection service</a>

## <a name="ipc">Internal communication</a>
### <a name="ipc">Internal communication</a>
### <a name="ipc">Internal communication</a>
### <a name="ipc">Internal communication</a>
