# Configuration and administration manual

## INDIGO-DataCloud IAM Configuration

Configuring Keystone to accept the INDIGO IAM OpenID Connect
authentication is basically configuring it for OpenID Connect.

This is a step-by-step guide on how to set it up.

### Using INDIGO IAM OpenID connect API (PROVISIONAL)

INDIGO IAM needs to be configured to work with a client, so it need to
be registered and some parameters tuned.

1. Go to the test IAM instance. You need to be registered by INDIGO AAI
Team, so contact them in order to do so.
2. Register a new client, under Self Service Client Registration.
3. Introduce the name of your choice.
4. Introduce the allowed redirect URIs. Assuming that your server is
keystone.example.org and the endpoint is running in port 5000 the
redirect URI will be https://keystone.example.org:5000/v3/auth/OS-FEDERATION/websso/oidc/redirect.
5. In Credentials tab, keep Client Secret over HTTP Basic. It is the
only valid method at this moment.
6. Once you save it, go to the main tab again and keep a copy of the
following fields.
  * Client ID.
  * Client Secret.
  * Registration Endpoint.
  * Registration Access Token.

Please keep them in a secure place, as you will need them to configure
your Keystone server and further modify your client if needed.

### Installing mod_auth_openidc

Keystone is a WSGI application that runs under Apache 2 and can be
combined with different authentication systems provided by Apache modules.

In order to work with OpenID Connect, install and configure mod_auth_openidc.

There are different releases available for different Linux distributions
as well as source code to be compiled.

Please check the most appropriate version for your distribution and install it:
* https://github.com/pingidentity/mod_auth_openidc/releases

### `mod_auth_openidc` configuration

IAM is a prototype at this stage, and it will only work with Basic HTTP
authentication (this will changed in the future), so you need to specify
this in the configuration.

Ensure that the mod_auth_openidc is enabled:

`LoadModule auth_openidc_module /usr/lib/apache2/modules/mod_auth_openidc.so`

Add the following lines inside the `VirtualHost` in your Keystone Apache
configuration file.

Default location varies between distributions:
* Ubuntu/Debian: /etc/apache2/sites-enabled/wsgi-keystone-conf
* Centos: /etc/httpd/conf.d/wsgi-keystone.conf

```
    (...)
    <VirtualHost *:5000>

        (...)

        OIDCClaimPrefix                 "OIDC-"
        OIDCResponseType                "code"
        OIDCScope                       "openid email profile"
        OIDCProviderMetadataURL         https://iam-test.indigo-datacloud.eu/.well-known/openid-configuration
        OIDCClientID                    <CLIENT ID>
        OIDCClientSecret                <CLIENT SECRET>
        OIDCProviderTokenEndpointAuth   client_secret_basic
        OIDCCryptoPassphrase            <PASSPHRASE>
        OIDCRedirectURI                 https://<KEYSTONE HOST>:5000/v3/auth/OS-FEDERATION/websso/oidc/redirect

        # The JWKs URL on which the Authorization publishes the keys used to sign its JWT access tokens.
        # When not defined local validation of JWTs can still be done using statically configured keys,
        # by setting OIDCOAuthVerifyCertFiles and/or OIDCOAuthVerifySharedKeys.
        OIDCOAuthVerifyJwksUri "https://iam-test.indigo-datacloud.eu/jwk"

        <Location ~ "/v3/auth/OS-FEDERATION/websso/oidc">
            AuthType  openid-connect
            Require   valid-user
            LogLevel  debug
        </Location>

        <Location ~ "/v3/OS-FEDERATION/identity_providers/indigo-dc/protocols/oidc/auth">
            AuthType  oauth20
            Require   valid-user
            LogLevel  debug
        </Location>

        (...)

    </VirtualHost>
```

Substitute the following values:

* `<CLIENT ID>`: Client ID as obtained from the IAM.
* `<CLIENT SECRET>`: Client Secret as obtained from the IAM.
* `<PASSPHRASE>`: A password used for crypto purposes. Put something of your choice here.
* `<KEYSTONE HOST>`: Your Keystone host.

### OpenStack Identity (Keystone) configuration

Once Apache is set up to authenticate using OpenID Connect, you need to
configure Keystone so that it is able to consume the OpenID Connect credentials.

This is done by setting up the Federation plugins. Ensure that the
following options are set in the corresponding sections in your keystone
configuration file (`/etc/keystone/keystone.conf`):

```
[auth]
methods = external,password,token,oauth1,oidc
oidc = keystone.auth.plugins.mapped.Mapped

[oidc]
remote_id_attribute = HTTP_OIDC_ISS

[federation]
remote_id_attribute = HTTP_OIDC_ISS
trusted_dashboard = https://<HORIZON ENDPOINT>/dashboard/auth/websso/
sso_callback_template = /etc/keystone/sso_callback_template.html
```

Ensure that `/etc/keystone/sso_callback_template.html` exists on your system.

### Keystone Groups, Projects and Mapping setup

To manage federated users, Keystone will maps them according to some
rules inside groups. Then, roles can be granted to these groups within
one or more projects/tenants.

First create a group that will hold all the INDIGO users, after create a
project for them. Note that you can chose whatever naming you want.

```
$ openstack group create indigo_group --description "INDIGO Federated users group"
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | INDIGO Federated users group     |
| domain_id   | default                          |
| id          | 75fd662f04774c45b2c99da729a6bcfe |
| name        | indigo_group                     |
+-------------+----------------------------------+

$ openstack project create indigo --description "INDIGO project"
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | INDIGO project                   |
| domain_id   | default                          |
| enabled     | True                             |
| id          | 3832777e8b50461a855323793dbc9c31 |
| is_domain   | False                            |
| name        | indigo                           |
| parent_id   | None                             |
+-------------+----------------------------------+
```

Grant user roles to the whole indigo_group into the indigo project:

```
$ openstack role add user --group indigo_group --project indigo
```

Now the federation plugin needs to be setup.

#### Openstack Liberty

The first step is adding a mapping, so that incoming users are matched to a specific group. Using the group created above, create the following JSON mapping. Save the following into a temporary file like indigo_mapping.json:

```
    [
      {
        "local": [
          {
            "group": {
              "id": "75fd662f04774c45b2c99da729a6bcfe"
              }
            }
          ],
        "remote": [
            {
              "type": "HTTP_OIDC_ISS",
              "any_one_of": [
                "https://iam-test.indigo-datacloud.eu/"
                ]
              }
            ]
      }
    ]
```

#### Openstack Mitaka

In the Mitaka release, shadow userswhere introduced, as such the JSON mapping is the following:

```
[
  {
    "local": [
      {
        "group": {
          "id": "45af1ef0f45345428b814d421f0ccf57"
        },
        "user": {
          "domain": {
            "id": "default"
          },
          "type": "ephemeral",
          "name": "{0}_iamID"
        }
      }
    ],
    "remote": [
      {
        "type": "OIDC-sub"
      },
      {
        "type": "HTTP_OIDC_ISS",
        "any_one_of": [
          "https://iam-test.indigo-datacloud.eu/"
        ]
      }
    ]
  }
]
```

The remote (IAM) ID of the user - `OIDC-sub` attribute - will be mapped
into into an ephemeral keystone local user - `< OIDC-sub >_iamID`

#### Hereon procedure is common to liberty and mitaka.

Load the mapping as follows:

```
$ openstack mapping create indigo_mapping --rules indigo_mapping.json
```

Create the corresponding Identity Provider and protocol.

Note: to get an homogeneous authentication flow in the INDIGO sites, one
should use the same values for the identity provider (indigo-dc) and
protocol (openidc):

```
$ openstack identity provider create indigo-dc --remote-id https://iam-test.indigo-datacloud.eu/
+-------------+--------------------------------------------+
| Field       | Value                                      |
+-------------+--------------------------------------------+
| description | None                                       |
| enabled     | True                                       |
| id          | indigo-dc                                  |
| remote_ids  | [u'https://iam-test.indigo-datacloud.eu/'] |
+-------------+--------------------------------------------+

$ openstack federation protocol create oidc --identity-provider indigo-dc --mapping indigo_mapping
+-------------------+----------------+
| Field             | Value          |
+-------------------+----------------+
| id                | oidc           |
| identity_provider | indigo-dc      |
| mapping           | indigo_mapping |
+-------------------+----------------+
```

If you need to change the mapping at a later stage, you can update it by:

```
$ openstack mapping set --rules indigo_mapping.json indigo_mapping
```

### OpenStack Dashboard (Horizon) Configuration

Whenever a user selects the IAM authentication it is redirected to the
correct authentication server.

Ensure that the following is included in your dashboard configuration
file, usually located under `/etc/openstack_dashboard/local_settings.py`
or `/etc/openstack-dashboard/local_settings`:

```
WEBSSO_ENABLED = True
WEBSSO_INITIAL_CHOICE = "credentials"

WEBSSO_CHOICES = (
    ("credentials", _("Keystone Credentials")),
    ("oidc", _("INDIGO-DataCloud IAM"))
)
```

## Using Google OpenID connect API

Google need to trust in requester application to work with. To do so, you
need to access to Google developers console and create and configure a
new credential project.

1. Go to: https://console.developers.google.com/apis/credentials
2. Create Credentials > OAuth Client ID
3. Application Type: Web Application
4. Name: Service Provider (SP) name
5. Let's assume that our server is testkeystone.com and it is configured public in port 5000:
  * Authorized JavaScript origins: http://testkeystone.com.
  * Authorized redirect URIs: http://testkeystone.com:5000/v3/auth/OS-FEDERATION/websso/oidc/redirect. Keep that in mind.
  * Copy client ID and client secret. You will need them for configuration.

### Installing mod_auth_openidc

Keystone works under apache2 and can be combined with different authentication
systems provided by Apache modules.

In order to work with OpenID connect, mod_auth_openidc needs to be installed
and configured. There are different releases available for different Linux
distributions as well as source code to be compiled. All those packages and
source code can be found here:
* https://github.com/pingidentity/mod_auth_openidc/releases

### `mod_auth_openidc` Configuration

Add the following lines (example) to wsgi-keystone configuration file:

* Ubuntu systems: `/etc/apache2/sites-enabled/wsgi-keystone.conf`
  * `LoadModule auth_openidc_module /usr/lib/apache2/modules/mod_auth_openidc.so`
* Centos/RHEL: `/etc/httpd/conf.d/wsgi-keystone.conf`
  * After installation of `mod_auth_openidc-<VERSION>.el7.centos.x86_64.rpm`
  you should have everything properly configure to be used by the apache server:
  `/etc/httpd/conf.modules.d/10-auth_openidc.conf`
  `/usr/lib64/httpd/modules/mod_auth_openidc.so`

Inside VirtualHost *:5000:
```
    OIDCClaimPrefix "OIDC-"
    OIDCResponseType "id_token"
    OIDCScope "openid email profile"
    OIDCProviderMetadataURL https://accounts.google.com/.well-known/openid-configuration
    OIDCClientID <your_Client_ID>
    OIDCClientSecret <your_secret>
    OIDCCryptoPassphrase <passphrase>
    OIDCRedirectURI http://testkeystone.com:5000/v3/auth/OS-FEDERATION/websso/oidc/redirect

    <Location ~ "/v3/auth/OS-FEDERATION/websso/oidc">
      AuthType openid-connect
      Require valid-user
      LogLevel debug
    </Location>
```

* your_Client_ID: Client ID copied from Google developers Console
* your_secret: Secret copied from Google developers Console
* passphrase: Whatever

### Keystone configuration

Add or modify the following configuration options in /etc/keystone/keystone.conf

```
[auth]
methods = external,password,token,oauth1,oidc
oidc = keystone.auth.plugins.mapped.Mapped
...
[oidc]
remote_id_attribute = HTTP_OIDC_ISS
...
[federation]
remote_id_attribute = HTTP_OIDC_ISS
trusted_dashboard = http://testkeystone.com/dashboard/auth/websso/
sso_callback_template = /etc/keystone/sso_callback_template.html
```

### Dashboard Configuration

Add or modify the following configuration options in /etc/openstack_dashboard/local_settings:

```python
ALLOWED_HOSTS = ['testkeystone.com']
OPENSTACK_API_VERSIONS = {
    "identity": 3,
}

CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
        'LOCATION': '0.0.0.0:11211',
    }
}

OPENSTACK_HOST = "testkeystone.com"
OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "_member_"

WEBSSO_ENABLED = True
WEBSSO_INITIAL_CHOICE = "credentials"

WEBSSO_CHOICES = (
    ("credentials", _("Keystone Credentials")),
    ("oidc", _("OpenID Connect"))
)
```

### Create Groups, Projects

For matching Google user to a certain project, you could use one existing
or create a new one like this:

```
openstack group create google_group
openstack project create google_project
openstack role add admin --group google_group --project google_project
```

### Group matching

OpenID connect/Google login needs to be matched with an specific group.
Regarding the previous configuration (group ID, type), create the following
json file (`google_mapping.json`), that will be used to create the mappings
in the keystone federations:

```
[
  {
    "local": [
        {
        "group": {
          "id": "6a651fb4bbb4494a9bb93c5afdaa9b24"
          }
        }
    ],
    "remote": [
        {
        "type": "HTTP_OIDC_ISS",
        "any_one_of": [
          "https://accounts.google.com"
          ]
        }
    ]
  }
]
```

The next steps are to create the mapping, the identity provder and the
federation protocol:

```
openstack mapping create google_mapping --rules google_mapping.json
openstack identity provider create google_idp --remote-id https://accounts.google.com
openstack federation protocol create oidc --identity-provider google_idp --mapping google_mapping
```

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
