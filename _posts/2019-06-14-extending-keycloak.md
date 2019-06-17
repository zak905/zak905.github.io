---
layout: post
title: "Extending Keycloak: adding API key authentication"
author: "Zakaria"
comments: true
---

# Background

API key authentication is one of the simplest ways for securing access to resources and APIs. The principle is simple: you are provided with a static key that you should keep safe and use to access the protected APIs (usually sent as a special header, or using `Authentication` header). If you are using Keycloak, and would like to add an API key authentication feature, read along. In this post, I would like to demonstrate how to extend Keycloak by adding a simple API key authentication mechansim. This can be useful if you work in a microservices architecture and have different services that authenticates differently.  

# Design

Let's suppose our system is composed of two services: a Spring boot app that serves our app dashboard pages, and a node js stateless Rest API that provides some important weather forecast data. The access to both services can be done separately and through different URIs, but the sign up through the dashboard is required. The end user would sign up to obtain the API key which can be used anytime to access the weather Rest API. This is a common scenario for APIs as-a- service applications. Additionally, our system has a Keycloak auth server that the services turn to for authentication and authorization. To secure our dashboard service, we can use the sso mechanism provided by Keycloak. To secure the Rest API service, we can introduce API key authentication: a random key that can be generated and stored with user data at registration time. We need also an endpoint to check if the API key exists.

Accordingly, we need to extend Keycloak with a module that adds the following features:
  - generating a random key string and storing it with user attributes at registration time
  - endpoint that checks whether the key is valid or no. 

# You name it, I code it

One of the most attractive Keycloak features is its extensibility. Keycloak can be extended quite easily by implementing its [SPI](https://github.com/keycloak/keycloak/blob/master/server-spi/src/main/java/org/keycloak/provider/Spi.java) interfaces or overriding its [providers](https://github.com/keycloak/keycloak/blob/master/server-spi-private/src/main/java/org/keycloak/events/EventListenerProvider.java). In this section we are going to dive directly into the module implementation, more details on how to extend Keycloak can be found in the [docs](https://www.keycloak.org/docs/latest/server_development/index.html#_extensions). 

We can start by implementing the API key generation. For this purpose, we need to capture the registration event and perform our key generation. To capture different Keycloak internal events and take action, `EventListenerProvider` needs to be implemented: 

{% highlight java  %}

public class RegisterEventListenerProvider implements EventListenerProvider  {

    private KeycloakSession session;
    private RealmProvider model;
    //keycloak utility to generate random strings, anything can be used e.g UUID,..
    private RandomString randomString;
    private EntityManager entityManager;

    public RegisterEventListenerProvider(KeycloakSession session) {
        this.session = session;
        this.model = session.realms();
        this.entityManager = session.getProvider(JpaConnectionProvider.class).getEntityManager();
        this.randomString = new RandomString(50);
    }

    public void onEvent(Event event) {
        //we are only interested in the register event
        if (event.getType().equals(EventType.REGISTER)) {
            RealmModel realm = model.getRealm(event.getRealmId());
            String userId = event.getUserId();
            addApiKeyAttribute(userId);
        }
    }

    public void onEvent(AdminEvent adminEvent, boolean includeRepresentation) {
        // in case the user is created from admin or rest api
        if (Objects.equals(adminEvent.getResourceType(), ResourceType.USER) && Objects.equals(adminEvent.getOperationType(), OperationType.CREATE)) {
            String userId = adminEvent.getResourcePath().split("/")[1];
            if (Objects.nonNull(userId)) {
                addApiKeyAttribute(userId);
            }
        }
    }

  public void addApiKeyAttribute(String userId) {
        String apiKey = randomString.nextString();
        UserEntity userEntity = entityManager.find(UserEntity.class, userId);
        UserAttributeEntity attributeEntity = new UserAttributeEntity();
        attributeEntity.setName("api-key");
        attributeEntity.setValue(apiKey);
        attributeEntity.setUser(userEntity);
        attributeEntity.setId(UUID.randomUUID().toString());
        entityManager.persist(attributeEntity);
   }

        public void close() {
           //belongs to the interface, used in case there is some clean up to do, before destroying instances.  
        }
  }
{% endhighlight %}


In Keycloak, every provider is associated with a factory which is responsible for creating instances, so we are going to need to implement the `EventListenerProviderFactory` : 

{% highlight java %}

public class RegisterEventListenerProviderFactory implements EventListenerProviderFactory {

    public EventListenerProvider create(KeycloakSession keycloakSession) {
        return new RegisterEventListenerProvider(keycloakSession);
    }

    public void init(Config.Scope scope) {
    }

    public void postInit(KeycloakSessionFactory keycloakSessionFactory) {

    }

    public void close() {
    }

    public String getId() {
        //unique id for the provider
        return "api-key-registration-generation";
    }
}
{% endhighlight %}

Next we need to create an endpoint that can be used to check if the API key is valid:

{% highlight java  %}

public class ApiKeyResource {

    private KeycloakSession session;

    public ApiKeyResource(KeycloakSession session) {
        this.session = session;
    }

    @GET
    @Produces("application/json")
    public Response checkApiKey(@QueryParam("apiKey") String apiKey) {
        //if key exists return 200 otherwise 401
        List<UserModel> result = session.userStorageManager().searchForUserByUserAttribute("api-key", apiKey, session.realms().getRealm("example"));
        return result.isEmpty() ? Response.status(401).build(): Response.ok().build();
    }
}
{% endhighlight %}

Keycloak is, for most of its parts, implemented using Java (Jakarta) EE, so `JAX-RS` annotations shoud be used to create endpoints.

To make Keycloak recognize our endpoint, we need to implement `RealmResourceProvider` and `RealmResourceProviderFactory`. 

{% highlight java  %}

public class ApiKeyResourceProvider implements RealmResourceProvider {

    private KeycloakSession session;

    public ApiKeyResourceProvider(KeycloakSession session) {
        this.session = session;
    }

    public Object getResource() {
        return new ApiKeyResource(session);
    }

    public void close() {}
}

{% endhighlight %}

{% highlight java  %}
public class ApiKeyResourceProviderFactory implements RealmResourceProviderFactory {

    public RealmResourceProvider create(KeycloakSession session) {
        return new ApiKeyResourceProvider(session);
    }

    public void init(Config.Scope config) {}

    public void postInit(KeycloakSessionFactory factory) {}

    public void close() {}

    public String getId() {
        return "check";
    }
}
{% endhighlight %}

Finally, we need to tell Keycloak that we are overriding its providers by creating mappings under `META-INF/services` :

Filename: org.keycloak.events.EventListenerProviderFactory

```
com.gwidgets.providers.RegisterEventListenerProviderFactory
```

Filename: org.keycloak.services.resource.RealmResourceProviderFactory

```
com.gwidgets.providers.ApiKeyResourceProviderFactory
```

# Module packaging:

Keycloak is provided as a standalone web app running on Wildfly, so modules can be installed as `.ear` or `.jar` under `standalone/deployments`. More on that can be found in the official [documentation](https://www.keycloak.org/docs/latest/server_development/index.html#register-a-provider-using-modules). For the sake of simplicity, we will follow the structure in this demo repository: [https://github.com/dteleguin/beercloak](https://github.com/dteleguin/beercloak)

So our project has the following structure: 

  | api-key-ear
  | api-key-module
  | pom.xml

Full source code can be found at: [https://github.com/zak905/keycloak-api-key-demo](https://github.com/zak905/keycloak-api-key-demo)

Finally, we are going to create our docker image to be able to test our Keycloak module with other services: 

```
FROM java:8-jre-alpine

ENV KEYCLOAK_VERSION 6.0.1

#ca-certificates and openssl are required to download from https
RUN apk add --no-cache ca-certificates openssl && wget https://downloads.jboss.org/keycloak/${KEYCLOAK_VERSION}/keycloak-${KEYCLOAK_VERSION}.tar.gz

RUN tar xvf keycloak-${KEYCLOAK_VERSION}.tar.gz && rm keycloak-${KEYCLOAK_VERSION}.tar.gz

WORKDIR keycloak-${KEYCLOAK_VERSION}

#add admin user
RUN ./bin/add-user-keycloak.sh -u admin -p admin --realm master

COPY target/api-key-ear-0.1.ear standalone/deployments

EXPOSE 8080

ENTRYPOINT ["./bin/standalone.sh", "-b", "0.0.0.0"]
```

# Further configuration:

For this example, we will create a realm named `example`, and a client for our dashboard app named `dashboard-client`, I will not get into the details of realm and client configuration. More on that can be found in Keycloak [docs](https://www.keycloak.org/docs/latest/getting_started/index.html#creating-a-realm-and-user) 

Once Keycloak starts, the event listener needs to be manually added from the admin menu, under the master realm, by going to events -> config tab, and adding the id of the  `RegisterEventListenerProviderFactory` under Event Listeners field. 

![event listener registration](https://gwidgets.s3-eu-west-1.amazonaws.com/event_id_registration.jpg)

Also, we have to configure Keycloak to return the `api-key` attribute with the authentication token response (JWT token). For this purpose, we need to create a user attribute `Mapper` by going to Clients -> dashboard-client -> Mappers tab and then clicking `create` 

![keycloak mapper creation](https://gwidgets.s3-eu-west-1.amazonaws.com/mapper-keycloak.jpg)

The mapper needs to be created with following settings:

- Mapper Type: User Attribute
- User Attribute: api-key
- Token Claim Name: api-key
- Claim JSON Type: String

This will tell Keycloak to return the api-key created for the user with the authentication response. 

# Deploying and testing our services:

To test our services, we are going to create a `docker-compose.yaml` with our services:

```
version: '3.3'
services:
  auth-server:
    build: api-key-ear
    environment:
      REALM_NAME: example
    command: ["-Dkeycloak.migration.action=import", "-Dkeycloak.migration.provider=dir", "-Dkeycloak.migration.dir=/import", "-Dkeycloak.migration.strategy=OVERWRITE_EXISTING"]
    volumes:
      - ./api-key-ear/import:/import
    ports:
    - "8080:8080"
  dashboard-service:
    build: dashboard-service
    environment:
      REALM_NAME: example
    ports:
    - "8180:8180"
  rest-api-service:
    build: rest-api-service
    environment:
      REALM_NAME: example
      AUTH_SERVER_URL: auth-server:8080
    ports:
    - "8280:8280"
```

If we try to access our dashboard at `localhost:8180`, we will be redirected to the auth-server login page (please note that since we are in localhost settings, the `/etc/hosts` need to be edited to make `auth-server` point to localhost):

![login](https://gwidgets.s3-eu-west-1.amazonaws.com/login.jpg)

Since this is our first login, we need to register 

![register](https://gwidgets.s3-eu-west-1.amazonaws.com/register.jpg)

Once registred, we are redirected to our dashboard page that displays our generated API key.

![dashboard](https://gwidgets.s3-eu-west-1.amazonaws.com/dashboard2.jpg)

We can now call our `rest-api-service` using the API key

```
curl -H "X-API-KEY: YPqIeqhbxUcOgDd6ld2jl9txfDrHxAPme89WLMuC8e0oaYXeA7" localhost:8280

{"forecast" : "weather is cool today"}
```

If we try to access with a wrong key, we get a 401 response.

```
curl -v -H "X-API-KEY: wrongkey" localhost:8280

< HTTP/1.1 401 Unauthorized
< X-Powered-By: Express
< Date: Sun, 16 Jun 2019 18:41:34 GMT
< Connection: keep-alive
< Content-Length: 0

```

Source code: [https://github.com/zak905/keycloak-api-key-demo](https://github.com/zak905/keycloak-api-key-demo)