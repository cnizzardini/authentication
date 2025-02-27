Authenticators
##############

Authenticators handle converting request data into an authentication
operations. They leverage :doc:`/identifiers` to find a
known :doc:`/identity-object`.

Session
=======

This authenticator will check the session if it contains user data or
credentials. When using any stateful authenticators like ``Form`` listed
below, be sure to load ``Session`` authenticator first so that once
logged in user data is fetched from session itself on subsequent
requests.

Configuration options:

-  **sessionKey**: The session key for the user data, default is
   ``Auth``
-  **identify**: Set this key with a value of bool ``true`` to enable checking
   the session credentials against the identifiers. When ``true``, the configured
   :doc:`/identifiers` are used to identify the user using data
   stored in the session on each request. Default value is ``false``.
-  **fields**: Allows you to map the ``username`` field to the unique
   identifier in your user storage. Defaults to ``username``. This option is
   used when the ``identify`` option is set to true.

Form
====

Looks up the data in the request body, usually when a form submit
happens via POST / PUT.

Configuration options:

-  **loginUrl**: The login URL, string or array of URLs. Default is
   ``null`` and all pages will be checked.
-  **fields**: Array that maps ``username`` and ``password`` to the
   specified POST data fields.
-  **urlChecker**: The URL checker class or object. Default is
   ``DefaultUrlChecker``.
-  **useRegex**: Whether or not to use regular expressions for URL
   matching. Default is ``false``.
-  **checkFullUrl**: Whether or not to check full URL including the query
   string. Useful when a login form is on a different subdomain. Default is
   ``false``. This option does not work well when preserving unauthenticated
   redirects in the query string.

If you are building an API and want to accept credentials via JSON requests make
sure you have the ``BodyParserMiddleware`` applied **before** the
``AuthenticationMiddleware``.

.. warning::
    If you use the array syntax for the URL, the URL will be
    generated by the CakePHP router. The result **might** differ from what you
    actually have in the request URI depending on your route handling. So
    consider this to be case sensitive!

Token
=====

The token authenticator can authenticate a request based on a token that
comes along with the request in the headers or in the request
parameters.

Configuration options:

-  **queryParam**: Name of the query parameter. Configure it if you want
   to get the token from the query parameters.
-  **header**: Name of the header. Configure it if you want to get the
   token from the header.
-  **tokenPrefix**: The optional token prefix.

An example of getting a token from a header, or query string would be::

    $service->loadAuthenticator('Authentication.Token', [
        'queryParam' => 'token',
        'header' => 'Authorization',
        'tokenPrefix' => 'Token'
    ]);

The above would read the ``token`` GET parameter or the ``Authorization`` header
as long as the token was preceded by ``Token`` and a space.

The token will always be passed to the configured identifier as follows::

    [
        'token' => '{token-value}',
    ]

JWT
===

The JWT authenticator gets the `JWT token <https://jwt.io/>`__ from the
header or query param and either returns the payload directly or passes
it to the identifiers to verify them against another datasource for
example.

-  **header**: The header line to check for the token. The default is
   ``Authorization``.
-  **queryParam**: The query param to check for the token. The default
   is ``token``.
-  **tokenPrefix**: The token prefix. Default is ``bearer``.
-  **algorithm**: The hashing algorithm for Firebase JWT.
   Default is ``'HS256'``.
-  **returnPayload**: To return or not return the token payload directly
   without going through the identifiers. Default is ``true``.
-  **secretKey**: Default is ``null`` but you’re **required** to pass a
   secret key if you’re not in the context of a CakePHP application that
   provides it through ``Security::salt()``.
-  **jwks**: Default is ``null``. Associative array with a ``'keys'`` key.
   If provided will be used instead of the secret key.

You need to add the lib `firebase/php-jwt <https://github.com/firebase/php-jwt>`_
^5.5 to your app to use the ``JwtAuthenticator`` (v6.0 is not currently supported). 

By default the ``JwtAuthenticator`` uses ``HS256`` symmetric key algorithm and uses
the value of ``Cake\Utility\Security::salt()`` as encryption key.
For enhanced security one can instead use the ``RS256`` asymmetric key algorithm.
You can generate the required keys for that as follows::

    # generate private key
    openssl genrsa -out config/jwt.key 1024
    # generate public key
    openssl rsa -in config/jwt.key -outform PEM -pubout -out config/jwt.pem

The ``jwt.key`` file is the private key and should be kept safe.
The ``jwt.pem`` file is the public key. This file should be used when you need to verify tokens
created by external applications, eg: mobile apps.

The following example allows you to identify the user based on the ``sub`` (subject) of the
token by using ``JwtSubject`` identifier, and configures the ``Authenticator`` to use public key
for token verification.

Add the following to your ``Application`` class::

    public function getAuthenticationService(ServerRequestInterface $request): AuthenticationServiceInterface
    {
        $service = new AuthenticationService();
        // ...
        $service->loadIdentifier('Authentication.JwtSubject');
        $service->loadAuthenticator('Authentication.Jwt', [
            'secretKey' => file_get_contents(CONFIG . '/jwt.pem'),
            'algorithm' => 'RS256',
            'returnPayload' => false
        ]);
    }

In your ``UsersController``::

    use Firebase\JWT\JWT;

    public function login()
    {
        $result = $this->Authentication->getResult();
        if ($result->isValid()) {
            $privateKey = file_get_contents(CONFIG . '/jwt.key');
            $user = $result->getData();
            $payload = [
                'iss' => 'myapp',
                'sub' => $user->id,
                'exp' => time() + 60,
            ];
            $json = [
                'token' => JWT::encode($payload, $privateKey, 'RS256'),
            ];
        } else {
            $this->response = $this->response->withStatus(401);
            $json = [];
        }
        $this->set(compact('json'));
        $this->viewBuilder()->setOption('serialize', 'json');
    }

Using a JWKS fetched from an external JWKS endpoint is supported as well::

    // Application.php
    public function getAuthenticationService(ServerRequestInterface $request): AuthenticationServiceInterface
    {
        $service = new AuthenticationService();
        // ...
        $service->loadIdentifier('Authentication.JwtSubject');

        $jwksUrl = 'https://appleid.apple.com/auth/keys';

        // Set of keys. The "keys" key is required. Additionally keys require a "alg" key.
        // Add it manually to your JWK array if it doesn't already exist.
        $jsonWebKeySet = Cache::remember('jwks-' . md5($jwksUrl), function () use ($jwksUrl) {
            $http = new Client();
            $response = $http->get($jwksUrl);
            return $response->getJson();
        });

        $service->loadAuthenticator('Authentication.Jwt', [
            'jwks' => $jsonWebKeySet,
            'returnPayload' => false
        ]);
    }

The JWKS resource will return the same set of keys most of the time.
Applications should cache these resources, but they also need to be
prepared to handle signing key rotations.

.. warning::

    Applications need to pick a cache lifetime that balances performance and security.
    This is particularly important in situations where a private key is compromised.

Beside from sharing the public key file to external application, you can
distribute it via a JWKS endpoint by configuring your app as follows::

    // config/routes.php
    $builder->setExtensions('json');
    $builder->connect('/.well-known/:controller/*', [
        'action' => 'index',
    ], [
        'controller' => '(jwks)',
    ]); // connect /.well-known/jwks.json to JwksController

    // controller/JwksController.php
    public function index()
    {
        $pubKey = file_get_contents(CONFIG . './jwt.pem');
        $res = openssl_pkey_get_public($pubKey);
        $detail = openssl_pkey_get_details($res);
        $key = [
            'kty' => 'RSA',
            'alg' => 'RS256',
            'use' => 'sig',
            'e' => JWT::urlsafeB64Encode($detail['rsa']['e']),
            'n' => JWT::urlsafeB64Encode($detail['rsa']['n']),
        ];
        $keys['keys'][] = $key;

        $this->viewBuilder()->setClassName('Json');
        $this->set(compact('keys'));
        $this->viewBuilder()->setOption('serialize', 'keys');
    }

Refer to https://datatracker.ietf.org/doc/html/rfc7517 or https://auth0.com/docs/tokens/json-web-tokens/json-web-key-sets for
more information about JWKS.

HttpBasic
=========

See https://en.wikipedia.org/wiki/Basic_access_authentication

.. note::

    This authenticator will halt the request when authentication credentials are missing or invalid.

Configuration options:

-  **realm**: Default is ``$_SERVER['SERVER_NAME']`` override it as
   needed.

HttpDigest
==========

See https://en.wikipedia.org/wiki/Digest_access_authentication

.. note::

    This authenticator will halt the request when authentication credentials are missing or invalid.

Configuration options:

-  **realm**: Default is ``null``
-  **qop**: Default is ``auth``
-  **nonce**: Default is ``uniqid(''),``
-  **opaque**: Default is ``null``

Cookie Authenticator aka "Remember Me"
======================================

The Cookie Authenticator allows you to implement the “remember me”
feature for your login forms.

Just make sure your login form has a field that matches the field name
that is configured in this authenticator.

To encrypt and decrypt your cookie make sure you added the
EncryptedCookieMiddleware to your app *before* the
AuthenticationMiddleware.

Configuration options:

-  **rememberMeField**: Default is ``remember_me``
-  **cookie**: Array of cookie options:

   -  **name**: Cookie name, default is ``CookieAuth``
   -  **expires**: Expiration, default is ``null``
   -  **path**: Path, default is ``/``
   -  **domain**: Domain, default is an empty string.
   -  **secure**: Bool, default is ``false``
   -  **httponly**: Bool, default is ``false``
   -  **value**: Value, default is an empty string.
   -  **samesite**: String/null The value for the same site attribute.

   The defaults for the various options besides ``cookie.name`` will be those
   set for the ``Cake\Http\Cookie\Cookie`` class. See `Cookie::setDefaults() <https://api.cakephp.org/4.0/class-Cake.Http.Cookie.Cookie.html#setDefaults>`_
   for the default values.

-  **fields**: Array that maps ``username`` and ``password`` to the
   specified identity fields.
-  **urlChecker**: The URL checker class or object. Default is
   ``DefaultUrlChecker``.
-  **loginUrl**: The login URL, string or array of URLs. Default is
   ``null`` and all pages will be checked.
-  **passwordHasher**: Password hasher to use for token hashing. Default
   is ``DefaultPasswordHasher::class``.
-  **salt**: When ``false`` no salt is used. When a string is passed that value is used as a salt value. 
   When ``true`` the default Security.salt is used. Default is ``true``. When a salt is used, the cookie value 
   will contain `hash(username + password + hmac(username + password, salt))`. This helps harden tokens against possible 
   database leaks and enables cookie values to be invalidated by rotating the salt value.

Usage
-----

The cookie authenticator can be added to a Form & Session based
authentication system. Cookie authentication will automatically re-login users
after their session expires for as long as the cookie is valid. If a user is
explicity logged out via ``AuthenticationComponent::logout()`` the
authentication cookie is **also destroyed**. An example configuration would be::

    // In Application::getAuthService()

    // Reuse fields in multiple authenticators.
    $fields = [
        IdentifierInterface::CREDENTIAL_USERNAME => 'email',
        IdentifierInterface::CREDENTIAL_PASSWORD => 'password',
    ];

    // Put form authentication first so that users can re-login via
    // the login form if necessary.
    $service->loadAuthenticator('Authentication.Form', [
        'loginUrl' => '/users/login',
        'fields' => [
            IdentifierInterface::CREDENTIAL_USERNAME => 'email',
            IdentifierInterface::CREDENTIAL_PASSWORD => 'password',
        ],
    ]);
    // Then use sessions if they are active.
    $service->loadAuthenticator('Authentication.Session');

    // If the user is on the login page, check for a cookie as well.
    $service->loadAuthenticator('Authentication.Cookie', [
        'fields' => $fields,
        'loginUrl' => '/users/login',
    ]);

You'll also need to add a checkbox to your login form to have cookies created::

    // In your login view
    <?= $this->Form->control('remember_me', ['type' => 'checkbox']);

After logging in, if the checkbox was checked you should see a ``CookieAuth``
cookie in your browser dev tools. The cookie stores the username field and
a hashed token that is used to reauthenticate later.

Events
======

There is only one event that is fired by authentication:
``Authentication.afterIdentify``.

If you don’t know what events are and how to use them `check the
documentation <https://book.cakephp.org/4/en/core-libraries/events.html>`__.

The ``Authentication.afterIdentify`` event is fired by the
``AuthenticationComponent`` after an identity was successfully
identified.

The event contains the following data:

-  **provider**: An object that implements
   ``\Authentication\Authenticator\AuthenticatorInterface``
-  **identity**: An object that implements ``\ArrayAccess``
-  **service**: An object that implements
   ``\Authentication\AuthenticationServiceInterface``

The subject of the event will be the current controller instance the
AuthenticationComponent is attached to.

But the event is only fired if the authenticator that was used to
identify the identity is *not* persistent and *not* stateless. The
reason for this is that the event would be fired every time because the
session authenticator or token for example would trigger it every time
for every request.

From the included authenticators only the FormAuthenticator will cause
the event to be fired. After that the session authenticator will provide
the identity.

URL Checkers
============

Some authenticators like ``Form`` or ``Cookie`` should be executed only
on certain pages like ``/login`` page. This can be achieved using URL
Checkers.

By default a ``DefaultUrlChecker`` is used, which uses string URLs for
comparison with support for regex check.

Configuration options:

-  **useRegex**: Whether or not to use regular expressions for URL
   matching. Default is ``false``.
-  **checkFullUrl**: Whether or not to check full URL. Useful when a
   login form is on a different subdomain. Default is ``false``.

A custom URL checker can be implemented for example if a support for
framework specific URLs is needed. In this case the
``Authentication\UrlChecker\UrlCheckerInterface`` should
be implemented.

For more details about URL Checkers :doc:`see this documentation
page </url-checkers>`.

Getting the Successful Authenticator or Identifier
==================================================

After a user has been authenticated you may want to inspect or interact with the
Authenticator that successfully authenticated the user::

    // In a controller action
    $service = $this->request->getAttribute('authentication');

    // Will be null on authentication failure, or an authenticator.
    $authenticator = $service->getAuthenticationProvider();

You can also get the identifier that identified the user as well::

    // In a controller action
    $service = $this->request->getAttribute('authentication');

    // Will be null on authentication failure, or an identifier.
    $identifier = $service->getIdentificationProvider();


Using Stateless Authenticators with Stateful Authenticators
===========================================================

When using ``HttpBasic``, ``HttpDigest`` with other authenticators,
you should remember that these authenticators will halt the request when
authentication credentials are missing or invalid. This is necessary as these
authenticators must send specific challenge headers in the response::

    use Authentication\AuthenticationService;

    // Instantiate the service
    $service = new AuthenticationService();

    // Load identifiers
    $service->loadIdentifier('Authentication.Password', [
        'fields' => [
            'username' => 'email',
            'password' => 'password'
        ]
    ]);
    $service->loadIdentifier('Authentication.Token');

    // Load the authenticators leaving Basic as the last one.
    $service->loadAuthenticator('Authentication.Session');
    $service->loadAuthenticator('Authentication.Form');
    $service->loadAuthenticator('Authentication.HttpBasic');

If you want to combine ``HttpBasic`` or ``HttpDigest`` with other
authenticators, be aware that these authenticators will abort the request and
force a browser dialog.

Handling Unauthenticated Errors
================================

The ``AuthenticationComponent`` will raise an exception when users are not
authenticated. You can convert this exception into a redirect using the
``unauthenticatedRedirect`` when configuring the ``AuthenticationService``.

You can also pass the current request target URI as a query parameter
using the ``queryParam`` option::

   // In the getAuthenticationService() method of your src/Application.php

   $service = new AuthenticationService();

   // Configure unauthenticated redirect
   $service->setConfig([
       'unauthenticatedRedirect' => '/users/login',
       'queryParam' => 'redirect',
   ]);

Then in your controller's login method you can use ``getLoginRedirect()`` to get
the redirect target safely from the query string parameter::

    public function login()
    {
        $result = $this->Authentication->getResult();

        // Regardless of POST or GET, redirect if user is logged in
        if ($result->isValid()) {
            // Use the redirect parameter if present.
            $target = $this->Authentication->getLoginRedirect();
            if (!$target) {
                $target = ['controller' => 'Pages', 'action' => 'display', 'home'];
            }
            return $this->redirect($target);
        }
    }

Having Multiple Authentication Flows
====================================

In an application that provides both an API and a web interface
you may want different authentication configurations based on
whether the request is an API request or not. For example, you may use JWT
authentication for your API, but sessions for your web interface. To support
this flow you can return different authentication services based on the URL
path, or any other request attribute::

    public function getAuthenticationService(
        ServerRequestInterface $request
    ): AuthenticationServiceInterface {
        $service = new AuthenticationService();

        // Configuration common to both the API and web goes here.

        if ($request->getParam('prefix') == 'Api') {
            // Include API specific authenticators
        } else {
            // Web UI specific authenticators.
        }

        return $service;
    }
