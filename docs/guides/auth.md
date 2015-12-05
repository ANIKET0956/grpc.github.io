---
layout: docs
title: Authentication
---

<h1 class="page-header">Authentication</h1>

<p class="lead">gRPC is designed to plug-in a number of authentication mechanisms. This document provides a quick overview of the various auth mechanisms supported, discusses the API with some examples, and concludes with a discussion of extensibility.</p>

More documentation and examples are coming soon!

<div id="toc"></div>

## Supported auth mechanisms

###SSL/TLS

gRPC has SSL/TLS integration and promotes the use of SSL/TLS to authenticate the server,
and encrypt all the data exchanged between the client and the server. Optional
mechanisms are available for clients to provide certificates to accomplish mutual
authentication.

###OAuth 2.0

gRPC provides a generic mechanism (described below) to attach metadata based credentials
to requests and responses. Additional support for acquiring Access Tokens while
accessing Google APIs through gRPC is provided for certain auth flows, demonstrated
through code examples below.

*WARNING*: Google OAuth2 credentials shall only be used to connect to Google services.
Sending a Google issued OAuth2 token to a non-Google service could result in this token
being stolen and used to impersonate the client to Google services.

## API

To reduce complexity and minimize API clutter, gRPC works with a unified concept of
Credentials objects.

Credentials can be of two types:

- Channel Credentials which are attached to a channel such as SSL credentials.
- Call Credentials which are attached to a call (or `ClientContext` in C++).

Credentials can be composed using `CompositeChannelCredentials` which associates
a `ChannelCredentials` and a `CallCredentials` in order to create a new
`ChannelCredentials`. The result will, for each call on the channel, send the
authentication data associated with the composed `CallCredentials`.

For example, a `ChannelCredentials` could be created from an `SslCredentials`
and an `AccessTokenCredentials`. The result, applied to a `Channel` would send
the access token for each call on this channel.

`CallCredentials` can also be composed using `CompositeCallCredentials`. The
resulting `CallCredentials`, when applied to a `ClientContext`, will trigger the
sending of the authentication data associated with the two `CallCredentials`.


###SSL/TLS for server authentication and encryption

This is the simplest authentication scenario, where a client just wants to
authenticate the server and encrypt all data.

```cpp
SslCredentialsOptions ssl_opts;  // Options to override SSL params, empty by default
// Create the credentials object by providing service account key in constructor
auto channel_creds = CredentialsFactory::SslCredentials(ssl_opts);
// Create a channel using the credentials created in the previous step
auto channel = CreateChannel(server_name, creds);
// Create a stub on the channel
std::unique_ptr<Greeter::Stub> stub(Greeter::NewStub(channel));
// Make actual RPC calls on the stub.
grpc::Status s = stub->sayHello(&context, *request, response);
```

For advanced use cases such as modifying the root CA or using client certs,
the corresponding options can be set in the SslCredentialsOptions parameter
passed to the factory method.


###Authenticating with Google

gRPC applications can use a simple API to create a credential that works in
various deployment scenarios.

```cpp
auto creds = CredentialsFactory::GoogleDefaultCredentials();
// Create a channel, stub and make RPC calls (same as in the previous example)
auto channel = CreateChannel(server_name, creds);
std::unique_ptr<Greeter::Stub> stub(Greeter::NewStub(channel));
grpc::Status s = stub->sayHello(&context, *request, response);
```

This channel credentials object works for applications using Service Accounts as
well as for applications running in [Google Compute Engine
(GCE)](https://cloud.google.com/compute/).  In the former case, the service
account’s private keys are loaded from the file named in the environment
variable `GOOGLE_APPLICATION_CREDENTIALS`. The keys are used to generate bearer
tokens that are attached to each outgoing RPC on the corresponding channel.

For applications running in GCE, a default service account and corresponding
OAuth2 scopes can be configured during VM setup. At run-time, this credential
handles communication with the authentication systems to obtain OAuth2 access
tokens and attaches them to each outgoing RPC on the corresponding channel.


### Extending gRPC to support other authentication mechanisms

The Credentials plugin API allows developers to plug in their own type of
credentials.

- The `MetadataCredentialsPlugin` abstract class contains the pure virtual
  `GetMetadata` method that needs to be implemented by a sub-class created by
  the developer.
- The `MetadataCredentialsFromPlugin` function which creates a `CallCredentials`
  from the `MetadataCredentialsPlugin`.

Here is example of a simple credentials plugin which sets an authentication
ticket in a custom header.

```cpp
class MyCustomAuthenticator : public grpc::MetadataCredentialsPlugin {
 public:
  MyCustomAuthenticator(const grpc::string& ticket) : ticket_(ticket) {}

  Status GetMetadata(
      grpc::string_ref service_url, grpc::string_ref method_name,
      const AuthContext& channel_auth_context,
      std::multimap<grpc::string, grpc::string>* metadata) override {
    metadata->insert(std::make_pair("x-custom-auth-ticket", ticket_));
    return Status::OK;
  }

 private:
  grpc::string ticket_;
};

auto call_creds =
    MetadataCredentialsFromPlugin(std::unique_ptr<MetadataCredentialsPlugin>(
        new MyCustomAuthenticator("super-secret-ticket")));
```

A deeper integration can be achieved by plugging in a gRPC credentials
implementation at the core level. gRPC internals also allow switching out
SSL/TLS with other encryption mechanisms.

## Examples

These authentication mechanisms will be available in all gRPC's supported languages.
The following sections demonstrate how authentication and authorization features
described above appear in each language: more languages are coming soon.

###SSL/TLS for server authentication and encryption (Ruby)

```ruby
# Base case - No encryption
stub = Helloworld::Greeter::Stub.new('localhost:50051')
...

# With server authentication SSL/TLS
creds = GRPC::Core::Credentials.new(load_certs)  # load_certs typically loads a CA roots file
stub = Helloworld::Greeter::Stub.new('localhost:50051', creds: creds)
```

###SSL/TLS for server authentication and encryption (C#)

```csharp
// Base case - No encryption/authentication
var channel = new Channel("localhost:50051", ChannelCredentials.Insecure);
var client = new Greeter.GreeterClient(channel);
...

// With server authentication SSL/TLS
var channelCredentials = new SslCredentials(File.ReadAllText("roots.pem"));  // Load a custom roots file.
var channel = new Channel("myservice.example.com", channelCredentials);
var client = new Greeter.GreeterClient(channel);
```

###Authenticating with Google (Ruby)

```ruby
# Base case - No encryption/authorization
stub = Helloworld::Greeter::Stub.new('localhost:50051')
...

# Authenticating with Google
require 'googleauth'  # from [googleauth](http://www.rubydoc.info/gems/googleauth/0.1.0)
...
creds = GRPC::Core::Credentials.new(load_certs)  # load_certs typically loads a CA roots file
scope = 'https://www.googleapis.com/auth/grpc-testing'
authorization = Google::Auth.get_application_default(scope)
stub = Helloworld::Greeter::Stub.new('localhost:50051',
                                     creds: creds,
                                     update_metadata: authorization.updater_proc)
```

###Authenticating with Google (Node.js)

```
// Base case - No encryption/authorization
var stub = new helloworld.Greeter('localhost:50051');
...
// Authenticating with Google
var GoogleAuth = require('google-auth-library'); // from https://www.npmjs.com/package/google-auth-library
...
var creds = grpc.Credentials.createSsl(load_certs); // load_certs typically loads a CA roots file
var scope = 'https://www.googleapis.com/auth/grpc-testing';
(new GoogleAuth()).getApplicationDefault(function(err, auth) {
  if (auth.createScopeRequired()) {
    auth = auth.createScoped(scope);
  }
  var stub = new helloworld.Greeter('localhost:50051',
                                    {credentials: creds},
                                    grpc.getGoogleAuthDelegate(auth));
});
```

###Authenticating with Google (C#)

**Base case - No encryption/authentication**
```csharp
var channel = new Channel("localhost:50051", ChannelCredentials.Insecure);
var client = new Greeter.GreeterClient(channel);
...
```

**Authenticate using JWT access token (recommended approach)**
```csharp
using Grpc.Auth;  // from Grpc.Auth NuGet package
...
// Loads Google Application Default Credentials with publicly trusted roots.
var channelCredentials = await GoogleGrpcCredentials.GetApplicationDefaultAsync();

var channel = new Channel("greeter.googleapis.com", channelCredentials);
var client = new Greeter.GreeterClient(channel);
...
```

**Authenticate using OAuth2 token (legacy approach)**
```csharp
using Grpc.Auth;  // from Grpc.Auth NuGet package
...
string scope = "https://www.googleapis.com/auth/grpc-testing";
var googleCredential = await GoogleCredential.GetApplicationDefaultAsync();
if (googleCredential.IsCreateScopedRequired)
{
    googleCredential = googleCredential.CreateScoped(new[] { scope });
}
var channel = new Channel("greeter.googleapis.com", googleCredential.ToChannelCredentials());
var client = new Greeter.GreeterClient(channel);
...
```

**Authenticate a single RPC call**
```csharp
var channel = new Channel("greeter.googleapis.com", new SslCredentials());  // Use publicly trusted roots.
var client = new Greeter.GreeterClient(channel);
...
var googleCredential = await GoogleCredential.GetApplicationDefaultAsync();
var result = client.SayHello(request, new CallOptions(credentials: googleCredential.ToCallCredentials()));
...

```

###Authenticating with Google (PHP)

```php
// Base case - No encryption/authorization
$client = new helloworld\GreeterClient(
  new Grpc\BaseStub('localhost:50051', []));
...

// Authenticating with Google
// the environment variable "GOOGLE_APPLICATION_CREDENTIALS" needs to be set
$scope = "https://www.googleapis.com/auth/grpc-testing";
$auth = Google\Auth\ApplicationDefaultCredentials::getCredentials($scope);
$opts = [
  'credentials' => Grpc\Credentials::createSsl(file_get_contents('ca.pem'));
  'update_metadata' => $auth->getUpdateMetadataFunc(),
];

$client = new helloworld\GreeterClient(
  new Grpc\BaseStub('localhost:50051', $opts));

```
