# Azure Active Directory Provider for OAuth 2.0 Client
[![Latest Version](https://img.shields.io/github/release/thenetworg/oauth2-azure.svg?style=flat-square)](https://github.com/thenetworg/oauth2-azure/releases)
[![Total Downloads](https://img.shields.io/packagist/dt/thenetworg/oauth2-azure.svg?style=flat-square)](https://packagist.org/packages/thenetworg/oauth2-azure)
[![Software License](https://img.shields.io/packagist/l/thenetworg/oauth2-azure.svg?style=flat-square)](LICENSE.md)

This package provides [Azure Active Directory](https://azure.microsoft.com/en-us/services/active-directory/) OAuth 2.0 support for the PHP League's [OAuth 2.0 Client](https://github.com/thephpleague/oauth2-client).

## Table of Contents
- [Installation](#installation)
- [Usage](#usage)
    - [Authorization Code Flow](#authorization-code-flow)
        - [Advanced flow](#advanced-flow)
        - [Using custom parameters](#using-custom-parameters)
    - [**NEW** - Logging out](#logging-out)
    - [Scopes](#scopes)
    - [Validating issuer](#validating-issuer)
- [Making API Requests](#making-api-requests)
    - [Variables](#variables)
- [Resource Owner](#resource-owner)
- [Microsoft Graph](#microsoft-graph)
- [**NEW** - Protecting your API - *experimental*](#protecting-your-api---experimental)
- [Azure Active Directory B2C](https://github.com/TheNetworg/oauth2-azure/wiki/Azure-Active-Directory-B2C)
- [Multipurpose refresh tokens](#multipurpose-refresh-tokens)
- [Known users](#known-users)
- [Contributing](#contributing)
- [Credits](#credits)
- [Support](#support)
- [License](#license)

## Installation

To install, use composer:

```
composer require thenetworg/oauth2-azure
```

## Usage

Usage is the same as The League's OAuth client, using `\TheNetworg\OAuth2\Client\Provider\Azure` as the provider.

### Authorization Code Flow

```php
$provider = new TheNetworg\OAuth2\Client\Provider\Azure([
    'clientId'          => '{azure-client-id}',
    'clientSecret'      => '{azure-client-secret}',
    'redirectUri'       => 'https://example.com/callback-url',
    'resource'          => 'https://graph.windows.net/'
]);

if (!isset($_GET['code'])) {

    // If we don't have an authorization code then get one
    $authUrl = $provider->getAuthorizationUrl();
    $_SESSION['oauth2state'] = $provider->getState();
    header('Location: '.$authUrl);
    exit;

// Check given state against previously stored one to mitigate CSRF attack
} elseif (empty($_GET['state']) || ($_GET['state'] !== $_SESSION['oauth2state'])) {

    unset($_SESSION['oauth2state']);
    exit('Invalid state');

} else {

    // Try to get an access token (using the authorization code grant)
    $token = $provider->getAccessToken('authorization_code', [
        'code' => $_GET['code']
    ]);

    // Optional: Now you have a token you can look up a users profile data
    try {

        // We got an access token, let's now get the user's details
        $me = $provider->get("me?api-version=1.6", $token);

        // Use these details to create a new profile
        printf('Hello %s!', $me['givenName']);

    } catch (Exception $e) {

        // Failed to get user details
        exit('Oh dear...');
    }

    // Use this to interact with an API on the users behalf
    echo $token->getToken();
}
```

#### Advanced flow

The [Authorization Code Grant Flow](https://msdn.microsoft.com/en-us/library/azure/dn645542.aspx) is a little bit different for Azure Active Directory. Instead of scopes, you specify the resource which you would like to access - there is a param `$provider->resource` (`null` on default) which will automatically populate the `resource` param of request with its value. If you plan to use v2.0 endpoint of Azure AD or Azure AD B2C you can disregard this and use [scopes](#scopes) (see more [here](https://azure.microsoft.com/en-us/documentation/articles/active-directory-v2-compare/#scopes-not-resources)).

#### Using custom parameters

With [oauth2-client](https://github.com/thephpleague/oauth2-client) of version 1.3.0 and higher, it is now possible to specify custom parameters for the authorization URL, so you can now make use of options like `prompt`, `login_hint` and similar. See the following example of obtaining an authorization URL which will force the user to reauthenticate:
```php
$authUrl = $provider->getAuthorizationUrl([
    'prompt' => 'login'
]);
```
You can find additional parameters [here](https://msdn.microsoft.com/en-us/library/azure/dn645542.aspx).

### Logging out
If you need to quickly generate a logout URL for the user, you can do following:
```php
// Assuming you have provider properly initialized.
$post_logout_redirect_uri = 'https://www.msn.com'; // The logout destination after the user is logged out from their account.
$logoutUrl = $provider->getLogoutUrl($post_logout_redirect_uri);
header('Location: '.$logoutUrl); // Redirect the user to the generated URL
```

### Scopes
When using with Azure AD v2.0 endpoints, you have to specify the [scopes](https://msdn.microsoft.com/library/azure/ad/graph/howto/azure-ad-graph-api-permission-scopes). In order to pass scopes into the authorization URL, you would do following:
```php
$authUrl = $provider->getAuthorizationUrl([
    'scope' => [
        'scope1',
        'scope2',
        ...
    ]
]);
```

### Validating issuer
By default, the library is configured to be multitenant (using the common endpoint). In order to lock the application down to a single tenant, you can set the metadata to only single organization.
```php
$provider = new TheNetworg\OAuth2\Client\Provider\Azure([
    'metadata' => 'https://login.microsoftonline.com/tenant.onmicrosoft.com/.well-known/openid-configuration',
    ...other configuration
]);
```
If you would like to restrict access to multiple organizations, you can do following (using the default, common endpoint):
```php
$provider = new TheNetworg\OAuth2\Client\Provider\Azure([
    'validateIssuer' => false,
    ...other configuration
]);
```
After that, the validation of token's issuer is up to your application logic.

## Making API Requests

This library also provides easy interface to make it easier to interact with [Azure Graph API](https://msdn.microsoft.com/en-us/library/azure/hh974476.aspx) and [Microsoft Graph](http://graph.microsoft.io), the following methods are available on `provider` object (it also handles automatic token refresh flow should it be needed during making the request):

- `get($ref, $accessToken, $headers = [])`
- `post($ref, $body, $accessToken, $headers = [])`
- `put($ref, $body, $accessToken, $headers = [])`
- `delete($ref, $body, $accessToken, $headers = [])`
- `patch($ref, $body, $accessToken, $headers = [])`
- `getObjects($tenant, $ref, $accessToken, $headers = [])` This is used for example for listing large amount of data - where you need to list all users for example - it automatically follows `odata.nextLink` until the end.
  - `$tenant` tenant has to be provided since the `odata.nextLink` doesn't contain it.
  - `$objects` should be either an empty array or a set of data which will be included in the results

*Please note that if you need to create a custom request, the method getAuthenticatedRequest and getResponse can still be used.*

### Variables
- `$ref` The URL reference without the leading `/`, for example `myOrganization/groups`
- `$body` The contents of the request, make has to be either string (so make sure to use `json_encode` to encode the request)s or stream (see [Guzzle HTTP](http://docs.guzzlephp.org/en/latest/request-options.html#body))
- `$accessToken` The access token object obtained by using `getAccessToken` method
- `$headers` Ability to set custom headers for the request (see [Guzzle HTTP](http://docs.guzzlephp.org/en/latest/request-options.html#headers))

## Resource Owner
With version 1.1.0 and onward, the Resource Owner information is parsed from the JWT passed in `access_token` by Azure Active Directory. It exposes few attributes and one function.

**Example:**
```php
$resourceOwner = $provider->getResourceOwner($token);
echo 'Hello, '.$resourceOwner->getFirstName().'!';
```
The exposed attributes and function are:
- `getId()` - Gets user's object id - unique for each user
- `getFirstName()` - Gets user's first name
- `getLastName()` - Gets user's family name/surname
- `getTenantId()` - Gets id of tenant which the user is member of
- `getUpn()` - Gets user's User Principal Name, which can be also used as user's e-mail address
- `claim($name)` - Gets any other claim (specified as `$name`) from the JWT, full list can be found [here](https://azure.microsoft.com/en-us/documentation/articles/active-directory-token-and-claims/)

## Microsoft Graph
Calling [Microsoft Graph](http://graph.microsoft.io/) is very simple with this library. After provider initialization simply change the API URL followingly (replace `v1.0` with your desired version):
```php
$provider->resource = "https://graph.microsoft.com/";
// Authorization stuff
$me = $provider->get('1.0/me', $token);
```
After that, when requesting access token, refresh token or so, provide the `resource` with value `https://graph.microsoft.com/` in order to be able to make calls to the Graph (see more about `resource` [here](#advanced-flow)).

## Protecting your API - *experimental*
With version 1.2.0 and onward you can now use this library to protect your API with Azure Active Directory authentication very easily. The Provider now also exposes `validateAccessToken(string $token)` which lets you pass an access token inside which you for example received in the `Authorization` header of the request on your API. You can use the function followingly (in vanilla PHP):
```php
// Assuming you have already initialized the $provider

// Obtain the accessToken - in this case, we are getting it from Authorization header
$headers = getallheaders();
// Assuming you got the value of Authorization header as "Bearer [the_access_token]" we parse it
$authorization = explode(' ', $headers['Authorization']);
$accessToken = $authorization[1];

try {
    $claims = $provider->validateAccessToken($accessToken);
} catch (Exception $e) {
    // Something happened, handle the error
}

// The access token is valid, you can now proceed with your code. You can also access the $claims as defined in JWT - for example roles, group memberships etc.
```

You may also need to access some other resource from the API like the Microsoft Graph to get some additional information. In order to do that, there is `urn:ietf:params:oauth:grant-type:jwt-bearer` grant available ([RFC](https://tools.ietf.org/html/draft-jones-oauth-jwt-bearer-03)). An example (assuming you have the code above working and you have the required permissions configured correctly in the Azure AD application):
```php
$graphAccessToken = $provider->getAccessToken('jwt_bearer', [
    'resource' => 'https://graph.microsoft.com/',
    'assertion' => $accessToken,
    'requested_token_use' => 'on_behalf_of'
]);

$me = $provider->get('https://graph.microsoft.com/v1.0/me', $graphAccessToken);
print_r($me);
```
Just to make it easier so you don't have to remember entire name for `grant_type` (`urn:ietf:params:oauth:grant-type:jwt-bearer`), you just use short `jwt_bearer` instead.

## Multipurpose refresh tokens
In cause that you need to access multiple resources (like your API and Microsoft Graph), you can use multipurpose [refresh tokens](https://msdn.microsoft.com/en-us/library/azure/dn645538.aspx). Once obtaining a token for first resource, you can simply request another token for different resource like so:
```php
$accessToken2 = $provider->getAccessToken('refresh_token', [
    'refresh_token' => $accessToken1->getRefreshToken(),
    'resource' => 'http://urlOfYourSecondResource'
]);
```
You would do the same for getting token for another resource and so on.

## Known users
If you are using this library and would like to be listed here, please let us know!
- [TheNetworg/DreamSpark-SSO](https://github.com/thenetworg/dreamspark-sso)

## Contributing
We accept contributions via [Pull Requests on Github](https://github.com/thenetworg/oauth2-azure).

## Credits
- [Jan Hajek](https://github.com/hajekj) ([TheNetw.org](https://thenetw.org))
- [Vittorio Bertocci](https://github.com/vibronet) (Microsoft)
    - Thanks for the splendid support while implementing #16
- [All Contributors](https://github.com/thenetworg/oauth2-azure/contributors)

## Support
If you find a bug or encounter any issue or have a problem/question with this library please create a [new issue](https://github.com/TheNetworg/oauth2-azure/issues).

## License
The MIT License (MIT). Please see [License File](https://github.com/thenetworg/oauth2-azure/blob/master/LICENSE) for more information.
