# Auth Server API
The auth server is responsible both for authenticating users and for deciding
what users and services may access what content.  It is designed using
[OAuth 2.0][1], which is widely used across the internet to secure user facing
and back end services.  It is therefore necessary to reference the OAuth 2.0
specification for additional details not provided in this document.

Usage of OAuth 2.0 requires extensive use of TLS, so the auth server and all
clients should use HTTPS as the only protocol for accessing resources.


## Justification
The auth service provides the following benefits:

- Removes requirement for individual services to authenticate (authN) and
  authorize (authZ) users.
- Encapsulates security needs of services. For example, workflow does not need
  to concern itself with the security model for shell-command.


## Definitions
- API key: A unique, randomly-generated id that is created by the Auth Server
  and is used both to identify and to authenticate a user.
- user role: A label that represents a subset of the permissions available to a
  user.

Additional important terms are defined in the
[OAuth 2.0 RFC][1] and in the [OAuth 2.0 Bearer Token RFC][2].


## API Keys
A user may only have one active API Key at a time.  API keys may have different
expiration durations.  The storage and access of the user's current API key is
up to the SDK.

API keys should be transmitted in the `Authorization` header of requests:

    Authorization: API-Key 0123456789abcdef

where `0123456789abcdef` is an example API key.

## ID Tokens

ID Tokens are defined in the [OpenID Connect Core 1.0][4] specification.  ID
Tokens are supported in this specificatino with the following extensions and
limitations:

The `aud` claim must be a list of one or more values determined from the
requested scopes.  The `at_hash` claim is always included.

Additional collision-resistant claims in the UUID Version 5 namespace, 
`66deca4c-4e8a-44ce-a617-3d37bc0bcfaa`.  These additional claims may be
required depending on the audiences included in the audience list.  The
additional claims are defined as:
- `posix` (`d9294df3-f60f-504c-aabf-9f8af93cc008`)
    -  Dictionary containing
        - `username`
        - `uid` integer
        - `gid` integer
        - `groups` list of integers
- `roles` (`b15901ac-6238-5e23-8fc7-02f4d26053e6`)
    -  List of role names


Sample additional claims from an OpenID Connect `id_token`:

    "d9294df3-f60f-504c-aabf-9f8af93cc008": {
       "username": "ltorvalds",
       "uid": 10001,
       "gid": 10001,
       "groups": [13, 24],
    },

    "b15901ac-6238-5e23-8fc7-02f4d26053e6": ["foo", "bar"]

The `id_token` should be passed to services in the `Identity` header, as in
this example:

    Identity: JWT eyJ0eXAiOiJKV1QiLA0KICJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJqb2UiLA0KICJleHAiOjEzMDA4MTkzODAsDQogImh0dHA6Ly9leGFtcGxlLmNvbS9pc19yb290Ijp0cnVlfQ.dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk


## Authentication API
This part of the API is primarily specified by the [OAuth 2.0 RFC][1].  This
section exists to clarify the necessary endpoints and feature support required
for PTero.

### GET /v1/authorization
This is the "Authorization Endpoint" specified in section 3.1 of the
[OAuth 2.0 RFC][1].  Clients should redirect user agents to this endpoint to
request access tokens.

The value of the `response_type` parameter must be one of `code`, `token`, or
`id_token token`.  The value of the `prompt` parameter must be `none` when the
`prompt` parameter is provided.

If the server does not get an `Authorization` header with and API-Key value,
the server must respond with 401 and the header `WWW-Authenticate: API-Key`.
If the `Authorization` header with `API-Key` value is provided in the request,
but the API-Key is expired or otherwise invalid, the server must respond with
403.

If an ID Token is returned by this endpoint, the ID Token must be encrypted.

### POST /v1/tokens
This is the "Token Endpoint" specified in section 3.2 of the
[OAuth 2.0 RFC][1].  Requires [HTTP Basic authentication][3] of the client
using `client_id` and `client_secret`.  Used by clients to acquire an access
tokens.

This endpoint must support `access_token` creation based on an
`authorization_code` generated by the `/authorization` endpoint or based on a
`refresh_token` generated by the `/tokens` endpoint.

The server must support getting an `access_token` based on a `refresh_token`
with reduced `scope`.  Creating an `access_token` with reduced `scope` should
not invalidate any other `access_token` with a different `scope`.

The response will contain ID Token when `OpenID` is requested `scope` value, as
per the OpenID Connect standard.

If an ID Token is returned by this endpoint, the ID Token must be signed.


## Client API
This section describes API endpoints for registering and modifying clients of
the auth server.

### POST /v1/clients
Requires API key authentication of an administrative user.  Used directly by
administrative users to register a new client, generating its `client_id` and
`client_secret` (if the client is `confidential`).

Only `confidential` clients are allowed to use endpoints that require
authentication.

The request body is a JSON dictionary with the following parameters:

- `allowed_scopes`
    - List of scopes the client is allowed to request access to.
    - Required
- `type`
    - String
    - Required
    - Allowed values: `confidential`, `public`
- `default_scopes`
    - JSON list of scopes the client should be granted access to if `scope` is
      omitted in a request to `/authorization`.
    - May be an empty list
    - Required
- `name`
    - String
    - Unique
    - Required
- `redirect_uri_regex`
    - Regular expression for validating values of `redirtect_uri`.
    - Required
- `public_key`
    - Required if `type` is `confidential`
    - Forbidden if `type` is `public`
    - Dictionary containing
        - `kid`
        - `key` the serialized key
        - `alg` the preferred JWE alg to use for encryption
        - `enc` the preferred JWE enc to use for encryption
- `audience_for`
    - List of scopes for which the client should be added to `aud` in generated
      ID Tokens.
    - Optional

#### Responses
Success:

- HTTP 201 (Created)
    - Respond with the full resource including `client_id` and, if
      `confidential` then `client_secret`.

Error:

- HTTP 400 (Bad Request)
    - possible causes:
        - Missing or invalid `redirect_uri_regex`.
        - Missing scope parameters.
        - Client `name` is not unique
        - `default_scopes` includes scopes that are not in `allowed_scopes`
- HTTP 401 (Unauthenticated)
    - Includes the header `WWW-Authenticate: Basic`.
- HTTP 403 (Not Authorized)
    - Invalid `Authorization` header.

### PATCH /v1/clients/(id)
Requires API key authentication of an administrative user.  Used directly by
administrative users to update or invalidate a client and all access tokens and
authorization codes associated with it.


## User API
This section describes API endpoints for managing user credentials.

### POST /v1/api-keys
Requires [HTTP Basic authentication][3] of the user.  Used directly by users to
generate a new API key for themselves.  Invalidates existing API keys, but not
the access tokens or authorization codes associated with them.

### PATCH /v1/api-keys/(key)
Requires [HTTP Basic authentication][3] of the user to whom the API Key
belongs, or API key authentication of an administrative user.  Used directly by
users and administrative users to revoke a key and the access tokens and
authorization codes associated with it.

### PATCH /v1/users/(id)
Requires API key authentication of an administrative user.  Used directly by
administrative users to revoke all API keys, access tokens and authorization
codes associated with a user.


<!-- References -->
[1]: https://tools.ietf.org/html/rfc6749 "The OAuth 2.0 Authorization Framework"
[2]: https://tools.ietf.org/html/rfc6750 "The OAuth 2.0 Authorization Framework: Bearer Token Usage"
[3]: https://tools.ietf.org/html/rfc2617 "HTTP Authentication: Basic and Digest Access Authentication"
[4]: http://openid.net/specs/openid-connect-core-1_0.html "OpenID Connect Core 1.0"