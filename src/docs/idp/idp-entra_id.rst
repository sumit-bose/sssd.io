Entra ID
########

As explained in :doc:`idp-introduction` SSSD is using IdP clients to access
user an group data in the IdP and to offer the `OAuth 2.0 Device Authorization
Grant flow <https://datatracker.ietf.org/doc/html/rfc8628>`_ for
authentication. How to setup such a client, which is also called application in
Entra ID, `can be found here
<https://learn.microsoft.com/en-us/entra/identity-platform/quickstart-register-app>`_.

To authenticate the application credentials must be added as `explained here
<https://learn.microsoft.com/en-us/entra/identity-platform/how-to-add-credentials?tabs=client-secret>`_.
Currently SSSD only supports client secrets.

Device Authorization Grant
**************************

Entra ID knows about `Public client and confidential client applications
<https://learn.microsoft.com/en-us/entra/identity-platform/msal-client-applications>`_.
To enable the `OAuth 2.0 Device Authorization Grant flow
<https://datatracker.ietf.org/doc/html/rfc8628>`_ the Entra ID setting "Allow
public client flow" must be set to "Yes". You can find this option when
selecting "Manage->Authentication" for your application in the "Settings" tab.

API Permissions
***************

To be able to read user and group information with the help of the
client/application credentials the application must have permissions to read
them. The permissions can be set by selection "Manage->API permissions" for the
application. SSSD requires

 * Group.Read.All
 * GroupMember.Read.All
 * User.Read
 * User.Read.All

and "admin consent" should be granted where needed.

If there is no admin consent given to "User.Read" user authentication might
fail if the requested scopes in the SSSD option `idp_auth_scope` does not
contain the "user.read" scope.
