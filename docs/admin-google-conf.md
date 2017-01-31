# Configuration and administration manual

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
