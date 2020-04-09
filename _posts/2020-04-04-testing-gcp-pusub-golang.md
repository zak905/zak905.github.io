---
layout: post
title: "Testing GCP Pub/Sub applications using the emulator (golang example)"
author: "Zakaria"
comments: true
description: testing pubsub using the emulator
---

Pub/Sub is a messaging/queuing service provided by Google Cloud platform. It seems like a rather simplistic alternative to the many well established messaging systems like Kafka or RabbitMQ. In this post, I would like to share few tips for testing applications that uses Pub/Sub using the emulator tool provided by GCP. We will use `golang` language as an example. 

# Setting up the Pub/Sub emulator

First things first, the first step is to run the [emualtor](https://cloud.google.com/sdk/gcloud/reference/beta/emulators/pubsub) using the [gcloud cli tool](https://cloud.google.com/sdk/gcloud): `gcloud beta emulators pubsub start --project=test` which will run the emulator exposed in the default port 8085. Ideally, the command could also be run using a docker container with `gcloud` installed. In this case, it is necessary to add the `--host` option to bind the emulator host to the docker container interface: `--host-port=0.0.0.0:8085`. In this example, we will use the following docker-compose:

```
version: "3.3"
services:
  pub-sub-emulator:
    image: google/cloud-sdk:latest
    command: ["gcloud", "beta", "emulators", "pubsub", "start", "--host-port=0.0.0.0:8085", "--project=test"]
    ports:
      - "8085:8085"
```

the `google/cloud-sdk` was chosen because it's the official image, but it's also possible to build a custom image with less layers and lighter size. 

# The app to test

Let's suppose we want to test a simple application that reads a json document from a Pub/Sub subscription, adds a "processed_time" field that contains the current timestamp, and sends the result to another Pub/Sub topic. The application expects that the topics and subscriptions are created in advance. This can be done programatically, from the CLI tool or the GCP UI.    

{% highlight go  %}
package main

import (
	"context"
	"encoding/json"
	"log"
	"time"

	"cloud.google.com/go/pubsub"
	"google.golang.org/api/option"
)

type App struct {
	context context.Context
	client  *pubsub.Client
	config  Config
}

type Config struct {
	context          context.Context
	gcpProjectName   string
	subscriptionName string
	topicName        string
	options          []option.ClientOption
}

func newApp(config Config) (*App, error) {
	client, err := pubsub.NewClient(config.context, config.gcpProjectName, config.options...)
	if err != nil {
		return nil, err
	}

	return &App{context: config.context, client: client, config: config}, nil
}

func (app *App) run() {

	log.Println("waiting for messages")

	app.client.Subscription(app.config.subscriptionName).Receive(app.config.context, func(ctx context.Context, message *pubsub.Message) {

		var messageJson map[string]interface{}

		json.Unmarshal(message.Data, &messageJson)

		log.Printf("received message with id: %s and content %v", message.ID, messageJson)

		messageJson["processed_time"] = time.Now()

		result, _ := json.Marshal(messageJson)

		app.client.Topic(app.config.topicName).Publish(ctx, &pubsub.Message{Data: result})

		message.Ack()
	})

	log.Println("stopped waiting for messages")
}
{% endhighlight %}

# Let's test

The tricky part about Pub/Sub is that any subscription created after the reception of a message will not be able to catch this message (sent prior to the subscription time). Therefore it's necessary to create a subscription before all the messages arrive, ideally after the topic creation. The first step before testing is to set the `PUBSUB_EMULATOR_HOST` env variable to the host where the emulator host runs, in our case `localhost:8085`. This is automatically picked up by the sdk as shown in the source code [here](https://github.com/googleapis/google-cloud-go/blob/master/pubsub/pubsub.go#L58).

{% highlight go  %}
os.Setenv("PUBSUB_EMULATOR_HOST", "localhost:8085")
{% endhighlight %}

Then, we have to create the topics and the subscriptions. In the case of our app, we have 2 topics with their subscriptions. The first subscription is used by our app to receive the messages. We will use the second subscription to test that the received messages have been processed and published to the destination topic. 

{% highlight go  %}

	const (
		testSub      = "in-sub"
		testSubTopic = "in-topic"
		resultTopic  = "out-topic"
		resultSub    = "out-sub"
	)

   //...

	//no error checking, since this is just a demo. also "already exist" errors can be ignored in case the test are run several times
	topic, _ := app.client.CreateTopic(app.config.context, testSubTopic)
	app.client.CreateSubscription(app.config.context, testSub, pubsub.SubscriptionConfig{Topic: topic})
	res, _ := app.client.CreateTopic(app.config.context, resultTopic)
	app.client.CreateSubscription(app.config.context, resultSub, pubsub.SubscriptionConfig{Topic: res})

{% endhighlight %}

Now we are ready to create the app instance. Since the [Receive](https://pkg.go.dev/cloud.google.com/go/pubsub?tab=doc#Subscription.Receive) method keeps listening endlessly unless a timeout or deadline is specified, our test risk running for a long time until interuppted by the user or the golang test timeout, if defined. To avoid this, we will make use of [context.WithTimeout](https://pkg.go.dev/context?tab=doc#example-WithTimeout). This will limit the time scope of the test.

{% highlight go  %}

	ctx, _ := context.WithTimeout(context.Background(), 10*time.Second)
	app, err := newApp(Config{context: ctx, gcpProjectName: "test", subscriptionName: testSub, topicName: resultTopic, options: []option.ClientOption{option.WithoutAuthentication()}})
	//we are using "github.com/stretchr/testify/assert" for assertions
	assert.Nil(t, err, "app creation is successfull")

{% endhighlight %}

now the `Receive` method will stop listening after 10 seconds and our test can resume its flow.

Prior to exectuting `app.run()`, we need to start listening on the result topic. Oh wait! `Receive` is a blocking call, so we will run our assertion logic inside a goroutine, and then run our app afterwards.

{% highlight go  %}

	go func() {
		app.client.Topic(testSubTopic).Publish(ctx, &pubsub.Message{Data: []byte("{\"greeting\" : \"hello\"}")})
		app.client.Subscription(resultSub).Receive(ctx, func(ctx context.Context, msg *pubsub.Message) {
			var jsonMessage map[string]interface{}
			json.Unmarshal(msg.Data, &jsonMessage)
			assert.Equal(t, "hello", jsonMessage["greeting"], "greeting field is kept as is")
			assert.NotEmpty(t, jsonMessage["processed_time"], "processed time field is added")
			fmt.Println("finished assertions")
		})
	}()

	app.run()

{% endhighlight %}

and there we go, we have seen how the Pub/Sub emulator can be a useful tool for integration testing and debuging without having to use the official GCP service.

Full source code can be found at: [https://github.com/zak905/gcp-pub-sub-emulator-test-demo](https://github.com/zak905/gcp-pub-sub-emulator-test-demo)