# Configuration and administration manual

## Configuring Multiple OpenID Providers

Apache2 allows you to configure multiple OpenID connect providers. To do
so, you need to change few things from previous configuration, oriented
to work with only one provider. In this example, we will configure both
Google OpenID and INDIGO IAM.

### Preparing Metadata files

The main difference between between original configuration (to accept just
one provider) and this one, is that we need to create a set of files that
stores the configuration ready to be queried by apache. In particular,
these are the files:

* `<urlencoded-issuer-value-with-https-prefix-and-trailing-slash-stripped>.provider`
  * contains (standardized) OpenID Connect Discovery OP JSON metadata where 
  ach name of the file is the url-encoded issuer name of the OP that is
  described by the metadata in that file.
* `<urlencoded-issuer-value-with-https-prefix-and-trailing-slash-stripped>.client`
  * contains statically configured or dynamically registered Dynamic Client
  Registration specific JSON metadata (based on the OpenID Connect Client
  Registration specification) and the filename is the url-encoded issuer
  name of the OP that this client is registered with.
* `<urlencoded-issuer-value-with-https-prefix-and-trailing-slash-stripped>.conf`
  * contains mod_auth_openidc specific custom JSON metadata that can be
  used to overrule some of the settings defined in auth_openidc.conf on
  a per-client basis. The filename is the URL-encoded issuer name of the
  OP that this client is registered with.

Step by step would be:

1. A folder where all the files will be stored must be created. In this
case we select `/var/cache/apache/metadata`. The directory and files must
be writable by user running apache (likely apache user).
2. The files `*.provider` must contain the JSON metadata that is provided
by the IdP. It can be obtained with a curl (both on Google and INDIGO IAM):
  * `curl https://iam-test.indigo-datacloud.eu/.well-known/openid-configuration`
  * `curl https://accounts.google.com/.well-known/openid-configuration`
3. Create both files accounts.google.com.provider and `iam-test.indigo-datacloud.eu.provider`.
4. The files `*.client` contain some of the configuration used on the
previous "one IdP" configuration, but in JSON form. It must contain
Client ID, Client Secret and response type:
```
{
  "client_id" : "client_id",
  "client_secret" : "Client_Secret",
  "response_type" : "id_token"
}
```

The INDIGO IAM response type should be `code`. INDIGO IAM offers a complete
JSON that can be pasted in this file (We recommend to do so). This JSON
can be found in JSON tab in Client Configuration. That way we have create
both files `accounts.google.com.client` and `iam-test.indigo-datacloud.eu.client`.

1. `*.conf` file can contain other OIDC module parameters, like scope:
```
{
  "scope" : "openid email profile"
}
```

That way we have create both files `accounts.google.com.conf` and
`iam-test.indigo-datacloud.eu.conf`.

### Keystone configuration

Although apache could manage both providers with just one address, it
would add one more step to authentication, so the recommended way is to
add a new authentication method.

To do so, edit the keystone.conf file:

```
methods = external,password,token,oauth1,oidc,iam
oidc = keystone.auth.plugins.mapped.Mapped
iam = keystone.auth.plugins.mapped.Mapped
```

### Apache2 Configuration

Final step is to change site configuration to manage the new schema shown
in the following example:

```
    OIDCMetadataDir /var/cache/httpd/metadata
    OIDCProviderTokenEndpointAuth client_secret_basic
    OIDCCryptoPassphrase <passphrase>
    OIDCRedirectURI http://testkeystone.com:5000/v3/auth/OS-FEDERATION/websso/oidc/redirect
    OIDCClaimPrefix "OIDC-"

    <Location ~ "/v3/auth/OS-FEDERATION/websso/oidc">
      OIDCDiscoverURL http://testkeystone.com:5000/v3/auth/OS-FEDERATION/websso/oidc/redirect?iss=https%3A%2F%2Faccounts.google.com
      AuthType openid-connect
      Require valid-user
      LogLevel debug
    </Location>

    <Location ~ "/v3/auth/OS-FEDERATION/websso/iam">
      OIDCDiscoverURL http://testkeystone.com:5000/v3/auth/OS-FEDERATION/websso/oidc/redirect?iss=https%3A%2F%2Fiam-test.indigo-datacloud.eu
      AuthType openid-connect
      Require valid-user
      LogLevel debug
    </Location>
```
