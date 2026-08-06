# Keycloak OIDC authentication

Authentication in the TU Graz repository is done through OIDC protocol.

## Configuration

```
from invenio_oauthclient.contrib.keycloak import KeycloakSettingsHelper

_keycloak_helper = KeycloakSettingsHelper(
  title="TUGRAZ",
  description="TUGRAZ SSO",
  base_url="https://auth-test.tugraz.at",
  realm="tugraz",
  app_key="TUGRAZ_KEYCLOAK_APP_CREDENTIALS",
  # Adds "/auth/" between the base URL and realm names for
  # generated Keycloak URLs (default: True)
  legacy_url_path=True,
)
```

The `_keycloak_helper` is the main object that is used to build the Keycloak configuration.

Setup the credentials in the ENV variables:

```
INVENIO_TUGRAZ_KEYCLOAK_APP_CREDENTIALS={'consumer_key': '<client>', 'consumer_secret': '<secret>'}
``` 

Pay attention that the variable name after `INVENIO_` prefix must match the `app_key` value previously setup in the `_keycloak_helper`.

```
_keycloak_helper.remote_app["signup_handler"]["setup"] = "invenio_config_tugraz.utils:tugraz_setup_handler"
_keycloak_helper.remote_app["signup_handler"]["info_serializer"] = "invenio_config_tugraz.utils:tugraz_info_serializer"
```

Setup custom signup handlers for specific username and role management for users that login with this provider.

``` 
CONFIG_TUGRAZ_OAUTH_USERNAME_ATTRIBUTE = "sub"
```

Choose which token attribute should be used to configure the username.

``` 
OAUTHCLIENT_KEYCLOAK_REALM_URL = _keycloak_helper.realm_url
OAUTHCLIENT_KEYCLOAK_USER_INFO_URL = _keycloak_helper.user_info_url
OAUTHCLIENT_KEYCLOAK_VERIFY_EXP = True  # whether to verify the expiration date of tokens
OAUTHCLIENT_KEYCLOAK_VERIFY_AUD = True  # whether to verify the audience tag for tokens
OAUTHCLIENT_KEYCLOAK_AUD = "<client>"  # probably the same as the client ID
OAUTHCLIENT_KEYCLOAK_USER_INFO_FROM_ENDPOINT = True

from invenio_oauthclient.views.client import auto_redirect_login

ACCOUNTS_LOGIN_VIEW_FUNCTION = auto_redirect_login  # autoredirect to external login if enabled
OAUTHCLIENT_AUTO_REDIRECT_TO_EXTERNAL_LOGIN = False  # autoredirect to external login
```

OAuth specific configurations. Make sure to have `OAUTHCLIENT_KEYCLOAK_AUD` as the Keycloak configured client for the instance.

``` 
OAUTHCLIENT_REMOTE_APPS = {
    "keycloak": _keycloak_helper.remote_app,
}
``` 
Finish up by adding the remote app to the respective configuration list.




Before configuring the invenio instance, make sure to have a client setup in a Keycloak instance.

## Moving from one Keycloak client to another

If the running instance is configured with one client and there is a plan to update/change this Keycloak client, make sure to invalidate all users that have a linked account to Keycloak.

**Note!** This only applies if the updated client will return the same username. Nevertheless, it would still be a good idea to remove previous users from this table.

Go to the postgresql server and run:

`delete from accounts_useridentity where method = 'keycloak'`

This will only delete the link between that username and Keycloak provider, allowing a new link to be created with the new client.
