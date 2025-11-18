# Current Auth Flows

> Confluence Page ID: 675863003175, Version: 3

There are 4 possible authentication mechanisms currently supported for External API integration.

note

There appear to be additional legacy flows in place

There appear to be additional legacy flows in place

1. App Key and Digest authentication. (Note: This method is currently listed as deprecated and is in the process of being retired by the API application teams).
2. Portal Framework (PFW) token based auth. This is an oauth2 based authentication flow used by Control Center for human interactions with the system.
3. Lumen Identity Access Management (LIAM) token based auth. This is an oauth2 based authentication flow used by Developer Center APIs for machine to machine interactions. (This is the go forward mechanism for API integrations).
4. APIGEE token based auth. This is an oauth2 based authentication flow hosted by APIGEE itself and is only supported for legacy app integrations.

It is critical to note that the 4 JWT flows are distinct and generate tokens independently, signed by different authorities.

Detailed sequence diagrams for the assorted flow can be found here: Integrating with LIAM APIs

For the purposes of this document I will maintain focus on the go forward LIAM auth flow as highlighted in the current developer center documentation.

note

There is a secondary auth flow that is similar to the one outlined below but instead adds the following additional steps:

* Initial client login occurs directly with Azure B2C using standard client\_credential grant type with client credential and secret
* The B2C auth token is then presented to LIAM as client\_assertion with client\_id where it is validated by:

  1. Ensuring “azp” claim equals registered client’s clientId
  2. Uses clientId to fetch LIAM user details
  3. Validates that environment in returned user matches current environement

There is a secondary auth flow that is similar to the one outlined below but instead adds the following additional steps:

* Initial client login occurs directly with Azure B2C using standard client\_credential grant type with client credential and secret
* The B2C auth token is then presented to LIAM as client\_assertion with client\_id where it is validated by:

  1. Ensuring “azp” claim equals registered client’s clientId
  2. Uses clientId to fetch LIAM user details
  3. Validates that environment in returned user matches current environement

API credential creation
-----------------------

1. A control center user with developer permissions in their account can request to create and API application. At creation time the developer will:

   1. Associate a set of BANs to the application.
   2. Select token lifetime.
2. App creation occurs in AzureB2C and client credentials are returned to the user.
3. Once crential has been created they are sent to the `https://api.lumen.com/oauth/v2/token` using standard oauth client credentials grant\_type.
4. Client credentials are sent to AzureB2C where they are authorized.
5. After authentication is validated a LIAM token is generated populating claims from it’s own user info. This token is returned to the login requester.
6. Token is then sent in the Authorization header for subsequent calls to the API where the API may enforce additional Authz.

note

APIs may require additional headers to be sent as part of their Authentication requirements such as x-customer-number and x-billing-account-number

APIs may require additional headers to be sent as part of their Authentication requirements such as x-customer-number and x-billing-account-number

At a high level the flow appears as follows:

![image-20251117-161652.png](images/675863003175-0-image-20251117-161652.png)