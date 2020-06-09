---
layout: post
title: "rest-assured style integration testing in Golang"
author: "Zakaria"
comments: true
description: integrations tests tips in golang 
---

Being a java developer for long years, and having used rest-assured pretty much systematically in every project for integration testing, I kind of missed its intuitive style of chaining request specifications and response assertions, when writing tests in golang. Luckily I did a lucky encouter while wandering in golang world: [Baloo](https://github.com/h2non/baloo). Baloo offers rest-assured like style of testing for golang testers minus some assertions functionalities like json path, field extraction...etc. The good news is, with a bit of hacking around, and with the help of other libraries, it's possible to achieve roughly the same. 

## The basic functionality

Baloo allows to test HTTP requests in an expressive "fluent" style like :

{% highlight go  %}
    api := baloo.New("http://integration-test.com")

	api.Get("/user/1234").
		SetHeader("Content-Type", "application/json").
		SetHeader("Authorization", "Bearer token").
		Expect(t).
		Status(200).
		Type("json").
		Done()
{% endhighlight %}

Furthermore, it's possible to do assertions on json (the exact string):

{% highlight go  %}
        api.Get("/user/1234").
		SetHeader("Content-Type", "application/json").
		SetHeader("Authorization", "Bearer token").
		Expect(t).
		Status(200).
                JSON(map[string]string{"user_email": "tester@tester.com"}).
		Type("json").
		Done()

    // or 

        api.Get("/user/1234").
		SetHeader("Content-Type", "application/json").
		SetHeader("Authorization", "Bearer token").
		Expect(t).
		Status(200).
                JSON(`{"user_email":"tester@tester.com"}`).
		Type("json").
		Done()

{% endhighlight %}

or the json schema:

{% highlight go  %}

const createUserSchema = `{
	"title": "schema for user response",
	"type": "object",
	"properties": {
	  "info": {
		"type": "string"
	  },
	  "email": {
		"type": "string"
	  },
	},
	"required": ["email"]
  }`

        api.Get("/user/1234").
		SetHeader("Content-Type", "application/json").
		SetHeader("Authorization", "Bearer token").
		Expect(t).
		Status(200).
                JSONSchema(schema).
		Type("json").
		Done()

{% endhighlight %}

However, this would only test if the fields exist and match the type. It's also possible to obtain the response and convert it to a struct or map :

{% highlight go  %}
    res, err := api.Get("/user/1234").
		SetHeader("Content-Type", "application/json").
		SetHeader("Authorization", "Bearer token").
		Expect(t).
		Status(200).
		Type("json").
		Send()

	//res is a wrapped http.response with some util functions https://github.com/h2non/gentleman/blob/master/response.go

	var responseJson map[string]interface{}

    res.JSON(&responseJson)

	//do assertions on responseJson
{% endhighlight %}

One can do with all the above functionalities, but our assertions would be more expressive if we can use json path or xml path.

## Hacking around assertions

Baloo provides an extension point to customize assertions through the `AssertFunc` function. It's enough to have a function that has the following arguments and return type: 

{% highlight go  %}
func myCustomAssertion(res *http.Response, req *http.Request) error
{% endhighlight %}

If the method returns `nil` then the assertion succeeds, otherwise a non nil error is taken as a failed assertion. 

For example (from their example [here](https://github.com/h2non/baloo/blob/master/_examples/custom_assertion/custom_test.go)):

{% highlight go  %}

func assert(res *http.Response, req *http.Request) error {
	if res.StatusCode >= 400 {
		return errors.New("Invalid server response (> 400)")
	}
	return nil
}

func TestBalooCustomAssertion(t *testing.T) {
	api.Post("/post").
		SetHeader("Foo", "Bar").
		JSON(map[string]string{"foo": "bar"}).
		Expect(t).
		Status(200).
		Type("json").
		AssertFunc(assert).
		Done()
}

{% endhighlight %}

However, we still need to create a custom function for every assertion that we need to make. To improve this, one approach is to make use of golang's "polymorphism" and create a custom assertion type with a method (not a function right ?) that has the format described above. The method would have as a receiver type our custom assertion type that will hold the assertion values: the expected, and the actual. For example:

{% highlight go  %}
type jsonPathAssertion struct {
	jsonPath string
	expected interface{}
}

func (assertion jsonPathAssertion) assertCustom1(res *http.Response, req *http.Request) error {

	//we can here access assertion.jsonPath and assertion.Expected and do something 

	return nil
}

func (assertion jsonPathAssertion) assertCustom2(res *http.Response, req *http.Request) error {
	return nil
}

//...etc

{% endhighlight %}

__Note__: The request body needs to be copied if it is to be reused again in the next assertions

In this way, we can use (and reuse) our assertions in tests like: 

{% highlight go  %}

func TestBalooCustomAssertion(t *testing.T) {
	api.Post("/post").
		SetHeader("Foo", "Bar").
		JSON(map[string]string{"foo": "bar"}).
		Expect(t).
		Status(200).
		Type("json").
		AssertFunc(jsonPathAssertion{"field1.field2", "foo"}.assertCustom1).
		AssertFunc(jsonPathAssertion{"field3", true}.assertCustom2).
		Done()
}
{% endhighlight %}

Now, let's implement the methods above (let's rename them as well). We want a method that reads the json path from the response and compare it to our expected value, and another one that looks up if a value is in an array. To do so, we will use [gjson](https://github.com/tidwall/gjson), which is a library that provides json path helpers for extracting values from a json document. 

{% highlight go  %}
func (assertion jsonPathAssertion) assertJSONPathEquals(res *http.Response, req *http.Request) error {

    //reseting the body after consuming it, to be able to use it for next assertions. 
	buf, _ := ioutil.ReadAll(res.Body)
	rdr1 := ioutil.NopCloser(bytes.NewBuffer(buf))
	res.Body = rdr1

	result := gjson.GetBytes(buf, assertion.jsonPath)

	if assertion.expected != result.Value() {
		return fmt.Errorf("%v is not equal to %v, types: %s, %s", assertion.expected, result.Value(), reflect.TypeOf(result.Value()), reflect.TypeOf(assertion.expected))
	}

	return nil
}

func (assertion testAssertion) assertJSONArrayContains(res *http.Response, req *http.Request) error {

	buf, _ := ioutil.ReadAll(res.Body)
	rdr1 := ioutil.NopCloser(bytes.NewBuffer(buf))
	res.Body = rdr1

	result := gjson.GetBytes(buf, assertion.jsonPath)

	contains := false

	for _, val := range result.Array() {
		if val.Value() == assertion.expected {
			contains = true
		}
	}

	if !contains {
		return fmt.Errorf("%v does not contain %v", result.Array(), assertion.expected)
	}

	return nil
}

{% endhighlight %}

now we can easily test for json path, and also reuse the methods for different assertions. 

{% highlight go  %}
func TestBalooCustomAssertion(t *testing.T) {
	api.Post("/post").
		SetHeader("Foo", "Bar").
		JSON(map[string]string{"foo": "bar"}).
		Expect(t).
		Status(200).
		Type("json").
		AssertFunc(jsonPathAssertion{"field1.field2", "foo"}.assertJSONPathEquals).
		AssertFunc(jsonPathAssertion{"field3", 43}.assertJSONPathEquals).
		// # here is used to extract a json array
		AssertFunc(jsonPathAssertion{"users.#.id", true}.assertJSONArrayContains).
		Done()
}
{% endhighlight %}

by using `AssertFunc`, we have built a rest-assured like testing in golang. One can plug as much helper methods as need by adding just the `jsonPathAssertion` type as a receiver type. 







