---
layout: post
title: "Implementing OAuth2 in Spring: part 1"
author: "Zakaria"
comments: true
---

OAuth2 is a set specifications that provide means of securing access to Rest APIs mainly. The main purpose of OAuth is to allow performing authentication and authorization through the use of a token rather than having to provide credentials for each operation. As the focus of this post is the implementation, and in order not to reeinvent the wheel, it's possible to look at [OAuth RFC](https://tools.ietf.org/html/rfc6749) or [Wikipedia](https://en.wikipedia.org/wiki/OAuth) for more theoritical background. In this post, we will dive into OAuth2 implementation in Spring and how to use different grant types, but before it is worth providing a brief definition of some important concepts.

# Access Token and Refresh Token:

The access token is provided upon successful authentication along with the refresh token. The access token has a limited validity period (standard is 1 hour) after which the refresh token is neccessary to obtain a new access token and new refresh token. The referesh token usually expires upon use.

# Resource Server & Authorization Server:

OAuth introduces the concept of Authorization Server which is the entity who issues access and refresh tokens, and is consulted on each operation to see if the token is valid. The resource server is simply the actual Rest API that is accessed by different client applications (front end application, mobile, other backend services...). the Resource and the Authorization servers can be different entities or can be the same entity.

# Grants: 

The most used grants in OAuth are: client credentials, password, authorization code, and implicit. Each grant has a specific flow and use case, but since the focus of this post is not theoritical, we will focus rather on their implementation. More details about grants and their uses, can be found in the [OAuth RFC](https://tools.ietf.org/html/rfc6749#page-8). 

# Implementation:   

For the implementation, we are going to use Spring Boot to take advantage from its automatic configuration and bootstrapping capabilities, and focus more on our core topic.    

 - Resouce server:

We have a resource server with the following enpoints that we would like to secure:

{% highlight java  linenos %}
@RestController("/")
public class ResourceController {

    @GetMapping("/hello")
    public String hello(){
        return "hello";
    }

    @GetMapping("/foo")
    public String foo(){
        return "foo";
    }

    @GetMapping("/bar")
    public String bar(){
        return "bar";
    }

    @GetMapping("/test")
    public String test(){
        return "test";
    }
}
 {% endhighlight %}

To do so, we need to configure a [`ResourceServerConfigurerAdapter`](https://docs.spring.io/spring-security/oauth/apidocs/org/springframework/security/oauth2/config/annotation/web/configuration/ResourceServerConfigurerAdapter.html) bean annotated with [`@EnableResourceServer`](https://docs.spring.io/spring-security/oauth/apidocs/org/springframework/security/oauth2/config/annotation/web/configuration/EnableResourceServer.html):

{% highlight java  linenos %}
@Configuration
@EnableResourceServer
public class ResourceSecurityConfiguration extends ResourceServerConfigurerAdapter {

    @Override
    public void configure(ResourceServerSecurityConfigurer resources)
            throws Exception {
        resources.resourceId("resource");
    }

    @Override
    public void configure(HttpSecurity http) throws Exception {
        http
                .authorizeRequests().antMatchers("/foo", "/bar", "/hello", "/test").authenticated().
                and().csrf().disable();
    }

    @Bean
    public RemoteTokenServices LocalTokenService() {
        final RemoteTokenServices tokenService = new RemoteTokenServices();
        tokenService.setCheckTokenEndpointUrl("http://localhost:8081/oauth/check_token");
        tokenService.setClientId("my-client");
        tokenService.setClientSecret("mysecret");
        return tokenService;
    }
}
{% endhighlight %}

We have told spring to check authentication for our endpoints (it's possible to use `"/*"` or `.anyRequest()` to denote all of them). Additionally, we have configured a `RemoteTokenServices` bean to tell Spring to provide a token check endpoint (the Authorization server), and configured the client id and the secret. And our resource server is configured. Finally, we have set the resource id which serves as an identification in the authorization server if it is used by several resource servers ( which is quite common). 

 - Authorization server:

For implementing the authorization server, we are are going to use an in-memory client configuration. Spring Security offers also the possibility to store the oauth client config in a database which is more suited for production applications.  

{% highlight java  linenos %}
@Configuration
@EnableAuthorizationServer
public class AuthorizationSecurityConfig extends AuthorizationServerConfigurerAdapter {

    @Autowired
    private AuthenticationManager authenticationManager;

    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints)
            throws Exception {
        endpoints.authenticationManager(authenticationManager);
    }

    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        clients.inMemory().withClient("my-trusted-client")
                .authorizedGrantTypes("password",
                        "refresh_token", "implicit", "client_credentials", "authorization_code")
                .authorities("ROLE_CLIENT", "ROLE_TRUSTED_CLIENT")
                .scopes("read", "write", "trust")
                .accessTokenValiditySeconds(60)
                .redirectUris("http://localhost:8081/test.html")
                .resourceIds("resource")
                .secret("mysecret");
    }

    @Override
    public void configure(AuthorizationServerSecurityConfigurer oauthServer)
            throws Exception {
                 oauthServer
                .tokenKeyAccess("permitAll()")
                .checkTokenAccess("permitAll()");
    }
}
{% endhighlight %}

In addition to the client configuration in which we configured the client, the secret, the oauth scopes (more on that in next post), the authorities (roles associated with a token), the token validity, the resource id, we have configured the access to the check token endpoint which is provided by Spring Boot at `/oauth/check_token`, and the access to the token issuing endpoint which is also mapped automatically at: `/oauth/token`.


# OAuth in action:

We have configured the authorization server to run at port 8081 and the resource server to run at port 8989. For all the examples below `curl` is used, but the client can be any application. 

Let's try first to access one of the enpoints in the resource server: 

```
curl localhost:8989/foo
```

{% highlight java  linenos %}
{
    "error": "unauthorized",
    "error_description": "Full authentication is required to access this resource"
}
{% endhighlight %}

Let's obtain a token and try again. 

- Client credentials grant: 
```
curl -X POST --user my-trusted-client:mysecret localhost:8081/oauth/token -d 'grant_type=client_credentials&client_id=my-trusted-client' -H "Accept: application/json"
```

Response:

{% highlight js  linenos %}
{
  "access_token": "3670fea1-eab3-4981-b80a-e5c57203b20e",
  "token_type": "bearer",
  "expires_in": 51,
  "scope": "read write trust"
}
{% endhighlight %}

We can now use the token for accessing the protected endpoint: 

```
curl -v localhost:8989/foo -H "Authorization: Bearer 6bb86f18-e69e-4c2b-8fbf-85d7d5b800a4"

foo

```

Client credentials grant does not support a refresh token. 

- Password grant: 

The password grant is similar to client credentials in term of the flow for obtaining the token except that it uses actual user credentials. It also implies that a user needs to be configured for the application. The web security config is as follows:

  {% highlight java  linenos %}
@Configuration
@EnableWebSecurity
public class WebSecurity extends WebSecurityConfigurerAdapter {
	
	@Override
	protected void configure(AuthenticationManagerBuilder auth) throws Exception {
		auth.inMemoryAuthentication().withUser("gwidgets").password("gwidgets").authorities("CLIENT");
	}
	 
	@Override
	protected void configure(HttpSecurity http) throws Exception {
		 http.authorizeRequests().anyRequest().authenticated().and().formLogin().defaultSuccessUrl("/test.html").and().csrf().disable();
	}
}

 {% endhighlight %}

 And then we can use the user credentials to obtain a token, as below: 

```
curl -X POST --user my-trusted-client:mysecret localhost:8081/oauth/token -d 'grant_type=password&username=gwidgets&password=gwidgets' -H "Accept: application/json"
```

Response:

{% highlight js  linenos %}
{
  "access_token": "3670fea1-eab3-4981-b80a-e5c57203b20e",
  "token_type": "bearer",
  "expires_in": 51,
  "scope": "read write trust"
}
{% endhighlight %}

The password grant does not support a refresh token. 

- Implicit grant:

The implicit grant is mostly suited for front end routed applications. The implicit grant requires basic authentication and a HTTP session. To perform the implicit grant, we are going to add a simple http page to the authorization server (it could be on a different server): 

{% highlight xml  linenos %}

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>

<p>we are here</p>

</body>
</html>
 {% endhighlight %}

To perform the implicit grant, we need to navigate to the following address in a browser: http://localhost:8081/oauth/authorize?response_type=token&client_id=my-trusted-client&redirect-uri=http://localhost:8081/test.html

![Login redirect](https://s3-eu-west-1.amazonaws.com/gwidgets/login-spring.png)

After logging in, we get an OAuth approval page (provided by default by spring, but can be customized): 

![OAuth approval](https://s3-eu-west-1.amazonaws.com/gwidgets/oauth-approval.png)

After approving the scopes for the token, we are finally redirected to our page, in which we find the token in the hash of the url: 

![Implicit grant](https://s3-eu-west-1.amazonaws.com/gwidgets/implicit_grant.png)

- Authorization code grant:

For the authorization code grant, we need to first authorize in the same way as the implicit flow except that the `response_type` is now `code`. To do so, we need to navigate to:
http://localhost:8081/oauth/authorize?response_type=code&client_id=my-trusted-client&redirect-uri=http://localhost:8081/test.html

we then get redirected to login and after login, we get redirected to the OAuth scopes approval, as the implicit flow in the previous section. After that, we get redirected to the following address: http://localhost:8081/test.html?code=bD0mVb which is the welcome page of our app, but with a special query parameter: `code`. We are going to use curl to obtain the token for demonstration purposes, but it could be done from the page using JavaScript as well:

```
curl -X POST --user my-trusted-client:mysecret localhost:8081/oauth/token -d 'grant_type=authorization_code&code=bD0mVb&redirect_uri=http://localhost:8081/test.html' -H "Accept: application/json"
```

Response:

{% highlight js  linenos %}
{
    "access_token": "0abe701b-0f5a-4d25-81df-f2c4db2af555",
    "token_type": "bearer",
    "refresh_token": "cf6aa9db-3757-465e-af68-b7d59d1f0b77",
    "expires_in": 59,
    "scope": "trust read write"
}
 {% endhighlight %}

- Refreshing the token:

We have seen that the Authorization grant is the only grant that supports the refresh token. After using the access token for 60 seconds, it expires and we get the following response: 

curl -v localhost:8989/foo -H "Authorization: Bearer 0abe701b-0f5a-4d25-81df-f2c4db2af555"

{% highlight js  linenos %}
{
    "error": "invalid_token",
    "error_description": "0abe701b-0f5a-4d25-81df-f2c4db2af555"
}
 {% endhighlight %}

This means that the access token has expired. To obtain a new token, we need to use the refresh token :

```
curl -X POST --user my-trusted-client:mysecret localhost:8081/oauth/token -d 'client_id=my-trusted-client&grant_type=refresh_token&refresh_token=cf6aa9db-3757-465e-af68-b7d59d1f0b77' -H "Accept: application/json"
```

response: 

{% highlight js  linenos %}

{
    "access_token": "2f9a6609-fc64-4b1e-93a3-8232827da881",
    "token_type": "bearer",
    "refresh_token": "cf6aa9db-3757-465e-af68-b7d59d1f0b77",
    "expires_in": 59,
    "scope": "trust read write"
}
{% endhighlight %}

This process can be repeated every time the token expires. 

# Wrap up:

Spring OAuth provides OAuth endpoints and flows implemented out of the box, and can be a great solution to get OAuth set up with a minimum effort. However, it can be a little bit daunting for developers who are not familiar with Spring as lots of things are happening behind the curtains. Hopefully, this post can help in seeing the big picture. In the next posts, we will talk about the use OAuth scopes to secure endpoints.   

Full source code can be found here:  https://github.com/zak905/oauth2-example