.. role:: javascript(code)
  :language: javascript

==============
Stitch SDK API
==============

:Spec: 1
:Title: Stitch SDK API
:Authors: Adam Chelminski
:Advisors: Jason Flax, Eric Daniels
:Status: Approved
:Type: Standards
:Minimum Server Version: cloud-2.0.0
:Last Modified: July 5, 2018

.. contents::

--------

Abstract
========

MongoDB Stitch supports client interactions over an HTTP client API. This specification defines the how a Stitch client SDK should use and expose this API backend. As not all languages/environments have the same abilities, parts of the spec may or may not apply. These sections have been called out.

Specification
=============

This specification is about `Guidance`_ for the developer-facing API of a Stitch SDK. Other than specifying the HTTP endpoints that the SDK needs to use on the Stitch server, it does not define internal implementation structure and provides room and flexibility for the idioms and differences in languages and frameworks.

-----------
Definitions
-----------

META
----

The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in `RFC 2119 <https://www.ietf.org/rfc/rfc2119.txt>`_.

Terms
-----

Stitch
  The MongoDB Stitch backend-as-a-service backend server.

Client SDK
  A software library for a particular language or platform that provides access to MongoDB Stitch functionality made available by the Stitch HTTP client API.

Client App ID
  The unique identification string required by clients to access their application and its services.

Authentication Provider
  An authentication principal in Stitch that can accept credentials to create a new Stitch user from an identity, or authenticate an existing identity. In either case, after successfully authenticating, Stitch issues access tokens and refresh tokens that the client SDK can use to make authenticated requests as a particular Stitch user. Examples of authentication providers include the username/password provider, and the Facebook OAuth2 provider.

Service
  Any third party extension that is supported by Stitch as a "service" in the Stitch UI.

Mobile Device
  Any reference to a device using the iOS and/or Android platforms, natively or otherwise.

Push Notification
  A message sent to a mobile device by an external messaging service (e.g. Firebase Cloud Messaging). The mobile device can handle the message in any way it wants. Typically, the message is used to display a notification on the device.

Push Provider
  An endpoint in the Stitch client API which can be used to register a Stitch user for push notifications from an external messaging service.

End-User Developer
  A person using a client SDK to build client applications with MongoDB Stitch.


--------
Guidance
--------

Documentation
-------------
The documentation provided in code below is merely for SDK authors and SHOULD NOT be taken as required documentation for the SDK.


Operations & Properties
-----------------------
All SDKs MUST offer the operations and properties defined in the following sections unless otherwise specified. This does not preclude an SDK from offering more.

Operation Parameters
--------------------
All SDKs MUST offer the same options for each operation as defined in the following sections. This does not preclude a SDKs from offering more. An SDK SHOULD NOT require a user to specify optional parameters, denoted by the Optional<> signature. Unless otherwise specified, optional values should not be sent to the Stitch server.

~~~~~~~~~~
Deviations
~~~~~~~~~~

A non-exhaustive list of acceptable deviations are as follows:

- Using named parameters instead of an options hash. For instance, ``collection.find({x:1}, sort: {a: -1})``.

- When using an ``Options`` class, if multiple ``Options`` classes are structurally equatable, it is permissible to consolidate them into one with a clear name. For instance, it would be permissible to use the name ``UpdateOptions`` as the options for ``UpdateOne`` and ``UpdateMany``.

- Using a fluent style builder for find or aggregate:

  .. code:: typescript

    collection.find({x: 1}).sort({a: -1}).skip(10);

  When using a fluent-style builder, all options should be named rather than inventing a new word to include in the pipeline (like options). Required parameters are still required to be on the initiating method.

  In addition, it is imperative that documentation indicate when the order of operations is important. For instance, skip and limit in find is order-irrelevant where skip and limit in aggregate is order-relevant.

Naming
------

All SDKs MUST name operations, objects, and parameters as defined in the following sections.

Deviations are permitted as outlined below.


~~~~~~~~~~
Deviations
~~~~~~~~~~

When deviating from a defined name, an SDKauthor should consider if the altered name is recognizable and discoverable to the user of another SDK.

A non-exhaustive list of acceptable naming deviations are as follows:

- Using the property "loggedIn" as an example, Kotlin would use "loggedIn", while Java would use "isLoggedIn()". However, calling it "isAuthenticated" would not be acceptable. Some languages idioms prefer the use of "is", "has", or "was" and this is acceptable.
- Using the method "loginWithCredential" as an example, Java would use "loginWithCredential", Swift would use "login(withCredential: ...", and Python would use "login_with_credential. However, calling it "loginWithSecret" would not be acceptable.

--------------
Client SDK API
--------------

This section describes how a client SDK should communicate with Stitch and expose its functionality. The section will provide room and flexibility for the idioms and differences in languages and frameworks.

Many of the top-level headers in this section should be made available as a language-appropriate structure that can hold state and expose methods and properties. (e.g. class or interface with class implementation in Java, class or protocol with class/struct implementation in Swift).

For the purposes of this section, we will use the terms "interface" and "object", but appropriate language constructs can be substituted for each SDK.

If a method in one of these interfaces is marked as ASYNC ALLOWED, the method SHOULD be implemented to return its result in an asynchronous manner if it is appropriate for the environment. The mechanism for this will depend on the platform and environment (e.g. via Promises in ES6, Tasks for Android, closure callbacks in iOS). However, some environments may not require or desire methods with asynchronous behavior (e.g. Java Server SDK). 

If a method is marked as ERROR POSSIBLE, the method MUST be written to cleanly result in an error when there is a server error, request error, or other invalid state. The mechanism for error handling will depend on the the language and environment, as well as whether the method is implemented synchronously or asynchronously. See the section on `Error Handling`_ for more information.

When methods contain parameters that are wrapped in an optional type, the method can be overloaded to have variants that don’t accept the parameter at all.

Many of the methods in this section require a request to be made to the Stitch server. See `Mechanism for Making Requests`_ for specific details on how to construct these requests.

Stitch
------

An SDK MUST have a ``Stitch`` interface which serves as the entry-point for initializing and retrieving client objects. The interface is responsible for statically storing initialized app clients. If a language has a multithreaded model, the implementation of this interface SHOULD be thread safe. It it cannot be made in such a way, the documentation MUST state it. The following methods MUST be provided, unless otherwise specified in the comment for a particular method:

.. code:: typescript

  interface Stitch {
      /**
       * (OPTIONAL)
       *
       * Initialize the Stitch SDK so that app clients can properly report 
       * device information to the Stitch server.
       *
       * This method should only be implemented for environments where the
       * initialization requires access to a platform-specific context object.
       * (e.g. android.content.Context in the Android SDK)
       *
       * If appropriate and possible for the environment, this method MAY be
       * called automatically when the user includes the SDK.
       */
      static initialize(context: PlatformSpecificContext): void

      /**
       * (REQUIRED, ERROR POSSIBLE)
       *
       * Initialize an app client for a specific app and configuration.
       * The client initialized by this method will be retrievable by
       * the getDefaultAppClient and getAppClient methods. If this method is
       * called more than once, it should result in a language-appropriate 
       * error, as only one default app client should ever be specified.
       *
       * If no configuration is specified, a default configuration should
       * be used. See the sections on Client Configuration for the properties
       * of a default configuration. 
       *
       * If appropriate and possible for the environment, this method MAY be
       * called automatically when the user includes the SDK.
       */
      static initializeDefaultAppClient(
          clientAppId: string,
          config: Optional<StitchAppClientConfiguration>
      ): StitchAppClient

      
      /**
       * (REQUIRED, ERROR POSSIBLE)
       *
       * Initialize an app client for a specific app and configuration.
       * The client initialized by this method will be retrievable by
       * the getAppClient method. If this method is called more than
       * once for a specific client app ID, it should result in a
       * language-appropriate error, as only one app client should be specified
       * for each client app ID.
       *
       * If no configuration is specified, a default configuration should
       * be used. See the sections on Client Configuration for the properties
       * of a default configuration.
       *
       * If appropriate and possible for the environment, this method MAY be
       * called automatically when the user includes the SDK.
       */
      static initializeAppClient(
          clientAppId: string,
          config: Optional<StitchAppClientConfiguration>
      ): StitchAppClient

      /**
       * (REQUIRED, ERROR POSSIBLE)
       *
       * Gets the default initialized app client. If one has not been set, then
       * a language-appropriate error should be thrown/returned.
       */
      static getDefaultAppClient(): StitchAppClient

      /**
       * (REQUIRED, ERROR POSSIBLE)
       *
       * Gets an app client by its client app ID if it has been initialized;
       * should result in a language-appropriate error if none can be found.
       */
      static getAppClient(clientAppId: string): StitchAppClient
  }

StitchAppClient
---------------

An SDK MUST have a ``StitchAppClient`` interface, which serves as the primary means of communicating with the Stitch server. The following methods MUST be provided, unless otherwise specified in the comment for a particular method:

.. code:: typescript

  interface StitchAppClient {

      /**
       * (REQUIRED)
       *
       * Gets a StitchAuth object which can be used to view and modify the
       * authentication status of this Stitch client.
       */
      getAuth(): StitchAuth

      /**
       * (OPTIONAL)
       *
       * Gets a StitchPush object which can be used to get push provider clients 
       * which can be used to subscribe the currently authenticated user for
       * push notifications from an external messaging system. MUST be
       * implemented in SDKs intended for mobile device platforms.
       */
      getPush(): StitchPush

      /**
       * (REQUIRED - see "Factories" for exceptions) 
       *
       * Gets a client for a particular named Stitch service.
       * See the "Factories" section for details on the factory type.
       */
      getServiceClient<T>(
          factory: NamedServiceClientFactory<T>, 
          serviceName: string
      ): T

      /**
       * (REQUIRED - see "Factories" for exceptions)
       *
       * Gets a client for a particular Stitch service
       * See the "Factories" section for details on the factory type.
       */
      getServiceClient<T>(factory: ServiceClientFactory<T>): T

      /**
       * (REQUIRED, ASYNC ALLOWED, ERROR POSSIBLE) 
       *
       * Calls the function in MongoDB Stitch with the provided name
       * and arguments. If no error occurs in carrying out the request, the 
       * extended JSON response by the Stitch server should be decoded into 
       * the type T.
       *
       * SHOULD also accept additional arguments to modify the request timeout,      
       * and to provide a mechanism for decoding.
       */
      callFunction<T>(name: string, args: List<BsonValue>): T

      /**
       * (REQUIRED, ASYNC ALLOWED, ERROR POSSIBLE)
       *
       * Calls the function in MongoDB Stitch with the provided name
       * and arguments. If no error occurs in carrying out the request, the 
       * response by the Stitch server should be ignored.
       * 
       * SHOULD also accept an additional argument to modify the request 
       * timeout.
       */
      callFunction(name: string, args: List<BsonValue>): void
  }

For the methods that make network requests, the following list enumerates how each of the requests should be constructed, as well as the shapes of the responses from the Stitch server:

*  ``callFunction``

   -  **Authenticated**: yes, with access token
   -  **Endpoint**: ``POST /api/client/v2.0/app/<client_app_id>/functions/call``
   -  **Request Body**: 

      + 
        ::
            
            {
                "name": (name argument),
                "arguments": (args argument)
            }

      + The arguments field in the request body MUST be encoded as canonical extended JSON. See the specification on `MongoDB Extended JSON <https://github.com/mongodb/specifications/blob/master/source/extended-json.rst>`_ for more information.

   -  **Response Shape**:

      + The MongoDB Extended JSON representation of the called Stitch function's return value.


~~~~~~~~~~~~~~~
Service Clients
~~~~~~~~~~~~~~~

MongoDB Stitch exposes much of its functionality via "Services". Services provide access via the Stitch server to MongoDB's own services such as MongoDB Atlas, as well access to third-party partner services such as Twilio and AWS S3. Each service should have a service client constructible from a ``StitchAppClient`` that exposes its functionality.

This specification does not cover exactly how these services should be exposed to end-user developers. However, for each available service there may be a specification that describes how its functionality should be exposed.

Each service client MUST be packaged independently of a Stitch client SDK's main package, and offered as a pluggable module to that particular client SDK. The pluggable module MUST be compatible with the main client SDK as described in the `Factories`_ section.

In general, service clients will take the form of an interface providing methods that call the service's available functions. The code below shows what a sample service client with two methods may look like.

.. code:: typescript

  interface SampleServiceClient {
      /** 
       * (ASYNC ALLOWED, ERROR POSSIBLE)
       */
      sampleServiceFunction(sampleArgument: string): number

      /**  
       * (ASYNC ALLOWED, ERROR POSSIBLE)
       */
      otherSampleServiceFunction(sampleArgument: number): string
  }

Each method of a service client can call the service's available functions with the following request.

*  service function call request

   -  **Authenticated**: yes, with access token
   -  **Endpoint**: ``POST /api/client/v2.0/app/<client_app_id>/functions/call``
   -  **Request Body**: 

      + 
        ::
            
            {
                "service": (name of the service in Stitch),
                "name": (name of the service function),
                "arguments": (arguments for the service function)
            }

      + The arguments field in the request body MUST be encoded as canonical extended JSON. See the specification on `MongoDB Extended JSON <https://github.com/mongodb/specifications/blob/master/source/extended-json.rst>`_ for more information. The contents of the arguments field will depend on the service function being called. See the specification for a particular service for more details about what arguments are expected by a specific service function.

   -  **Response Shape**:

      + The shape of the response will depend on the service function being called, but will generally be in MongoDB Extended JSON if not empty. See the specification for a particular service for more details about what a specific service function returns.

Note that this request is almost identical to the request for a normal Stitch function, with the addition of the ``"service"`` field to the request body.


StitchAuth
----------
An SDK MUST have a ``StitchAuth`` interface, which serves as the primary means of authenticating with Stitch and viewing authentication status. A ``StitchAuth`` is considered part of a client, and the term "client" will refer to the combined functionality of the ``StitchAuth`` and the parent ``StitchAppClient``. The following methods and properties MUST be provided, unless otherwise specified in the comment for a particular method:

.. code:: typescript

  interface StitchAuth {
      /**
       * (REQUIRED - see "Factories" for exceptions)
       *
       * Gets a client for a particular authentication provider.
       * See the "Factories" section for details on the factory type.
       */
      getProviderClient<T>(factory: AuthProviderClientFactory<T>): T

      /**
       * (REQUIRED - see "Factories" for exceptions)
       *
       * Gets a client for a particular named authentication provider and 
       * provider name. See the "Factories" section for details on the 
       * factory type.
       */
      getProviderClient<T>(factory: AuthProviderClientFactory<T>, 
                           providerName: string): T

      /**
       * (REQUIRED, ASYNC ALLOWED, ERROR POSSIBLE)
       *
       * Authenticates the Stitch client using the provided credential.
       * If the login is successful, additionally fetch the profile of the user.
       */
      loginWithCredential(credential: StitchCredential): StitchUser

      /**
       * (REQUIRED, ASYNC ALLOWED)
       *
       * Logs out the currently logged in user by clearing authentication
       * tokens locally, and sending a request to the Stitch server to 
       * invalidate the session. If the request fails, the error should be 
       * ignored and the method should still succeed.
       */
      logout(): void

      /**
       * (REQUIRED)
       *
       * Whether or not the client is currently authenticated as a Stitch user.
       */
      loggedIn: boolean

      /**
       * (REQUIRED)
       *
       * A StitchUser object representing the Stitch user that the
       * client is currently authenticated as. If the client is not
       * authenticated, this should return an empty optional.
       */
      user: Optional<StitchUser>

      /**
       * (OPTIONAL) 
       *
       * Registers a listener whose onAuthEvent method should be invoked
       * whenever an authentication event occurs on this client. An 
       * authentication event is defined as one of the following:
       *     - a user is logged in
       *     - a user is logged out
       *     - a user is linked to another identity
       *     - a listener is registered
       */
      addAuthListener(listener: StitchAuthListener): void

      /**
       * (OPTIONAL)
       *
       * Unregisters a listener from this client.
       */
      removeAuthListener(listener: StitchAuthListener): void    
  }

For the methods that make network requests, the following list enumerates how each of the requests should be constructed, as well as the shapes of the responses from the Stitch server:

*  ``loginWithCredential`` - initial request

   -  **Authenticated**: no
   -  **Endpoint**: ``POST /api/client/v2.0/app/<client_app_id>/auth/providers/<provider_name>/login``
   -  **Request Body**: 

      + The material of the credential as an extended JSON document, (see `Authentication Credentials`_), merged with the following document: 
        ::
            
            {
                "options": {
                    "device": {
                        (device information document)
                    }
                }
            }

      + The device information document contains the following key-value pairs:

        +-----------------+------------------------------+--------------------------+
        | Key             | Value                        |                          |
        +-----------------+------------------------------+--------------------------+
        | deviceId        | The device_id if one is      | REQUIRED unless omitted  |
        |                 | persisted, omitted otherwise | because no device ID is  |
        |                 |                              | persisted                |
        +-----------------+------------------------------+--------------------------+
        | appId           | The name of the              | RECOMMENDED              |
        |                 | local application            |                          |
        +-----------------+------------------------------+--------------------------+
        | appVersion      | The version of the           | RECOMMENDED              |
        |                 | local application            |                          |
        +-----------------+------------------------------+--------------------------+
        | platform        | The platform of the          | REQUIRED                 |
        |                 | SDK (e.g. "Android",         |                          |
        |                 | "iOS", etc.)                 |                          |
        +-----------------+------------------------------+--------------------------+
        | platformVersion | The version of the           | REQUIRED                 |
        |                 | SDK’s platform.              |                          |
        +-----------------+------------------------------+--------------------------+
        | sdkVersion      | The version of the           | REQUIRED                 |
        |                 | SDK.                         |                          |
        +-----------------+------------------------------+--------------------------+

   -  **Response Shape**:

      +
        ::

            {
                "access_token": (string),
                "user_id": (string),
                "device_id": (string),
                "refresh_token": (string)
            }
   -  **Behavior**:

      + The ``StitchAuth`` is responsible for persisting the authentication information returned in the response (``access_token`` and ``refresh_token``) so that it can be used to make authenticated requests on behalf of the newly logged in user. The ``user_id`` and ``device_id`` should also be persisted so they can returned be as part of the ``StitchAuth``’s user property.

      + If a user is already logged in when the call to ``loginWithCredential`` is made, the existing user MUST be logged out, unless the ``providerCapabilities`` property of the credential specifies that ``reusesExistingSession`` is true, and the the provider type of the credential is the same as the provider type of the currently logged in user.

*  ``loginWithCredential`` - profile request

   -  **Authenticated**: yes, with access token 
   -  **Endpoint**: ``GET /api/client/v2.0/app/<client_app_id>/auth/profile``
   -  **Response Shape**:

      + Base:
        ::

            {
                "type": (string),
                "data": (subdocument of key-string pairs),
                "identities": (array of identity objects)
            }

      + Identity:
        ::

            {
                "id": (string),
                "provider_type": (string)
            }

   -  **Behavior**:

      + If the profile request fails, the currently authenticated user should be logged out, and the error should be thrown/returned. If the request succeeds, the contents of the response should be persisted such that ``StitchAuth`` will return a fully populated ``StitchUser`` for its user property.

*  ``logout``

   -  **Authenticated**: yes, with refresh token
   -  **Endpoint**: ``DELETE /api/client/v2.0/app/<client_app_id>/auth/session``
   -  **Response Shape**:

      + Empty

   -  **Behavior**:

      + Even if this request fails, the currently logged in user should still be logged out by deleting the persisted authentication information. The error MAY be logged, but an error MUST NOT be thrown or returned. The request only serves to invalidate the user’s tokens.


StitchAuthListener
------------------
An SDK MAY have a ``StitchAuthListener`` interface, which is an interface that end-user developers can inherit to perform actions that will occur whenever an authentication event occurs in an application. ``StitchAuthListener`` objects can be registered with a ``StitchAuth`` if the ``StitchAuth`` interface implements the ``addAuthListener`` method. The following methods MUST be provided if ``StitchAuthListener`` is implemented:

.. code:: typescript

  interface StitchAuthListener {
    /**
     * (REQUIRED) 
     *
     * To be called any time a notable event regarding authentication happens.
     * These events include:
     * - When a user logs in.
     * - When a user logs out.
     * - When a user is linked to another identity.
     * - When a listener is registered.
     *
     * The auth parameter is the instance of StitchAuth where the event
     * happened. It should be used to infer the current state of
     * authentication.
     */
    onAuthEvent(auth: StitchAuth): void
  }

StitchPush
----------

An SDK MAY have a ``StitchPush`` interface, which is used for producing push provider clients. Push provider clients may be used by a Stitch user to subscribe for push notifications from an external messaging system. The following methods MUST be provided if ``StitchPush`` is implemented:

.. code:: typescript

  interface StitchPush {

      /**
       * (REQUIRED - see "Factories" for exceptions)
       *
       * Gets a push provider client for a particular named push provider 
       * in Stitch. See the "Factories" section for details on the factory type.
       */
      getClient<T>(factory: NamedPushClientFactory<T>, serviceName: string): T
  }


~~~~~~~~~~~~
Push Clients
~~~~~~~~~~~~

The purpose of a push provider client is to register a Stitch user for push notifications that may be sent by another Stitch user or from the Stitch admin console. A push client does not necessarily set up the device to receive the notifications, because that functionality will generally require the use of a third-party SDK from a third-party messaging service.

More commonly, the third-party messaging service will provide a "registration token" or some other unique identifying token for the device, and that token needs to be registered with the currently logged in Stitch user’s device so that push notifications sent to a particular user are also sent to the device with that registration token.

A sample push client implementation is as follows:

.. code:: typescript

  interface SampleServicePushClient {
      /**
       * (ASYNC ALLOWED, ERROR POSSIBLE)
       *
       * Registers the given registration token with the currently 
       * logged in user’s device on Stitch.
       */
      register(registrationToken: string): void

      /**
       * (ASYNC ALLOWED, ERROR POSSIBLE)
       *
       * Deregisters the registration token bound to the currently 
       * logged in user's device on Stitch.
       */
      deregister(): void
  }

Push provider clients MUST be offered as pluggable modules and not as part of an SDK's main package, similarly to how `Service Clients`_ must be provided in a pluggable way.


Client Configuration
--------------------

As discussed in the specification for the ``Stitch`` interface, Stitch clients should be configurable beyond just the client app ID.  The interfaces here define the configuration settings that are required to be available for an SDK. If appropriate and idiomatic for the target language, a builder should also be specified for each of these interfaces.


~~~~~~~~~~~~~~~~~~~~~~~~~
StitchClientConfiguration
~~~~~~~~~~~~~~~~~~~~~~~~~

An SDK MUST have a ``StitchClientConfiguration`` interface, which defines the low-level settings of how a client should communicate with Stitch and store data. The following properties MUST be provided:

.. code:: typescript

  interface StitchClientConfiguration {
      /**
       * (REQUIRED)
       *
       * The base URL of the Stitch server that the client will communicate
       * with. By default, this should be "https://stitch.mongodb.com".
       */
      baseUrl: string

      /**
       * (REQUIRED)
       * 
       * A simple key-value store abstraction that will be used to persist
       * authentication information, and potentially other data in the future.
       * By default, this should be an abstraction of a platform-appropriate 
       * persistence layer (e.g. UserDefaults on iOS, LocalStorage in the 
       * browser, SharedPreferences on Android).
       */
      storage: Storage

      /**
       * (RECOMMENDED)
       *
       * A local directory in which Stitch can store any data (e.g. embedded 
       * MongoDB data directory, authentication information). If the platform
       * does not have the concept of a local directory (e.g. browser), this may
       * be omitted.
       */
      dataDirectory: string

      /**
       * (REQUIRED)
       *
       * A simple HTTP round-trip abstraction that will be used to make HTTP 
       * requests on behalf of the client.  By default, this should be an 
       * abstraction of a platform-appropriate HTTP transport utility (e.g. 
       * URLSession on iOS, fetch in JavaScript, OkHttp in Android).
       */
      transport: Transport

      /**
       * (REQUIRED)
       * 
       * The default amount of time that a request should wait before it is 
       * considered timed out. This should passed as part of the request object
       * to the Transport. TimeIntervalType refers to the 
       * platform-idiomatic representation of a time interval (e.g.
       * TimeInterval on iOS, Long in Java). By default, this interval should
       * be 15 seconds.
       */
      defaultRequestTimeout: TimeIntervalType
  }

Additional properties MAY be included if necessary and appropriate for the target environment/language. For example, a Java-based SDK could offer a codec registry type to be used for decoding responses from the Stitch server.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~
StitchAppClientConfiguration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

An SDK MUST have a ``StitchAppClientConfiguration`` interface, which defines the local app information that the client should provide when it reports device information to the Stitch server. The ``StitchAppClientConfiguration`` must also inherit ``StitchClientConfiguration``. The ``Stitch`` interface should be responsible for providing defaults for these properties and inherited properties when no configuration is specified. The following properties MUST be provided:

.. code:: typescript

  interface StitchAppClientConfiguration: StitchClientConfiguration {

      /**
       * (REQUIRED)
       *
       * The name of the local application, as it should be reported
       * to the Stitch server. By default, the Stitch interface should
       * attempt to infer this information from platform-specific context.
       */
      localAppName: string

      /**
       * (REQUIRED)
       *
       * The version of the local application, as it should be reported
       * to the Stitch server. By default, the Stitch interface should
       * attempt to infer this information from platform-specific context.
       */
      localAppVersion: string
  }

Additional properties MAY be included if necessary and appropriate for the target environment/language.


User Information
----------------

~~~~~~~~~~
StitchUser
~~~~~~~~~~

An SDK must have a ``StitchUser`` interface, which exposes properties about a Stitch user, and offers functionality for linking that user to a new identity. The following methods and properties MUST be provided:

.. code:: typescript

  interface StitchUser {
      /**
       * (REQUIRED, ASYNC ALLOWED, ERROR POSSIBLE)
       *
       * Links this user with a new identity, using the provided credential.
       * If the linking is successful, the method also attempts to update the
       * user profile by fetching the latest user profile from the Stitch
       * server.
       */
      linkWithCredential(credential: StitchCredential): StitchUser

      /**
       * (REQUIRED)
       *
       * The id of this Stitch user.
       */
      id: string

      /**
       * (REQUIRED)
       *
       * The string representing the type of authentication provider 
       * used to log in as this user.
       */
      loggedInProviderType: string

      /**
       * (REQUIRED)
       *
       * The name of the authentication provider used to log in as this user.
       */
      loggedInProviderName: string

      /**
       * (REQUIRED)
       *
       * The type of this user ("normal" for normal users, or "server" for users
       * authenticated using the server API key authentication provider).
       */
      userType: string (or UserType enum)

      /**
       * (REQUIRED)
       *
       * A profile containing basic information about the user.
       */
      profile: StitchUserProfile

      /**
       * (REQUIRED)
       *
       * A list of the identities associated with this user.
       */
      identities: List<StitchUserIdentity>
  }

For the methods that make network requests, the following list enumerates how each of the requests should be constructed, as well as the shapes of the responses from the Stitch server:

*  ``linkWithCredential`` - initial request

   -  **Authenticated**: yes, with access token
   -  **Endpoint**: ``POST /api/client/v2.0/app/<client_app_id>/auth/providers/<provider_name>/login?link=true``
   -  **Request Body**: 

      + The material of the credential as an extended JSON document, (see `Authentication Credentials`_), merged with the following document: 
        ::
            
            {
                "options": {
                    "device": {
                        (device information document)
                    }
                }
            }

      + The contents of the device information document are covered in `StitchAuth`_.

   -  **Response Shape**:

      +
        ::

            {
                "access_token": (string),
                "user_id": (string)
            }

   -  **Behavior**:

      + The ``access_token`` in the response should be persisted as it is the most up-to-date access token.

*  ``linkWithCredential`` - profile request

   -  Identical to ``loginWithCredential``‘s profile request (covered in `StitchAuth`_), except that if the profile request fails, the currently logged in user should remain logged in even though an error is thrown or returned.


~~~~~~~~~~~~~~~~~
StitchUserProfile
~~~~~~~~~~~~~~~~~

An SDK must have a ``StitchUserProfile`` interface, which exposes basic profile information about a Stitch user. The following properties MUST be provided. The fields in this interface should be populated using the ``data`` field of the profile response from the Stitch server.

.. code:: typescript

  interface StitchUserProfile {

      /**
       * (REQUIRED)
       *
       * The full name of this user.
       */
      name: Optional<string>

      /**
       * (REQUIRED)
       *
       * The email address of this user.
       */
      email: Optional<string>

      /**
       * (REQUIRED)
       *
       * A URL to a profile picture of this user.
       */
      pictureUrl: Optional<string>

      /**
       * (REQUIRED)
       *
       * The first name of this user.
       */
      firstName: Optional<string>

      /**
       * (REQUIRED)
       *
       * The last name of this user.
       */
      lastName: Optional<string>

      /**
       * (REQUIRED)
       *
       * The gender of this user.
       */
      gender: Optional<string>

      /**
       * (REQUIRED)
       *
       * The birthdate of this user.
       */
      birthday: Optional<string>

      /**
       * (REQUIRED)
       *
       * The minimum age of this user (some social authentication providers,
       * such as Facebook and Google provide the age of a user as a range rather
       * than an exact number).
       */
      minAge: Optional<number>

      /**
       * (REQUIRED)
       *
       * The maximum age of this user (some social authentication providers,
       * such as Facebook and Google provide the age of a user as a range rather
       * than an exact number).
       */
      maxAge: Optional<number>
  }


~~~~~~~~~~~~~~~~~~
StitchUserIdentity
~~~~~~~~~~~~~~~~~~

An SDK must have a ``StitchUserIdentity`` interface, which exposes information about a Stitch user identity. The following properties MUST be provided:

.. code:: typescript

  interface StitchUserIdentity {
      /**
       * (REQUIRED)
       *
       * The id of this identity. This is NOT the id of the user. This is 
       * generally an opaque value that should not be used.
       */
      id: string

      /**
       * (REQUIRED)
       *
       * The type of authentication provider that this identity is for. A user
       * may be linked to multiple identities of the same type. This value is 
       * useful to check to determine if a user has registered with a certain
       * provider yet.
       */
      providerType: string
  }


Factories
---------

When appropriate and possible for an SDK’s language and environment, the SDK MUST support the construction of authentication provider clients and service clients with a factory approach so that ``StitchAppClient`` and ``StitchAuth`` can be as modular as possible. 

The exact mechanics of the factory will depend on the language and environment, but in general, the factory should be a generic type with the constructed client type as the generic type parameter.

When factories are supported by an SDK, all of the following factories MUST be offered:

* ``AuthProviderClientFactory`` - for unnamed authentication providers
* ``NamedAuthProviderClientFactory`` - for named authentication providers
* ``ServiceClientFactory`` - for unnamed services
* ``NamedServiceClientFactory`` - for named services

If push provider clients are supported by an SDK and factories are also supported, the following factory MUST be offered to support ``StitchPush``:

* ``NamedPushClientFactory`` - for push providers

For an example implementation of the factory approach, see the reference implementation of the SDK `in Java <https://github.com/mongodb/stitch-android-sdk>`_.

If a language or environment does not support this factory approach, the SDK MUST use an alternate approach to maintain modularity. An acceptable alternative approach is to include a ``StitchAppClient``, ``StitchAuth``, or ``StitchPush`` as a parameter to the constructor/initializer of the service client type or authentication provider client type. The following examples demonstrates this alternative approach in pseudocode:

.. code:: typescript

  class SomeServiceClient { 
      constructor(appClient: StitchAppClient, serviceName: string)
  }

  class SomeAuthProviderClient { 
      constructor(auth: StitchAuth)
  }

  class SomePushProviderClient { 
      constructor(push: StitchPush)
  }


Authentication Credentials
--------------------------

An SDK MUST have a ``StitchCredential`` interface that is accepted as a parameter by StitchAuth and StitchUser for authentication methods such as ``loginWithCredential`` and ``linkWithCredential``. The ``StitchCredential`` type should not be meant to be instantiated directly, but via a subclass implementation specific to an authentication provider. This section will cover the required interface for each of these types.


~~~~~~~~~~~~~~~~
StitchCredential
~~~~~~~~~~~~~~~~

This is the base credential type that is accepted by login and link methods, and MUST provide the following properties:

.. code:: typescript

  interface StitchCredential {
      /**
       * (REQUIRED)
       *
       * The name of the authentication provider being authenticated with.
       */
      providerName: string

      /**
       * (REQUIRED)
       *
       * The string denoting the type of the authentication provider being 
       * authenticated with.
       */
      providerType: string

      /**
       * (REQUIRED)
       *
       * A BSON document containing the credential contents of the credential. 
       * The subsections describing the specific credential types for each
       * authentication provider list the required fields for each 
       * authentication provider type.
       */
      material: BsonDocument

      /**
       * (REQUIRED)
       *
       * An interface describing the behavior that the credential should
       * exhibit when authenticating.
       */
      providerCapabilities: ProviderCapabilities
  }


ProviderCapabilities
^^^^^^^^^^^^^^^^^^^^

The ``ProviderCapabilities`` type describes the behavior that a credential should exhibit when authenticating. The following properties MUST be provided:

.. code:: typescript

  interface ProviderCapabilities {

      /**
       * (REQUIRED)
       *
       * When true, a StitchAuth using this credential to login should skip 
       * authentication and reuse existing authentication information when 
       * attempting to login with the same authentication provider as the 
       * already authenticated user.
       */
      reusesExistingSession: boolean    
  }


~~~~~~~~~~~~~~~~~~~
AnonymousCredential
~~~~~~~~~~~~~~~~~~~

An SDK MUST have an ``AnonymousCredential`` interface which supports logging in as an anonymous user. The following constructor MUST be provided:

.. code:: typescript

  interface AnonymousCredential: StitchCredential {
      constructor()
  }

The following table enumerates the properties that an ``AnonymousCredential`` should have when inheriting the ``StitchCredential`` interface:

+----------------------+-----------------------------------------------+
| providerName         | "anon-user"                                   |
+----------------------+-----------------------------------------------+
| providerType         | "anon-user"                                   |
+----------------------+-----------------------------------------------+
| material             | { }                                           |
+----------------------+-----------------------------------------------+
| providerCapabilities | { reusesExistingSession: true }               |
+----------------------+-----------------------------------------------+


~~~~~~~~~~~~~~~~
CustomCredential
~~~~~~~~~~~~~~~~

An SDK MUST have a ``CustomCredential`` interface which supports logging in with or linking to an identity from a custom authentication system. The following constructor MUST be provided:

.. code:: typescript

  interface CustomCredential: StitchCredential {
      constructor(token: string)
  }

The following table enumerates the properties that a ``CustomCredential`` should have when inheriting the ``StitchCredential`` interface:

+----------------------+-----------------------------------------------+
| providerName         | "custom-token"                                |
+----------------------+-----------------------------------------------+
| providerType         | "custom-token"                                |
+----------------------+-----------------------------------------------+
| material             | { "token": tokenFromConstructor }             |
+----------------------+-----------------------------------------------+
| providerCapabilities | { reusesExistingSession: false }              |
+----------------------+-----------------------------------------------+

~~~~~~~~~~~~~~~~~~
FacebookCredential
~~~~~~~~~~~~~~~~~~

An SDK MUST have a ``FacebookCredential`` interface which supports logging in with or linking to an identity via the Facebook Login API. The following constructor MUST be provided:

.. code:: typescript

  interface FacebookCredential: StitchCredential {
      constructor(accessToken: string)
  }

The following table enumerates the properties that a ``FacebookCredential`` should have when inheriting the ``StitchCredential`` interface:

+----------------------+-----------------------------------------------+
| providerName         | "oauth2-facebook"                             |
+----------------------+-----------------------------------------------+
| providerType         | "oauth2-facebook"                             |
+----------------------+-----------------------------------------------+
| material             | { "accessToken": accessTokenFromConstructor } |
+----------------------+-----------------------------------------------+
| providerCapabilities | { reusesExistingSession: false }              |
+----------------------+-----------------------------------------------+


~~~~~~~~~~~~~~~~
GoogleCredential
~~~~~~~~~~~~~~~~

An SDK MUST have a ``GoogleCredential`` interface which supports logging in with or linking to an identity via the Google Sign-In API. The following constructor MUST be provided:

.. code:: typescript

  interface GoogleCredential: StitchCredential {
      constructor(authCode: string)
  }

The following table enumerates the properties that a ``GoogleCredential`` should have when inheriting the ``StitchCredential`` interface:

+----------------------+-----------------------------------------------+
| providerName         | "oauth2-google"                               |
+----------------------+-----------------------------------------------+
| providerType         | "oauth2-google"                               |
+----------------------+-----------------------------------------------+
| material             | { "authCode": authCodeFromConstructor }       |
+----------------------+-----------------------------------------------+
| providerCapabilities | { reusesExistingSession: false }              |
+----------------------+-----------------------------------------------+


~~~~~~~~~~~~~~~~~~~~~~
ServerApiKeyCredential
~~~~~~~~~~~~~~~~~~~~~~

An SDK MUST have a ``ServerApiKeyCredential`` interface which supports logging in with a server API key created in the Stitch admin console. The following constructor MUST be provided:

.. code:: typescript

  interface ServerApiKeyCredential: StitchCredential {
      constructor(key: string)
  }

The following table enumerates the properties that a ``ServerApiKeyCredential`` should have when inheriting the ``StitchCredential`` interface:

+----------------------+-----------------------------------------------+
| providerName         | "api-key"                                     |
+----------------------+-----------------------------------------------+
| providerType         | "api-key"                                     |
+----------------------+-----------------------------------------------+
| material             | { "key": keyFromConstructor }                 |
+----------------------+-----------------------------------------------+
| providerCapabilities | { reusesExistingSession: false }              |
+----------------------+-----------------------------------------------+


~~~~~~~~~~~~~~~~~~~~
UserApiKeyCredential
~~~~~~~~~~~~~~~~~~~~

An SDK MUST have a ``UserApiKeyCredential`` interface which supports logging in with a user API key. The following constructor MUST be provided:

.. code:: typescript

  interface UserApiKeyCredential: StitchCredential {
      constructor(key: string)
  }

The following table enumerates the properties that a ``UserApiKeyCredential`` should have when inheriting the ``StitchCredential`` interface:

+----------------------+-----------------------------------------------+
| providerName         | "api-key"                                     |
+----------------------+-----------------------------------------------+
| providerType         | "api-key"                                     |
+----------------------+-----------------------------------------------+
| material             | { "key": keyFromConstructor }                 |
+----------------------+-----------------------------------------------+
| providerCapabilities | { reusesExistingSession: false }              |
+----------------------+-----------------------------------------------+


~~~~~~~~~~~~~~~~~~~~~~
UserPasswordCredential
~~~~~~~~~~~~~~~~~~~~~~

An SDK MUST have a ``UserPasswordCredential`` interface which supports logging in with or linking to an identity using an email address and password. The following constructor MUST be provided:

.. code:: typescript

  interface UserPasswordCredential: StitchCredential {
      constructor(username: string, password: string)
  }

The following table enumerates the properties that a ``UserPasswordCredential`` should have when inheriting the ``StitchCredential`` interface:

+----------------------+-------------------------------------------------------------------------------+
| providerName         | "local-userpass"                                                              |
+----------------------+-------------------------------------------------------------------------------+
| providerType         | "local-userpass"                                                              |
+----------------------+-------------------------------------------------------------------------------+
| material             | {  "username": usernameFromConstructor, "password": passwordFromConstructor } |
+----------------------+-------------------------------------------------------------------------------+
| providerCapabilities | { reusesExistingSession: false }                                              |
+----------------------+-------------------------------------------------------------------------------+


Authentication Provider Clients
-------------------------------

~~~~~~~~~~~~~~~~~~~~~~~~~~~~
UserApiKeyAuthProviderClient
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

An SDK MUST have a ``UserApiKeyAuthProviderClient`` interface which supports the creation, modification, and deletion of user API keys. The ``UserApiKeyAuthProviderClient`` MUST be constructible by the ``getProviderClient`` method on ``StitchAuth`` using a factory, or with an acceptable alternative approach where appropriate (see `Factories`_ for details).

The following methods MUST be provided:

.. code:: typescript

  interface UserApiKeyAuthProviderClient {
      /**
       * (REQUIRED, ASYNC ALLOWED, ERROR POSSIBLE)
       *
       * Creates a user API key which can be used to authenticate as the 
       * current user. Returns a UserApiKey with the key string specified.
       */
      createApiKey(name: string): UserApiKey

      /**
       * (REQUIRED, ASYNC ALLOWED, ERROR POSSIBLE)
       *
       * Feteches a user API key associated with the current user, using the 
       * specified key id.
       */
      fetchApiKey(id: BsonObjectId): UserApiKey

      /**
       * (REQUIRED, ASYNC ALLOWED, ERROR POSSIBLE)
       *
       * Fetches all of the user API keys associated with the current user.
       */
      fetchApiKeys(): List<UserApiKey>

      /**
       * (REQUIRED, ASYNC ALLOWED, ERROR POSSIBLE)
       *
       * Deletes a user API key associated with the current user, using the
       * specified key id.
       */
      deleteApiKey(id: BsonObjectId): void

      /**
       * (REQUIRED, ASYNC ALLOWED, ERROR POSSIBLE)
       *
       * Enables a user API key associated with the current user, using the
       * specified key id.
       */
      enableApiKey(id: BsonObjectId): void

      /**
       * (REQUIRED, ASYNC ALLOWED, ERROR POSSIBLE)
       *
       * Disables a user API key associated with the current user, using the
       * specified key id.
       */
      disableApiKey(id: BsonObjectId): void
  }

For the methods that make network requests, the following list enumerates how each of the requests should be constructed, as well as the shapes of the responses from the Stitch server:

*  ``createApiKey``

   -  **Authenticated**: yes, with refresh token
   -  **Endpoint**: ``POST /api/client/v2.0/app/<client_app_id>/auth/api_keys``
   -  **Request Body**: 

      + 
        ::
            
            {
                "name": (name argument)
            }

   -  **Response Shape**:

      +
        ::

            {
                "_id": (string),
                "key": (string),
                "name": (string),
                "disabled": (boolean)
            }
   -  **Behavior**:

      + A ``UserApiKey`` object should be constructed using the contents of the response.

*  ``fetchApiKey``

   -  **Authenticated**: yes, with refresh token
   -  **Endpoint**: ``GET /api/client/v2.0/app/<client_app_id>/auth/api_keys/<key_id>``
   -  **Response Shape**:

      +
        ::

            {
                "_id": (string),
                "name": (string),
                "disabled": (boolean)
            }
   -  **Behavior**:

      + A ``UserApiKey`` object should be constructed using the contents of the response.

*  ``fetchApiKeys``

   -  **Authenticated**: yes, with refresh token
   -  **Endpoint**: ``GET /api/client/v2.0/app/<client_app_id>/auth/api_keys``
   -  **Response Shape**:

      +
        ::

            [{
                "_id": (string),
                "name": (string),
                "disabled": (boolean)
            }, ...]
   -  **Behavior**:

      + A list of ``UserApiKey`` objects should be constructed using the contents of the response.

*  ``deleteApiKey``

   -  **Authenticated**: yes, with refresh token
   -  **Endpoint**: ``DELETE /api/client/v2.0/app/<client_app_id>/auth/api_keys/<key_id>``
   -  **Response Shape**:

      + Empty

*  ``enableApiKey``

   -  **Authenticated**: yes, with refresh token
   -  **Endpoint**: ``PUT /api/client/v2.0/app/<client_app_id>/auth/api_keys/<key_id>/enable``
   -  **Response Shape**:

      + Empty

*  ``disableApiKey``

   -  **Authenticated**: yes, with refresh token
   -  **Endpoint**: ``PUT /api/client/v2.0/app/<client_app_id>/auth/api_keys/<key_id>/disable``
   -  **Response Shape**:

      + Empty


UserApiKey
^^^^^^^^^^

An SDK MUST have a ``UserApiKey`` interface which represents a user API key (a key created by a Stitch user to sign in as that user via the user API key authentication provider). The following properties MUST be provided:

.. code:: typescript

  interface UserApiKey {

      /**
       * (REQUIRED)
       *
       * The id of this API key.
       */
      id: BsonObjectId

      /**
       * (REQUIRED)
       *
       * The actual API key. This should only be a non-empty optional when the 
       * API key is first created. Fetched API keys should always have an empty
       * optional for their key property.
       */
      key: Optional<string>

      /**
       * (REQUIRED)
       *
       * The name of this API key.
       */
      name: string

      /**
       * (REQUIRED)
       *
       * Whether or not this API key is currently disabled for login usage.
       */
      disabled: boolean
  }


~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
UserPasswordAuthProviderClient
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

An SDK MUST have a ``UserPasswordAuthProviderClient`` interface which exposes the functionality of the username/password authentication provider related to creating and recovering user identities associated with an email address. The ``UserPasswordAuthProviderClient`` MUST be constructible by the ``getProviderClient`` method on ``StitchAuth`` using a factory, or with an acceptable alternative approach where appropriate (see `Factories`_ for details).

The following methods MUST be provided:

.. code:: typescript

  interface UserPasswordAuthProviderClient {
      /**
       * (REQUIRED, ASYNC ALLOWED, ERROR POSSIBLE)
       *
       * Registers a new identity with the username/password authentication
       * provider. This creates an identity, but no Stitch user will be created
       * unless the identity is used to log in as a new user before it is used 
       * to link to an existing user.
       */
      registerWithEmail(email: string, password: string): void

      /**
       * (REQUIRED, ASYNC ALLOWED, ERROR POSSIBLE)
       *
       * Confirms a newly registered user identity with the token and
       * token id that were sent to the newly registered email.
       */
      confirmUser(token: string, tokenId: string): void

      /**
       * (REQUIRED, ASYNC ALLOWED, ERROR POSSIBLE)
       *
       * Resends the confirmation email for a newly registered identity.
       */
      resendConfirmationEmail(email: string): void

      /**
       * (REQUIRED, ASYNC ALLOWED, ERROR POSSIBLE)
       *
       * Resets the password of an existing username/password identity with
       * the token and token id that were sent in the password reset email.
       */
      resetPassword(token: string, tokenId: string, password: string): void

      /**
       * (REQUIRED, ASYNC ALLOWED, ERROR POSSIBLE)
       *
       * Sends a password reset email to a given email address associated 
       * with an existing identity.
       */
      sendResetPasswordEmail(email: string)
  }

For the methods that make network requests, the following list enumerates how each of the requests should be constructed, as well as the shapes of the responses from the Stitch server:

*  ``registerWithEmail``

   -  **Authenticated**: no
   -  **Endpoint**: ``POST /api/client/v2.0/app/<client_app_id>/auth/providers/local-userpass/register``
   -  **Request Body**: 

      + 
        ::
            
            {
                "email": (email argument),
                "password": (password argument)
            }

   -  **Response Shape**:

      + Empty

*  ``confirmUser``

   -  **Authenticated**: no
   -  **Endpoint**: ``POST /api/client/v2.0/app/<client_app_id>/auth/providers/local-userpass/confirm``
   -  **Request Body**: 

      + 
        ::
            
            {
                "token": (token argument),
                "tokenId": (tokenId argument)
            }

   -  **Response Shape**:

      + Empty

*  ``resendConfirmationEmail``

   -  **Authenticated**: no
   -  **Endpoint**: ``POST /api/client/v2.0/app/<client_app_id>/auth/providers/local-userpass/confirm/send``
   -  **Request Body**: 

      + 
        ::
            
            {
                "email": (email argument)
            }

   -  **Response Shape**:

      + Empty

*  ``resetPassword``

   -  **Authenticated**: no
   -  **Endpoint**: ``POST /api/client/v2.0/app/<client_app_id>/auth/providers/local-userpass/reset``
   -  **Request Body**: 

      + 
        ::
            
            {
                "token": (token argument),
                "tokenId": (tokenId argument),
                "password": (password argument)
            }

   -  **Response Shape**:

      + Empty

*  ``sendResetPasswordEmail``

   -  **Authenticated**: no
   -  **Endpoint**: ``POST /api/client/v2.0/app/<client_app_id>/auth/providers/local-userpass/reset/send``
   -  **Request Body**: 

      + 
        ::
            
            {
                "email": (email argument)
            }

   -  **Response Shape**:

      + Empty


Mechanism for Making Requests
-----------------------------

A Stitch SDK provides its core functionality by making HTTP requests to the Stitch server. Throughout this specification, there are descriptions of how requests should be made for certain methods. This section describes in detail how the requests should be structured and carried out based on those descriptions.

~~~~~~~~
Endpoint
~~~~~~~~

Every request has an endpoint to which the request should be made. This endpoint should be appended to the base URL configured in the Stitch client. By default, this base URL is ``https://stitch.mongodb.com``.

~~~~~~~~~~~~~~
Authentication
~~~~~~~~~~~~~~

A request to the Stitch server can either be made on behalf of no user (an "unauthenticated request"), or on behalf of the client’s currently authenticated user (an "authenticated request"). 

Unauthenticated requests are generally used for requests that are made when no user is logged in (e.g. login, user registration, password reset), and authenticated requests are generally used for requests that are made when a user is logged in (e.g. profile retrieval, Stitch function calls, logout).

For unauthenticated requests, an ``Authorization`` header MUST NOT be included.

For authenticated requests, an ``Authorization`` header MUST be included. The contents of this header depend on whether the request uses a refresh token or an access token. A refresh token is a permanent (until invalidated) token, whereas an access token is for temporary use and expires after 30 minutes.

The description for each request in this specification specifies the type of token that should be included. The contents of the ``Authorization`` header should be one of the following:

* ``Bearer <access_token>``
* ``Bearer <refresh_token>``

The token should be retrieved from the authentication information that a ``StitchAuth`` persisted when ``loginWithCredential`` or ``linkWithCredential`` was called, or when an access token was refreshed. If no user is currently logged in, the client should throw/return a client error.

When an authenticated request is completed, it is possible that the response will contain a service error with the error code ``InvalidSession``. This denotes that the access token or refresh token provided for the request is no longer valid because it expired or was invalidated. If this service error is in the response to an authenticated request made using an access token, the client MUST attempt to to refresh the access token and retry the request once using the new access token.

An access token can be refreshed with the following request:

*  refresh token request

   -  **Authenticated**: yes, using refresh token
   -  **Endpoint**: ``POST /api/client/v2.0/app/<client_app_id>/auth/session``
   -  **Request Body**: None
   -  **Response Shape**:

      +
        ::

            {
                "access_token": (string)
            }
   -  **Behavior**:

      + If the refresh request fails, an invalid session service error should be thrown for the original request, and the current user MUST be logged out by clearing persisted authentication information. If the refresh succeeds, the new access token should be persisted and the original request MUST be retried once and only once.


Proactive Token Refresher
^^^^^^^^^^^^^^^^^^^^^^^^^

In addition to automatically retrying requests when they fail due to an invalid session, clients SHOULD have a mechanism for proactively refreshing expired access tokens in the background. Access tokens are stored as JWT strings (see `RFC 7519 <https://tools.ietf.org/html/rfc7519>`_), thus expiration time can be checked on the client side without making any network requests.

How a client implements this mechanism will depend on the language and environment, and is ultimately at the discretion of the SDK author. For example, the reference implementation Swift SDK for iOS implements proactive token refresh by running a background thread that checks for access token expiration every 60 seconds. If the token is expired, the thread makes the refresh token request described in the parent section.

Other languages and environments will have different mechanisms for periodically running tasks in the background, and in some environments this may be infeasible. In environments where a background task is infeasible, it is RECOMMENDED to proactively check for token expiration and before making any request that uses an access token.


~~~~~~~~~~~~
Request Body
~~~~~~~~~~~~

Most ``POST`` requests made to the Stitch server also require a JSON request body to be included. When a request body is included, the client MUST also include the following header:

* ``Content-Type: application/json``


~~~~~~~~~~~~~~
Response Shape
~~~~~~~~~~~~~~

Many of the requests made to the Stitch server will contain a non-empty response. This specification provides the expected shape of the response for each request. The shape provided assumes a successful request. Responses denoting a service error will be structured differently, and this structure is described in the `Error Handling`_ section.


~~~~~~~~
Behavior 
~~~~~~~~

Most methods in the Stitch SDK API will require additional tasks to be performed after the request is complete and the response is received (or if the request failed for any reason). This specification describes any additional behavior that the client must exhibit once a request is completed.

Error Handling
--------------

Since a Stitch SDK makes network requests, it is inherently prone to errors. Errors may occur for a number of reasons, but in general there are three classes of errors that a Stitch SDK should naturally handle: service errors, request errors, and client errors. Each of these types of errors are described in this section.

A Stitch SDK MUST support a way of representing these errors to the end-user developers using the SDK. Since different languages and environments support error handling in vastly different ways, the way of representing and throwing errors is at the discretion of the SDK author, but the following general guidelines SHOULD be followed:

*  There should be an overarching Stitch error type from which all other error types inherit or are composed of. This allows Stitch errors to be handled in a unified way. This type should be called ``StitchError`` or ``StitchException`` depending on the idioms of the language.

   + Example: In the reference implementation `Java SDK <https://github.com/mongodb/stitch-android-sdk>`_, ``StitchException`` is the parent class for ``StitchServiceException``, ``StitchRequestException``, and ``StitchClientException``.
   + Example: In the reference implementation `Swift SDK <https://github.com/mongodb/stitch-ios-sdk>`_, ``StitchError`` is an enum with cases for ``.serviceError``, ``.requestError``, and ``.clientError``.

*  If a language or environment supports constraining the type of an error that is thrown or returned, the SDK should constrain errors returned by SDK methods to be of the overarching ``StitchError``/``StitchException`` type.

   + Example: In the reference implementation `Swift SDK for iOS <https://github.com/mongodb/stitch-ios-sdk>`_, the asynchronous methods that communicate with Stitch accept a callback to handle the result of a request. The callbacks contain a result that may contain a ``StitchError`` if the method failed for any reason.

The next few subsections describe the different classes of errors that a Stitch SDK MUST naturally handle and represent to the end-user developer.

~~~~~~~~~~~~~~
Service Errors
~~~~~~~~~~~~~~

Service errors are errors that are returned by the Stitch server after a request is completed, with an error message and error code. Service errors generally occur (but are not limited to occuring) when something is misconfigured on the Stitch server or parameters to a function or endpoint are invalid.

The response body of a service error will most likely be in the following format:

::

  {
      "error": (string containing error message),
      "error_code": (string denoting error code)
  }

The SDK MUST parse this response to produce an error interface containing an error message and error code. An SDK SHOULD represent the possible error codes as an enumeration. The reference implementations of the SDK will have the latest list of possible error codes, but the enumeration should always include the ``Unknown`` case for unrecognized codes or improperly constructed responses.

If the response is not in this format, the produced error interface should use the entire response body as the error message, and ``Unknown`` as the error code. For example, if the response body is the following:

::

  404 page not found

The produced error interface should have the error message "404 page not found", and the error code ``Unknown``.


~~~~~~~~~~~~~~
Request Errors
~~~~~~~~~~~~~~

Request errors are errors that occur while encoding, carrying out, or decoding a request. Request errors typically result from another component of the SDK throwing an error/exception. This could be the transport throwing a timeout error, or the response decoder throwing a decoding exception because the response was in an unexpected format.

The following list enumerates the error codes that should be provided and when they should used (naming can be adjusted to be idiomatic for a particular language/environment) :

*  ``TransportError``

   + A ``TransportError`` should be thrown/returned when the underlying transport for an HTTP request throws/returns an error. Reasons an underlying transport may throw an error include but are not limited to network timeouts or an unreachable server.

*  ``EncodingError``

   + An ``EncodingError`` should be thrown/returned when there is a failure in encoding a request body into JSON. In general, if an SDK is implemented correctly, this error should never occur. This type of error should only occur if there is a mistake in the application code and a non-encodable value is passed as an argument to a Stitch function.

*  ``DecodingError``

   + A ``DecodingError`` should be thrown/returned when there is a failure in decoding the response into the desired internal model or expected Stitch function return value.

*  ``UnknownError``

   + An ``UnknownError`` should be thrown/returned when an error occurs that is unrelated to any of the other error codes. This type of error should be uncommon.

An SDK MAY include additional error codes if a language or environment has a common type of request error that doesn’t fall under one of the above error codes.

When constructing a representation of a request error, the interface should contain the underlying error/exception object, along with the error code.


~~~~~~~~~~~~~
Client Errors
~~~~~~~~~~~~~

Client errors are errors that occur because the client is misconfigured, is used incorrectly, or is in an invalid state. The representation of these errors should contain an error code. The reasons that a client may result in an error will depend on the language and environment, but all SDKs should have the following error codes:

*  ``LoggedOutDuringRequest``

   + Should be thrown/returned if a client is logged out when attempting to refresh an access token.

*  ``MustAuthenticateFirst``

   + Should be thrown/returned if a client attempts to make an authenticated request without being logged in.

*  ``UserNoLongerValid``

   + Should be thrown/returned if a client attempts to use a ``StitchUser`` object to link to a new identity when that ``StitchUser`` has already been logged out.

*  ``CouldNotLoadPersistedAuthInfo``

   + Should be thrown/returned if a client fails to load persisted authentication information when attempting to make an authenticated request.

*  ``CouldNotPersistAuthInfo``

   + Should be thrown/returned if a client fails to persist authentication information after a successful login, link, or access token refresh request.

An SDK MAY define additional client error codes if appropriate for the language, environment, or internal client implementation.


Test Plan
=========

See `Reference Implementations`_


Motivation
==========

Polyglot developers, documentation authors, and support engineers working on multi-platform applications built on top of MongoDB Stitch may become frustrated and confused if different platforms have different idioms and semantics for communicating with MongoDB Stitch. Their jobs can be made easier if there is a unified specification for how SDKs should be structured and behave.


Backwards Compatibility
=======================

The specification should be mostly backwards compatible with respect to the v4.0.0 Java, Swift, and TypeScript SDKs. Slight modifications (including minor breaking changes) may be necessary to reach full specification compliance. Backwards compatibility with v3.0.0 SDKs was not a goal of this specification as it would require major breaking changes.


Reference Implementations
=========================

The following SDKs (officially supported by MongoDB) are provided as reference implementations of this specification:

:Java Android SDK: https://github.com/mongodb/stitch-android-sdk/tree/master/android
:Java Server SDK: https://github.com/mongodb/stitch-android-sdk/tree/master/server
:Swift iOS SDK: https://github.com/mongodb/stitch-ios-sdk
:JavaScript Browser SDK: https://github.com/mongodb/stitch-js-sdk/tree/master/packages/browser/sdk
:JavaScript Node.js SDK: https://github.com/mongodb/stitch-js-sdk/tree/master/packages/server/sdk

Although this specification doesn’t define requirements for internal implementation structure, we recommend that new SDKs base their structure on one of these reference implementations as their modular structure makes it easy to extend the SDK to support new MongoDB Stitch features.

Each reference implementation is also comprehensively tested, and their tests constitute the test plan for this specification.


Q & A
=====

This section will be updated with frequently asked questions from end-user developers and SDK authors.


Changes
=======

- 2018-07-06: Initial draft
