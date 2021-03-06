# Recommendations for Client App Implementations

**Note:** This spec is a component of the parent
[Solid specification](README.md); the parent spec and all its components are
versioned as a whole.

## Finding out the identity currently used

Regardless of the authentication mechanism that was used during an HTTP request,
the server must always return a `User` header, which contains a URI representing
the user's identity. For example, the `User` header may contain either an HTTP
URI (i.e. the WebID of the user that was just authenticated), or a different URI
(`mailto:`, `dns:`, `tel:`, etc.).

The `User` header can also be used to verify that a user has successfully
authenticated (specifically, that the HTTP URI that the header contains points
to a valid WebID profile), as well as to bootstrap the way apps personalize the
user experience, since apps have an easy way of discovering the user's identity.

Here is an example:

REQUEST:

```
GET /data/ HTTP/1.1
Host: example.org
```

RESPONSE:

```
HTTP/1.1 200 OK
...
User: https://alice.example.org/card#me
```

**Important:** Javascript clients MUST set the XHR flag `withCredentials` to
`true` when making authenticated requests, in order to force browsers to send
the client certificate. For example, Firefox will not send the client
certificate unless this flag is set to true.

## Creating new accounts

Before creating new accounts, client applications must be able to check whether
or not an account exists. To do that, clients only need to send a `HEAD` request
to the account root URI. For example, let's assume the user *Alice* wants to
create an account on `example.org`, using the username `alice`. The client will
perform a `HEAD` request to the `alice.example.org` subdomain.

REQUEST:

```
HEAD / HTTP/1.1
Host: alice.example.org
```

RESPONSE:

```
HTTP/1.1 200 OK
```

If the HTTP status code returned is `200`, then it means an account with that
name exists already.

If the status code returned is `404`, it means that the account is available.

Once the client application has verified that the account is available, it can
now proceed to create it. To do so, it must submit a form (or emulate it) to the
*account URI* it previously checked (e.g. alice.example.org), containing at
least the following form parameter names:

 * `username` (required) - the account name that will be used as the subdomain
  (i.e. `alice`)
 * `email` (optional) - the email of the user, which may be used for account
  recovery and/or account validation

**IMPORTANT** At this point, the server should also automatically consider the
user to be authenticated, and issue a cookie. This will allow the user to
properly manage the following steps that may require authentication.

Once submitted, the server will take charge of creating the necessary
workspaces, setting the access control policies and creating the user's WebID
profile document.

### Issuing the client certificate

**Attention!** Because creating client certificates requires the [keygen HTML5
element](http://www.w3schools.com/tags/tag_keygen.asp),
which does not work with AJAX reques ts, the client must submit a form to the
**account host URI** -- i.e. `https://user.example.org/`. This restriction means
that a predefined set of form element names must be respected on the server.
Here is the minimum list of form element names (case sensitive!) that **MUST**
be sent by signup applications, in order to achieve interoperability:

 * `spkac` - contains the *certificate signing request* (CSR) generated by the
  `<keygen>` element. (see [SPKAC](https://en.wikipedia.org/wiki/SPKAC))
 * `webid` - the WebID of the user
 * `name` - the name (CN) that will be used in the certificate

The server will update the user's profile by adding a representation of the
public key (as modulus and exponent) it obtained from the certificate, according
to the [WebID-TLS specification](http://www.w3.org/2005/Incubator/webid/spec/tls/#vocabulary).

**IMPORTANT** Servers should only return the certificate in the response, while
also setting the Content-Type header to the proper mime type value (as seen
below), otherwise the certificate will fail to install in the browser.

```
Content-Type: application/x-x509-user-cert
```

Unfortunately, there is currently no browser API to discover whether or not a
certificate was properly installed in the browser.
