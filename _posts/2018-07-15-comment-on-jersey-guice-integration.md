---
layout: post
title: "Using Guice with Jersey 2 (Part 2): guice-bridge library"
author: "Zakaria"
comments: true
---

# Introduction

The previous [post](http://www.zakariaamine.com/2017-12-11/how-to-use-guice-jersey-2) "Using Guice with Jersey 2, without external librairies" proposed a trick to integrate Guice into Jersey2 dependency injection system (HK2) without helpers or libraries. In that post, I have also mentioned that none of the metionned external libraries work well for this integration, which lead to the necessity of that hack. At that point, I was not aware of HK2 [guice-bridge](https://github.com/javaee/hk2/tree/master/guice-bridge), which is the solution proposed by [HK2](https://javaee.github.io/hk2/) to tackle the issues related to HK2 and guice integration. This post shows an example of guice-bridge usage.  

# Using HK2 guice-bridge:

Our [Config](https://github.com/zak905/jersey2-guice-example/blob/without_library/src/main/java/com/gwidgets/Config.java) class from the previous post becomes:

{% highlight java %}
@ApplicationPath("/")
public class Config extends ResourceConfig {
	@Inject
	public Config(ServiceLocator serviceLocator) {
		packages("com.gwidgets.resource");
		Injector injector = Guice.createInjector(new GuiceModule());
		initGuiceIntoHK2Bridge(serviceLocator, injector);
	}

	private void initGuiceIntoHK2Bridge(ServiceLocator serviceLocator, Injector injector) {
		GuiceBridge.getGuiceBridge().initializeGuiceBridge(serviceLocator);
		GuiceIntoHK2Bridge guiceBridge = serviceLocator.getService(GuiceIntoHK2Bridge.class);
		guiceBridge.bridgeGuiceInjector(injector);
	}
}
{% endhighlight %}

for readability purposes, I have encapsulated the logic for the bridge set up into the `initGuiceIntoHK2Bridge` method. We can now get rid of the [HK2toGuiceModule](https://github.com/zak905/jersey2-guice-example/blob/without_library/src/main/java/com/gwidgets/HK2toGuiceModule.java) class.   

# Wrap up: 

As of the time this post is written, the guice-bridge seems to be the cleanest and simplest way of integrating guice into HK2 (and thus Jersey 2), and therefore it is the recommended method to use. 

Source code: [https://github.com/zak905/jersey2-guice-example](https://github.com/zak905/jersey2-guice-example)