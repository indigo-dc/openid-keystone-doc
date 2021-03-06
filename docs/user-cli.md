# User manual

## Command Line Interface (CLI)

You need a working installation of the OpenStack client (that is,
python-openstackclient) with a recent version of keystoneauth
(at least 2.10.0).

Moreover, should to install the `keystoneauth-oidc-authz-code` plugin if you
plan to use the OpenID Connect authorization code grant type. The package is
available in the INDIGO repositories for the distribution of your choice.

When this package is installed it will provide you with a `v3oidccode`
authentication plugin.

## Manual installation of CLI

You can install the client and library from PyPI:

```
# pip install python-openstackclient
# pip install keystoneauth-oidc-authz-code
```

## Usage


### Using OpenID Connection Authorization Code grant type:

You have to specify the `v3oidccode` in the `--os-auth-type` option and provide
a valid autorization endpoint with `--os-authorization-endpoint` or a valid
discovery endpoint with `--os-discovery-endpoint`:

```
openstack --os-auth-url https://keystone.example.org:5000/v3 \
    --os-auth-type v3oidccode \
    --os-identity-provider <identity-provider> \
    --os-protocol <protocol> \
    --os-project-name <project> \
    --os-project-domain-id <project-domain> \
    --os-identity-api-version 3 \
    --os-client-id <OpenID Connect client ID> \
    --os-client-secret <OpenID Connect client secret> \
    --os-discovery-endpoint https://idp.example.org/.well-known/openid-configuration \
    --os-openid-scope "openid profile email" \
    token issue
```

### Using the Access Token plugin

* Go to: https://iam-test.indigo-datacloud.eu/
* select: Manage Active Tokens
* In `access tokens` you have the client - in this case "INCD-Openstack"
and the access token
* copy paste it into a file on the client machine or:
    `export iam_at=<THE ACCESS TOKEN>`

![IAM login](images/iam-at01.png)

With this access token you can obtain an openstack token:

```
$ openstack --os-auth-type v3oidcaccesstoken --os-access-token $iam_at --os-auth-url https://nimbus.ncg.ingrid.pt:5000/v3 --os-protocol oidc --os-identity-provider indigo-dc --os-identity-api-version 3 token issue

+---------+--------------------+
| Field   | Value              |
+---------+--------------------+
| expires | 2016-08-16 XXXX    |
| id      | XXXXXXXXXXXXXXX    |
| user_id | NNNNNNNNNNNNNNN    |
+---------+--------------------+
```

Note that there are at the moment two Openstack sites registered with
the IAM instance:
* test01.ifca.es
* nimbus.ncg.ingrid.pt

```
export kid=<keystone token id: XXXXX>
```
