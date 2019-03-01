Spring Security
============================
Some additional notes on Spring Security high level architecture


* 3 Key areas in Spring Security
    * Authentication
    * Authorization
    * Exception Handling
* Spring Security package name is: `org.springframework.security.*`
* Spring security is implemented as a series of servlet filters, (implemented by Spring Beans) that intercept 
incoming requests and perform specific tasks on them: such as authentication, authorization, exception 
handling/mapping, CSRF processing, Clearing the Security Context, HTTPBasic Header processing, 
ExceptionTranslationFilter... 

## Authentication
* is one of the primary interfaces in the spring security package 
* represents the "token" for an authentication request or authenticated principal once the request has been processed
by the `AuthenticationManager`
* some of the concrete implementations of this interface include:
    * `UserNamePasswordAuthenticationToken, AnonymousAuthenticationToken, OAuth2AuthenticationToken ...`


## Authentication Request Flow
this flow is applicable to any other authentication pattern you may use. HTTP Basic is shown here as it is a simple
example.

### 1. AuthenticationFilter
* first stage in processing incoming, browser-based HTTP authentication requests. (i.e.HTTP Basic)
* see the class: `Class AbstractAuthenticationProcessingFilter` for the general overview of these filters
* AuthenticationFilters perform the following tasks:
    1. extract username/password from the servlet request
    2. create a `UsernamePasswordAuthenticationToken` (an instance of the `Authentication` interface)
    3. pass the token to the `AuthenticationManager` for actual authentication

### 2. Authentication
* the authentication token is passed to an `AuthenticationManager` for authentication
* AuthenticationManager delegates to the `ProviderManager` which calls one or more `AuthenticationProvider`(s) to perform 
the actual authentication
* an `AuthenticationProvider` (such as `DaoAuthenticationProvider`) will attempt to authenticate the token and
return an `Authentication` instance with an "authenticated principal" stored within it and the authenticated flag set
to true

### 3. AuthenticationProvider calls UserDetails/UserDetailsService
* UserDetails and UserDetailsService are interfaces which load user-specific data from some data store
    * some examples are: `JdbcDaoImpl`,  `InMemoryUserDetailsManager`,`LdapUserDetailsManager` OR you can write your own
* these interfaces are used by the DaoAuthenticationProvider
* the AuthenticationProvider will call `UserDetailsService.loadByUserName()` to locate a user by username from the
underlying datastore
* `UserDetails` are returned (containing userName, password, and authorities) to the AuthenticationProvider 
(ie DaoAuthenticationProvider) which will then attempt to authenticate using these UserDetails
    * if authenticated, a **new** Authentication object (ie `UserNamePasswordAuthenticationToken`) is created with
    the UserDetails stored within it as the "authenticated principal". The UserDetails can be retrieved via a call
    to `Authentication.getPrincipal()` which returns type Object

### 4. Security Context
* the newly created Authentication object (from section 3 above) is returned by the AuthenticationManager to the
AuthenticationFilter
* the AuthenticationFilter will store the Authentication in the `SecurityContext` which is then stored in a
`SecurityContextHolder`
* the `SecurityContext` interface is simply a "holder" for context information related to security
    * i.e. `SecurityContext getAuthentication()` returns `Authentication`
* `SecurityContextHolder` is stored as a ThreadLocal variable and can be accessed using the following call:
    * `SecurityContextHolder.getContext().setAuthentication(authenticated)`
    * it is used by many Security Filters to get the current SecurityContext so that we don't pass the Authentication
    or `SecurityContext` as method parameters to different filters

### Authentication Request Flow (for HTTP Basic) Summary
* AuthenticationFilter creates an *AuthenticationRequest* and passes it to the `AuthenticationMananger`
* `AuthenticationManager` delegates to the `AuthenticationProvider(s)`
* `AuthenticationProvider` uses a `UserDetailsService` to load `UserDetails` and return it as a **Authenticated Principal**
    * note that you don't have to use a UserDetailsService, you can provide your own custom logic
* Authentication Filter sets the `Authentication` in the `SecurityContext`


## Authorization Request Flow

### FilterSecurityInterceptor
* the last filter in the security filter chain, basically checks to see if you are authorized to access some endpoint
    * ie. a Request is made to access: "/messages/inbox"
* the FilterSecurityInterceptor calls `securityMetadataSource.match(request)` to see if your request matches
the configured security rules
    * the SecurityMetadataSource contains your "Ant matcher" rules from your security config
    * i.e.  `.authorizeRequests().antMatchers( "/messages/**" ).hasRole( "ROLE_USER" )`
* if there is a match between the request and a pattern in the securityMetadataSource, then the current 
authentication is retrieved from the `SecurityContextHolder`
* a call is made to the `AccessDecisionManager`, passing to it the current `Authentication`,the `HttpServletRequest` and
 the `SecurityMetadata` to determine if you can access the resource

### AccessDecision
* `AccessDecisionManager` takes the input parameters and calls one or more `AccessDecisionVoter(s)` to determine
if the principal has the authority to access the resource

### Authorization Summary
* `FilterSecurityInterceptor` obtains the Security Metadata and uses it to match on the current request
* FilterSecurityInterceptor gets the current Authentication
* the Authentication, SecurityMetadata, and Request is passed to the `AccessDecisionManager`
* the AccessDecisionManager delegates to `AccessDecisionVoter(s)` for "decisioning"


## Exception Handling
this is an exception handling for HTTP Basic. There is also HTTP Form Login that is commonly used

### Access Denied Flow - User is authenticated, but doesn't have authority to access a resource
* user has ROLE_USER but is trying to access "/admin/messages" 
    * spring security is configured with  `.antMatchers("/admin/**").hasRole("ROLE_ADMIN")`
* FilterSecurityInterceptor will call the AccessDecisionManager which calls the AccessDecisionVoters which will
return "denied" to the AccessDecisionManager
* AccessDecisionManager with throw `AccessDeniedException` which gets handled by the `ExceptionTranslationFilter`
    * note that the ExceptionTranslationFilter sits in front of and wraps the FilterSecurityInterceptor. Its sole
    responsibility is to catch exceptions that might be thrown by FilterSecurityInterceptor
    * it is the bridge between Java exceptions and HTTP responses
* ExceptionTranslationFilter catches the exception and delegates to the `AccessDeniedHandler` which will return 
a HTTP response code of 403 (by default)

### Unauthenticated Flow
* in this example, an anonymous user is trying to access "/messages/inbox" which matches the request pattern
"/messages/**" and requires authentication and ROLE_USER
* the flow is the same as the access denied flow (above) (`AccessDeniedException` is still thrown)
* The ExceptionTranslationFilter will determine if this is really an AccessDeniedException or if it is
really an AuthenticatedException
* in our case, the user is unauthenticated so the ExceptionTranslationFilter calls the configured 
`AuthenticationEntryPoint`, we are using HTTP Basic so a 401 Response Code is returned along with a 
"WWW-Authenticate: Basic realm=spring" header.  This typically triggers a login page to display to the end-user 

### Exception Handling Recap
* when "Access Denied" for current Authentication, the `ExceptionTranslationFilter` delegates to the 
`AccessDeniedHandler`, which by default returns a 403 status
* when current Authentication is "Anonymous", the `ExceptionTranslationFilter` delegates to the 
`AuthenticationEntryPoint` to start the authentication process



## Videos
* good high level, architecture deep dive on [youtube](https://www.youtube.com/watch?v=8rnOsF3RVQc) from Spring 
Security's senior engineer

