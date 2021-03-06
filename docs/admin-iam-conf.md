# Configuration and administration manual

## INDIGO-DataCloud IAM Configuration

Configuring Keystone to accept the INDIGO IAM OpenID Connect
authentication is basically configuring it for OpenID Connect.

This is a step-by-step guide on how to set it up.

### Requirements

* Openstack Liberty or later.
* Keystone API v3 with federations enabled.
  * http://docs.openstack.org/security-guide/identity/federated-keystone.html

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
