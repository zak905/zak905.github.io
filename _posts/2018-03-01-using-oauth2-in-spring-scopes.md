---
layout: post
title: "Implementing OAuth2 in Spring: using scopes (part 2)"
author: "Zakaria"
comments: true
---

We have seen in the previous [post](http://www.zakariaamine.com/2018-01-27/using-oauth2-in-spring) basic OAuth2 concepts and how to implement and to perform different grants in Spring. In this post, we are going to go through another important concept of OAuth2: scopes. 

# OAuth scopes: 

Securing access to an application is usually carried out in two steps: authentication and authorization. To understand these two concept, suppose you work in a top secret governement building. Before starting, you were given a card that gives you access to building. The OAuth token can be seen as the card that allows you access. Once you are inside, you decide to go to the third floor to meet one of your colleagues, after trying to use your card to open the third floor's door, you get a beep telling you that you are not authorized. In OAuth, scopes are a way to define which resources can be accessed by the token and which cannot. Scopes allow access control, and can be seen as an equivalent to user roles in traditional authentication. 

# Implementation: 

To demonstrate scopes, we are going to use the example from [part 1](https://github.com/zak905/oauth2-example).  

In the [resource server's](https://github.com/zak905/oauth2-example/blob/master/resource-server/src/main/java/com/gwidgets/examples/resourceserver/ResourceController.java) controller, we have the following endpoints :


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

    @PostMapping("/bar")
    public String bar(){
        return "bar";
    }

    @DeleteMapping("/test")
    public String test(){
        return "test";
    }
}


 {% endhighlight %}


 the first step is to configure the [authorization server](https://github.com/zak905/oauth2-example/blob/master/authorization-server/src/main/java/com/gwidgets/examples/authorizationserver/AuthorizationSecurityConfig.java#L34) with the desired scopes: 

{% highlight java  linenos %}
 clients.inMemory().withClient("my-trusted-client")
                .authorizedGrantTypes("password",
                        "refresh_token", "implicit", "client_credentials", "authorization_code")
                .authorities("CLIENT")
                .scopes("read", "write", "trust")
                .accessTokenValiditySeconds(60)
                .redirectUris("http://localhost:8081/test.html")
                .resourceIds("resource")
                .secret("mysecret");
 {% endhighlight %}


To enable scopes checking in the resource server, we have two options: using the security configuration, or using method security. 

* using security configuration: 

{% highlight java  linenos %}
	@Override
	public void configure(HttpSecurity http) throws Exception {
		http
			.authorizeRequests()
			 .antMatchers(HttpMethod.GET,"/hello").access("#oauth2.hasScope('read')")
			 .antMatchers(HttpMethod.GET,"/foo").access("#oauth2.hasScope('read')")
			 .antMatchers(HttpMethod.POST,"/bar").access("#oauth2.hasScope('write')")
			 .antMatchers(HttpMethod.DELETE,"/test").access("#oauth2.hasScope('trust')")
			.anyRequest().authenticated().
			 and().csrf().disable();
	}
 {% endhighlight %}

* using method security: 

{% highlight java  linenos %}

    @PreAuthorize("#oauth2.hasScope('read')")
    @GetMapping("/hello")
    public String hello(){
        return "hello";
    }

    @PreAuthorize("#oauth2.hasScope('read')")
    @GetMapping("/foo")
    public String foo(){
        return "foo";
    }

    @PreAuthorize("#oauth2.hasScope('write')")
    @PostMapping("/bar")
    public String bar(){
        return "bar";
    }

    @PreAuthorize("#oauth2.hasScope('trust')")
    @DeleteMapping("/test")
    public String test(){
        return "test";
    }

{% endhighlight %}

additionally we need to add `@EnableGlobalMethodSecurity(prePostEnabled = true)` to any class that can be picked up by Spring (`@Configuration`, `@Service`, ...etc). In our example, we have added it to the [ResourceSecurityConfiguration](https://github.com/zak905/oauth2-example/blob/master/resource-server/src/main/java/com/gwidgets/examples/resourceserver/ResourceSecurityConfiguration.java#L18) class. The `prePostEnabled = true` tells Spring to enalble pre and post anotations like `@PreAuthorize`, `@PostFilter`, etc...


For those wondering about expressions like `#oauth2.hasScope('trust')`, they are built using the [Spring Expression Language(SpEL)](https://docs.spring.io/spring/docs/4.3.12.RELEASE/spring-framework-reference/html/expressions.html). 


# Scopes in action: 

By default, if the scopes are not present in the token request, Spring assumes that the token has all the configured scopes. Let's first request a token with `read` scope :

```
curl -X POST --user my-trusted-client:mysecret localhost:8081/oauth/token -d 'grant_type=client_credentials&client_id=my-trusted-client&scope=read' -H "Accept: application/json"
```
Response: 

{% highlight js  linenos %}
{
    "access_token": "acadbb31-f126-411d-ae5b-6a278cee2ed6",
    "token_type": "bearer",
    "expires_in": 60,
    "scope": "read"
}
 {% endhighlight %}

 Now, we can use the token to access the endpoints with `read` scope access: 

```
 curl -XGET localhost:8989/hello -H "Authorization: Bearer acadbb31-f126-411d-ae5b-6a278cee2ed6"

 hello

 ```

```
  curl -XGET localhost:8989/foo -H "Authorization: Bearer acadbb31-f126-411d-ae5b-6a278cee2ed6"

 foo

```

Now, let's try to use this token on an endpoint that accepts `write` scope only: 

```
curl -XPOST localhost:8989/bar -H "Authorization: Bearer acadbb31-f126-411d-ae5b-6a278cee2ed6"
```

Response:

{% highlight js linenos %}
{
    "error": "access_denied",
    "error_description": "Access is denied"
}
{% endhighlight %}

The access is rejected because the token does not have the required scope. Let's try to obtain a new token with `write` scope and try again: 

```
curl -X POST --user my-trusted-client:mysecret localhost:8081/oauth/token -d 'grant_type=client_credentials&client_id=my-trusted-client&scope=write' -H "Accept: application/json"
```
Response:

{% highlight js  linenos %}
{
    "access_token": "bf0fa83a-23bd-4633-ac6c-a06f40d53e5f",
    "token_type": "bearer",
    "expires_in": 3599,
    "scope": "write"
}
{% endhighlight %}



```
curl -XPOST localhost:8989/bar -H "Authorization: Bearer bf0fa83a-23bd-4633-ac6c-a06f40d53e5f"

bar
```

# Wrap up :

Scopes are an important aspect of OAuth as the tokens does not carry informations about their user or requester. Scopes allow to limit access to resources for better access control and security. In the next post we will see how to integrate external OAuth providers like Google, and Facebook into the flow.   