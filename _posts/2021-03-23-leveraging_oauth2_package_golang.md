---
layout: post
title: "Leveraging the golang.org/x/oauth2 package"
author: "Zakaria"
comments: true
description: golang.org/x/oauth2 package
---

It is quiet common for applications to integrate with third party services and APIs through OAuth. Since the OAuth protocol has well defined [standards](https://tools.ietf.org/html/rfc6749), many libraries and tools has arised for each programming language to help walk though the authrorization process smoothly, Golang is no exception. Golang comes with the  `golang.org/x/oauth2` package which is not part of the standard library, but it is built and maintained by the Golang team themselves. It is kind of an official "add-on" or extension to the standard library. I noticed recently in many code bases that this package is overlooked, with many developers doing the flow manually using http requests either because they have never heard of the package or because they are not too convinced (maybe the `/x` plays a role). I would like through this post to give this package what it deserves because it can make the OAuth integration (from a client perpective) hell lot simpler, while keeping the code base neat and clean, especially if you have more than one provider to integrate with.

## structs and utils to obtain and manipulate the tokens

`golang.org/x/oauth2` comes with a handefull of utils and structs to obtain and hold the token. The most important structs in the package is `oauth2.Config` and `oauth2.Token`. `oauth2.Config` allows the declaration of an OAuth provider and has receiver methods to obtain the token using authorization code or the password grant (the client id/secret is also supported but in the `golang.org/x/oauth2/clientcredentials` subpackage). For example, The `AuthCodeURL` method generates the authorize url that can be used from the front end to intiate an authorization code first step. After obtention of the code in the front end, the `Exchange` method can be called with the code as parameter to obtain the token which is automatically encoded as an `oauth2.Token`, so the steps to do the authorization code grant are:

{% highlight go  %}

//create a config or use an existing one
//let's leave this empty for now
config := oauth2.Config{}

//get the url and send it back to the front end 
//or do redirect if you are doing things server side
url := config.AuthCodeURL("some state")

//once you obtain the code 
//you can get the token using Exchange
token, err := config.Exchange(context.Background(), code)

//check error...etc

// same thing goes for the password grant 
token, err := h.oauthConfig.PasswordCredentialsToken(context.Background(), "username", "password")

//check error...etc
{% endhighlight %}

As you can see, using a three liner we were able to perform the authorization code grant and the password. This is much cleaner and expressive than using http requests directly. Same thing goes for the client id grant except that we need to use the config struct from `golang.org/x/oauth2/clientcredentials` and then use the `Token` method. 

The returned `Token` struct has its expiry field already converted to `time`, so it is ideal if you want to compare against the actual time for testing if the token is has expired. As a matter of fact, `Valid` method serves the purpose already `myToken.Valid()`. If you ever fall into a case where you need to encode JSON as a `Token` manually, you should be aware that, according to OAuth [specification](https://tools.ietf.org/html/rfc6749#section-4.4.3), the token `expires_in` field contains the duration after which the token will expire and not a date, and therefore a transformation is necessary (adding the `expires_in` so the current time). Here is how the package does things internally:  

{% highlight go  %}

		e := vals.Get("expires_in")
		expires, _ := strconv.Atoi(e)
		if expires != 0 {
			token.Expiry = time.Now().Add(time.Duration(expires) * time.Second)
		}

{% endhighlight %}

source: https://github.com/golang/oauth2/blob/master/internal/token.go#L262

## Bring your own OAuth provider or use one of providers already configured

`golang.org/x/oauth2` has a long list of subpackages that provides endpoints configuration for known oauth providers like Google, Github, Stackoverflow...etc It's also quiet straightforward to create an `oauth2.Endpoint` struct that has a token and an authorization endpoints fields. 

## golang.org/x/oauth2 in action

The below example shows the usage of the package to perform the client credential grant on a locally running instance of [keycloak](https://www.keycloak.org/) on the `4577` port. The `config.Client` method was called to obtain a `http.Client` with the oauth token already set. You can however use the `token.AccessToken` field directly, if you prefer using a different http client. 

{% highlight go  %}

package main

import (
	"context"
	"flag"
	"fmt"
	"io/ioutil"
	"log"

	"golang.org/x/oauth2/clientcredentials"
)

func main() {

	clientID := flag.String("client-id", "test", "client id from your keycloak client")
	clientSecret := flag.String("client-secret", "b5d6a35c-8ce5-11eb-9903-1758b78df60f", "client secret from your keycloak client")
	flag.Parse()

	config := clientcredentials.Config{
		ClientID:     *clientID,
		ClientSecret: *clientSecret,
		TokenURL:     "http://localhost:4577/auth/realms/myrealm/protocol/openid-connect/token",
		Scopes:       []string{"openid", "roles", "offline_access"},
	}

	token, err := config.Token(context.Background())

	if err != nil {
		log.Fatal(err.Error())
	}

	fmt.Println("oauth successfull here is the access token ", token.AccessToken)

	client := config.Client(context.Background())

	res, err := client.Get("http://localhost:4577/auth/admin/realms/myrealm/users")

	if err != nil {
		log.Fatal(err.Error())
	}

	body, err := ioutil.ReadAll(res.Body)

	if err != nil {
		log.Fatal(err.Error())
	}

	fmt.Println("Yay! Got response: ", (string)(body))

}
{% endhighlight %}


We have seen how the `golang.org/x/oauth2` makes coding the OAuth authorization process smooth and clean. Don't forget that the package is not part of the golang standard library, so `go get golang.org/x/oauth2` is necessary. 
