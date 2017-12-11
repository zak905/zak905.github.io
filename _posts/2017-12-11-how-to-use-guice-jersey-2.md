---
layout: post
title: "Using Guice with Jersey 2, without external librairies"
author: "Zakaria"
comments: true
---

If you have a legacy application or you are developing a new application that uses Guice for dependency injection, and there  is Jersey 2 involved, then you may face several issues due the new dependency injection mechanism introduced in Jersey 2 called Hk2. HK2 is an implementation of ([JSR 330](https://jcp.org/en/jsr/detail?id=330)) (`javax.inject`), and is an integral part of Jersey 2, which means that you cannot leave it out. Guice also implements JSR 330 as well which is a source of conflict because Jersey will not be aware of what Guice injects and Guice can not do the job of injecting Jersey's resources (filters, classes annotated with `@Path` or `@Provider`,...etc). 

There are several options available out there that integrates between the two like: [https://github.com/AetherWorks/jersey2-example](https://github.com/AetherWorks/jersey2-example), [https://github.com/javaee/hk2/tree/master/guice-bridge](https://github.com/javaee/hk2/tree/master/guice-bridge), [https://github.com/Squarespace/jersey2-guice](https://github.com/Squarespace/jersey2-guice), but none of them seem to work in 100% of the cases, at least from my experience. At this point, you can either ditch Guice and convert all your DI logic to HK2, or ditch Jersey 2 and look for another framework compatible with Guice (many recommend [RESTEasy](http://resteasy.jboss.org/)). In either case, there is a tradeoff: Guice is seen as more advanced and mature than HK2, Jersey is the reference JAX-RS implementation. After a number of experiments, I could a find a trick that allows working with both, which I am going to explain in this post. 

Suppose we have the following services and their respective implementation configured by a Guice module :

{% highlight java  linenos %}
public interface SimpleService {
	
	String getMessage();

}
{% endhighlight %}

{% highlight java  linenos %}
public interface AnotherService {
	
	String provideName();

}
{% endhighlight %}

{% highlight java  linenos %}
@Singleton
public class SimpleServiceImpl implements SimpleService {	
	@Inject
	AnotherService anotherService;

	@Override
	public String getMessage() {
		return "Howdy" + " " + anotherService.provideName();
	}
}
{% endhighlight %}

{% highlight java  linenos %}
public class AnotherServiceImpl implements AnotherService {
	@Override
	public String provideName() {
		return "foo";
	}
}
{% endhighlight %}

{% highlight java  linenos %}
public class GuiceModule extends AbstractModule {
	@Override
	protected void configure() {
		bind(SimpleService.class).to(SimpleServiceImpl.class);
		bind(AnotherService.class).to(AnotherServiceImpl.class);
	}
}
{% endhighlight %}


We want to return `SimpleService`'s `getMessage()` method from a Rest enpoint in Jersey 2, something like: 

{% highlight java  linenos %}
@Path("/resource")
public class MyResource {
	
	@Inject
	SimpleService simpleService;

    @GET
    @Produces(MediaType.TEXT_PLAIN)
    public String getIt() {
        return simpleService.getMessage();
    }
}
{% endhighlight %}

# javax.inject.* or com.google.inject.*:

After experimenting with Guice and Jersey, I realized that HK2 does not scan `com.google.inject.*` annotations, but only `javax.inject.*`, so it is possible to take advantage of this to workout a solution. All classes used by Guice can use `com.google.inject.*` while the ones used by HK2 (classes annotated with `@Path`, `@Provider`,...etc), can use `javax.inject.*`. Finally, to glue between Guice and HK2, we can create a HK2 module that pulls its bindings class from Guice. 
                         

# HK2 Guice bridge module :

We are injecting `SimpleService` in `MyResource`, so we need to tell HK2 how to bind it: 


{% highlight java  linenos %}
public class HK2toGuiceModule extends AbstractBinder {
	private Injector guiceInjector;

	public HK2toGuiceModule(Injector guiceInjector) {
		this.guiceInjector = guiceInjector;
	}

	@Override
	protected void configure() {
		bindFactory(new ServiceFactory<SimpleService>(guiceInjector, SimpleService.class)).to(SimpleService.class);
	}	
	
	private static class ServiceFactory<T> implements Factory<T> {

	    private final Injector guiceInjector;

	    private final Class<T> serviceClass;

	    public ServiceFactory(Injector guiceInjector, Class<T> serviceClass) {

	      this.guiceInjector = guiceInjector;
	      this.serviceClass = serviceClass;
	    }

	    @Override
	    public T provide() {
	      return guiceInjector.getInstance(serviceClass);
	    }

	    @Override
	    public void dispose(T versionResource) {
	    }
	  }
{% endhighlight %}


it is worth mentioning that ServiceFactory is equivalent to [`@Provides`](http://google.github.io/guice/api-docs/latest/javadoc/index.html?com/google/inject/Provides.html) in Guice. It is used when the initialization of a binding is not simply done by invoking the constuctor, but requires custom code. In our case , we have told HK2 to pull `SimpleService` from `guiceInjector` : 

{% highlight java  %}
           @Override
	    public T provide() {
	      return guiceInjector.getInstance(serviceClass);
	    }
{% endhighlight %}

We have not added `AnotherService` simply because this is not injected in a Jersey resource, so the dependency between `AnotherService` and `SimpleService` will be handled by Guice. 

# Final touch :

Finally, we need to configure both our Guice and Jersey modules on application start up, so we have to add a configuration class that wires up everything: 

{% highlight java  linenos %}
public class Config extends ResourceConfig {

	public Config() {
		packages("com.gwidgets");
		Injector injector = Guice.createInjector(new GuiceModule());
		HK2toGuiceModule hk2Module = new HK2toGuiceModule(injector);
		register(hk2Module);
	}
}
{% endhighlight %}

We have passed our Guice `injector` to the  `HK2toGuiceModule` which we registred as the source of bindings for Jersey.  

# Wrap up: 

Until Jersey folks come up with a viable solution to Guice/Jersey 2 integration, this trick, as hacky as it seems, may allow working with both on the same application without headaches. Remember `javax.inject.Inject` is not `com.google.inject.Inject`!

Full source code: [https://github.com/zak905/jersey2-guice-example](https://github.com/zak905/jersey2-guice-example) 