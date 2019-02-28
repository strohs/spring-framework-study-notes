OAuth2.0
====================================
High level overview of OAuth 2.0

* OAuth solves the delegated **authorization** problem:
   * how can I let a website access my data (without giving it my password)
* oauth is also being used to perform authentication even though it was never designed for it:
   * single sign on across sites
   * mobile app login
   * simple login


## Terms
* Resource Owner - the user, that owns the "Data", they can "click yes" and grant access to some resource holding their data
* Client - the application (i.e. Yelp.com, that wants access to your resource, "like a list of contacts from google")
* Authorization Server - the server that can grant authorization to access a resource, (like accounts.google.com)
* Resource Server - the server that actually holds the data a client wants to get to (like the google contacts api)
    * sometimes the resource server and authorization server are the same
* Authorization Grant - proves that a resource owner has granted permission to access their resource
    * there a couple of different Authorization Grant types:
        * `authorization_code` - sent to the Authorization server to signal you want to use an Authorization Code Flow  
* Redirect URI - where to send a user to after they "click yes" and grant access to a resource
* Access Token - a token that is returned to the client and used to get access to the resource server
* Back Channel - highly secure channel, i.e. a trusted server that the client owns, that can contact (ie google) 
securely over HTTPS. There is little chance of credential "leakage"
    * back channel exchanges the authorization `code` for an `access token`. A `secret key` and `clientID` is also sent 
    with the `code` when getting the access token
        * clients will register with the authorization/resource provider ahead of time, in order to get a `clientID` 
        and `secret key`. This data is only known to the client and sent on the back channel in order to get
        the `access token`
* Front Channel - less secure channel, like a web browser. Browsers are typically less secure because we have no
way to control how javascript and HTML code is written by developers. Or there could be a malicious code installed on
 the browser that exposes credentials.
* token exchange - this is where the authorization code is sent (along with clientID and secretKey) to get an Access Token

## Scopes
* clients can request certain scopes from an Authorization Server, these are like permissions given to a client
* the Authorization Server will have a list of scopes that it recognizes, for example:
    * email.read, email.write, contacts, profile,   etc...
    * plus there are some scopes defined by the OAuth specification
* scopes are often used to generate the consent page presented to a resource owner

## OAuth 2 Flows
* Authorization code flow - `response_type=code` 
    * (uses the front channel + back channel) to exchange authorization codes and access tokens
    * once the auth. code is obtained, a request for `grant_type=authorization_code` is made to get the actual
    access token
* implicit flow - `response_type=id_token` 
    * (front channel only), if for some reason you don't have a back channel, this gives you an access
    token immediately. Used by user-agent based client apps that can't securely store a client secret, 
    i.e. javascript single page apps running in a browser

* resource owner credentials password flow - `grant_type=password`
    * (back channel only) - a backend server uses resource owner credentials (usually a username and password) 
    to request an access token directly from the Authorization Server. Used for trusted first party clients. 
    Not a recommended approach
* client credentials flow - `grant_type=client_credentials`
    * (back channel only) - suitable for machine-to-machine authentication where a specific user’s permission 
    to access data is not required. used in micro-services
    
### Which flow (grant type) do I use?
* Web application w/server backend: **authorization code flow**
* Native mobile app: **authorization code flow with ProofKeyCodeExchange (PKCE)**
* javascript app (SPA) w/api backend: **implicit flow**
* micro-services and APIs: **client credentials flow**

## Problems with OAuth 2.0 for authentication
* no standard way to get user's information
* every implementation is a little bit different (impls from Facebook, google, Amazon are all different)
* no common set of scopes

### OpenID Connect
developed to solve Oauth2.0 authentication problems
* OpenId connect is for **Authentication**
    * it is an extension on top of Oauth2.0
* OpenID connect adds:
    * ID Token
        * contains information about the user
    * UserInfo endpoint for getting more user information
    * standard set of scopes
    * standardized implementation
    
#### OpenID Connect Authorization flow
* almost exactly like OAuth2.0 flow
* in the request to the authentication server, the `scope` parameter will have `openid` in it
* then when you go to exchange the authorization code for an access token, you get back an id token as well
* the ID Token can then be immediately consumed by the your application to understand who just logged in
    * it is a JSON Web Token
* access token can be used to call the UserInfo endpoint to get more information about the user