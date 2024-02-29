# Apigee Custom Attributes

This sample shows how to use Custom Attributes in Apigee; specifically, how to set
and retrieve custom attributes on Developer Apps, or on API Products, or on API Product Operations.

## About API Products

In Apigee, the [API Product](https://cloud.google.com/apigee/docs/api-platform/publish/what-api-product)
is the unit of packaging for APIs.

- it allows an API Publisher to share an API out, via an API Catalog, or developer portal.
- it is the thing an API Consumer Developer gains access to, via the self-service API Catalog

Each API Product has a set of configuration data that can be used at runtime.
This might be data that allows the API Proxy to behave differently, depending on
the product that is being used to expose it. Gold, Silver, and Bronze products
might each have a different rate limit, for example. Or different pricing
levels. Or different target systems. Information about which data fields to include or
exclude from a response. OAuth scopes. It's a very flexible model.

### The Operations within an API Product

One particular aspect of an API Product is the set of Operations it allows.

API Products can group subsets of the operations available in one or more API
proxies, into a consumable unit. Let's look at an example. Suppose an API Proxy
fronts a contract management system. The designers of the API might use these
REST-ful Verb + Path combinations to support the use cases:

|  # | use case                                             | verb + path       |
| -- | ---------------------------------------------------- | ----------------- |
|  1 | review the list of signed or unsigned contracts      | `GET /contracts`    |
|  2 | review the detail of a single contract               | `GET /contracts/*`  |
|  3 | add a new contract into the system                   | `POST /contracts`   |
|  4 | update an unsigned contract with a signature         | `POST /contracts/*` |

## Credentials are "the key" for checking the Operation

Configuring the set of API Products with the various operations is one step in
configuring access to APIs. The next step is to grant access to these products
to one or more applications - this is done via app credentials. In Apigee, you
need to configure an App authorized for a product, to get a set of credentials.
This is typically done in the self-service developer portal (aka API
Catalog). After obtaining credentials, the developer embeds the credentials into
the app, and sends those credentials along with API requests.

  > As an alternative to allowing a developer to provision his or her own
  > credentials via the Developer portal, an administrator can provision credentials
  > via the Administrative UI or API. The administrator would need to then
  > distribute these credentials to the developer in some way.

When the API request reaches Apigee, the API proxy that handles it must invoke a
policy to verify the credentials - either
[VerifyAPIKey](https://cloud.google.com/apigee/docs/api-platform/reference/policies/verify-api-key-policy)
or
[OAuthV2/VerifyAccessToken](https://cloud.google.com/apigee/docs/api-platform/reference/policies/oauthv2-policy#verifyaccesstoken). When
that happens, Apigee checks the credential and maps it to one or more API
Products. (In the simple case, an app is authorized for a single API Product,
but Apigee allows apps to have access to more than one product via a single credential.)  Then, Apigee
checks that the current executing operation - using the REST model, this is a
verb+path combination - is included within one of the API Products that is
authorized for that credential. If not, then the request is rejected.

In our example, for a `GET /contracts` request, the credential must be authorized
for the viewer or admin product. For a `POST /contracts` request, the credential
must be authorized for either creator or admin.

## What Are Custom Attributes?

Custom Attributes are simply extra data associated to Apigee entities - like
developers, apps, API Products, or even operations within ApI Products.

You can use Custom Attributes to attach information to a product, or app (etc)
that can later be used during API proxy execution, in the handling of the API
call.  Some uses of custom attributes are:

- routing information for calls, based on the app
- a list of data properties to include or exclude from the upstream response
- a set of defaults to apply to requests
- a billing or tracking ID for use in upstream log systems

And many others.

In Apigee, custom attributes are available at runtime after verifying
credentials on the inbound API request.

This proxy shows how that works.

## Implementation in the API Proxy

This sample uses a simple API Proxy to demonstrate this function. In one flow, the proxy uses
a
[VerifyAPIKey](https://cloud.google.com/apigee/docs/api-platform/reference/policies/verify-api-key-policy)
policy to identify the calling application and the API product associated with
the proxy. In another flow, the proxy uses
an
[OAuthV2/VerifyAccessToken](https://cloud.google.com/apigee/docs/api-platform/reference/policies/oauthv2-policy#verifyaccesstoken)
policy to verify credentials.

All of the magic of authorizing the operation for the given
credential is done _implicitly_ by these policies. There's nothing else
you need to do in the API Proxy to make that happen.

For the purposes of demonstration, the proxy uses an
[AssignMessage](https://cloud.google.com/apigee/docs/api-platform/reference/policies/assign-message-policy)
policy to set a success response, which includes the API product name, the
operation that was authorized, and a few custom attributes that had been set on
the app and product and operation.

## Prerequisites

1. [Provision Apigee X](https://cloud.google.com/apigee/docs/api-platform/get-started/provisioning-intro)
2. Configure [external access](https://cloud.google.com/apigee/docs/api-platform/get-started/configure-routing#external-access) for API traffic to your Apigee X instance
3. Permissions to create and deploy proxies, to create products, and to register apps and developers in Apigee. Get these permissions via the Apigee orgadmin role, or the combination of two roles: API Admin and Developer Admin. ([more on Apigee-specific roles](https://cloud.google.com/apigee/docs/api-platform/system-administration/apigee-roles#apigee-specific-roles))
4. Make sure the following tools are available in your terminal's $PATH
    - [gcloud SDK](https://cloud.google.com/sdk/docs/install)
    - unzip
    - curl
    - jq

    Cloud Shell has these preconfigured.

## Manual Setup instructions

0. open a browser to console.cloud.google.com

1. from within that browser window open a "Cloud Shell" terminal.

2. Authenticate in the cloud shell terminal
   ```sh
   gcloud auth login
   ```

   When you do that, you _may_ see a warning, telling you:

   > You are already authenticated with gcloud when running
   > inside the Cloud Shell and so do not need to run this
   > command. Do you wish to proceed anyway?

   Ignore that :). Proceed anyway.  You may have to  open a URL in
   a browser tab, and then obtain an authorization code, and paste it
   back into the terminal.  Do all of that.

3. Clone this repo, and cd into the `custom-attributes-sample` directory

4. Edit `env.sh` and configure the following variables:

   - `PROJECT` the project where your Apigee organization is located
   - `APIGEE_HOST` the externally reachable hostname of the Apigee environment group that contains APIGEE_ENV
   - `APIGEE_ENV` the Apigee environment where the demo resources should be created

   Now source the `env.sh` file

   ```bash
   source ./env.sh
   ```

5. Configure the API proxies, the product and the app into your Apigee organization

   ```bash
   ./setup-custom-attributes-sample.sh
   ```

   This step will take a few minutes.
   When it finishes, it will print some information about the credentials it has provisioned.


### Send requests

To test the proxy, make requests using the API keys created by the setup script.

First, set the client ID and secret into shell variables:
```bash
  CLIENT_ID=sym2Bvv75Ecpven...output-from-above-script...ruWJUjZ0rtiZg73
  CLIENT_SECRET=AdkdklkdO...output-from-above-script...ym2Bvv75Ecpven0
```

Invoke your first request using the API Key:

```bash
curl -i -X GET https://${APIGEE_HOST}/custom-attributes-sample-verify/apikey -H APIKEY:$CLIENT_ID
```

Observe the custom attributes in the response.

Now try the token version. First, get a token:

```bash
curl -i -X POST -H 'content-type: application/x-www-form-urlencoded' \
     -u ${CLIENT_ID}:${CLIENT_SECRET} \
     "https://${APIGEE_HOST}/custom-attributes-sample-oauth2/token" \
     -d 'grant_type=client_credentials'
```

Then, use the token:

```bash
access_token=kjdfkl09e...output..from..above
curl -i -X GET https://${APIGEE_HOST}/custom-attributes-sample-verify/apikey -H token:$access_token
```

You should again observe the custom attributes in the response.

### Modify

If you like, use the Admin UI to modify the values of the custom attributes on
the product, the operation within the product, or the app.  Then re-run the
requests from above. What results do you see?

You should see the new values in the responses. Keep in mind that Apigee
maintains a cache of product and app attributes, with a TTL of 180 seconds. So
you may have to wait a bit after modifying the values, before you see them in
the API responses.

### Cleanup

To remove the configuration from this example in your Apigee Organization, first
source your `env.sh` script, and then, in your shell, run:

```bash
./clean-apiproduct-operations.sh
```
