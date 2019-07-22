---
layout: post
title: "Extending Keycloak (continued): using a custom email sender"
author: "Zakaria"
comments: true
description: example of using SES as keycloak's email sender
---

# Introduction

In the last post [Extending Keycloak: adding API key authentication]({{ '/2019-06-14/extending-keycloak/' | relative_url}}), I took a glimpse at  how to extend Keycloak with a custom authentication method. I would like to continue in this post series about Keycloak with another useful customization: a custom email sender.

As mentioned before, Keycloak offers tons of customization capabilities. Whether it relates to how the data is stored, or how your login page will look like, Keycloak can be extended from the bottom up. Keycloak notifies users by e-mail about different events that can happen while handling their account (password reset, email confimation,..etc), and uses by default `javax.mail` for that which may not be ideal for some. Luckily, it is possible to plug a custom email sender. We just need to override the right provider, in this case [EmailSenderProvider](https://github.com/keycloak/keycloak/blob/master/server-spi-private/src/main/java/org/keycloak/email/EmailSenderProvider.java), and implement its corresponding factory, [EmailSenderProviderFactory](https://github.com/keycloak/keycloak/blob/master/server-spi-private/src/main/java/org/keycloak/email/EmailTemplateProviderFactory.java). For the demonstration purpose, we are going to use [Amazon SES](https://aws.amazon.com/ses/) as our email client. 

As a start, we are going to need the dependency for the java aws ses sdk:

{% highlight xml %}
        <dependency>
            <groupId>com.amazonaws</groupId>
            <artifactId>aws-java-sdk-ses</artifactId>
            <version>1.11.538</version>
        </dependency>
{% endhighlight %}


# Implementation

We can use the previous post's [project](https://github.com/zak905/keycloak-api-key-demo) as a starting point. Let's first implement our `SESEmailSenderProvider.java` :


{% highlight java %}

public class SESEmailSenderProvider implements EmailSenderProvider {

  private static final Logger log = Logger.getLogger("org.keycloak.events");

  private final AmazonSimpleEmailService sesClient;

  public SESEmailSenderProvider(
      AmazonSimpleEmailService sesClient) {
    this.sesClient = sesClient;
  }

  @Override
  public void send(Map<String, String> config, UserModel user, String subject, String textBody,
      String htmlBody) {

      log.info("attempting to send email using aws ses for " + user.getEmail());

      Message message = new Message().withSubject(new Content().withData(subject))
          .withBody(new Body().withHtml(new Content().withData(htmlBody))
              .withText(new Content().withData(textBody).withCharset("UTF-8")));

     //config.get("from") is set from the ui to avoid hardcoding it inside the module

      SendEmailRequest sendEmailRequest = new SendEmailRequest()
          .withSource("example<" + config.get("from") + ">")
          .withMessage(message).withDestination(new Destination().withToAddresses(user.getEmail()));

      sesClient.sendEmail(sendEmailRequest);
     log.info("email sent to " + user.getEmail() + " successfully");
  }

  @Override
  public void close() {

  }
}



{% endhighlight %}

Then we have to implement the provider factory which we will name `SESEmailSenderProviderFactory.java` 


{% highlight java  %}
public class SESEmailSenderProviderFactory implements EmailSenderProviderFactory {

  private static AmazonSimpleEmailService sesClientInstance;

  @Override
  public EmailSenderProvider create(KeycloakSession session) {

    //using the singleton pattern to avoid creating the client each time create is called
    if (sesClientInstance == null) {

      String awsRegion = Objects.requireNonNull(System.getenv("AWS_REGION"));

      sesClientInstance =
          AmazonSimpleEmailServiceClientBuilder
              .standard().withCredentials(new EnvironmentVariableCredentialsProvider())
              .withRegion(awsRegion)
              .build();
    }

    return new SESEmailSenderProvider(sesClientInstance);
  }

  @Override
  public void init(Scope config) {

  }

  @Override
  public void postInit(KeycloakSessionFactory factory) { }

  @Override
  public void close() {
  }

  @Override
  public String getId() {
    return "default";
  }
}

{% endhighlight %}

Finally, we have to create a new file named `org.keycloak.email.EmailSenderProviderFactory` in `META-INF/services/` with the full qualified name of our factory class:

```
com.gwidgets.providers.SESEmailSenderProviderFactory
```

In this way, Keycloak will be aware that we are overriding one of its provider factories. 

Easy peasy. Keycloak will now send e-mails using AWS SES. 

The added source code can be found in this git commit: [https://github.com/zak905/keycloak-api-key-demo/commit/3e686cfec7d3229cf3fd7ac8d26900a866c6e64f](https://github.com/zak905/keycloak-api-key-demo/commit/3e686cfec7d3229cf3fd7ac8d26900a866c6e64f)
