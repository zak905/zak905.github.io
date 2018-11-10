---
layout: post
title: "Good reasons to use Karate for integration testing"
author: "Zakaria"
comments: true
---

In the previous [post](http://www.zakariaamine.com/2018-09-01/karatedsl-contribution), I talked briefly about [Karate DSL](https://github.com/intuit/karate) testing framework and its usage. In this post, I would like to share my opinion about why Karate can be a good alternative for testing (especially a Rest API). I started working recently with Karate and I noticed some general improvements in my tests coverage and execution time.  

# Makes you less lazy to write tests, and promotes code reuse

In large projects, sometimes one refrain from testing more scenarios because he thinks he has done enough testing. Some testing scenarios are close and you, as developer, think that there is no need to write an additional test, and this can result in a decrease in coverage. Karate DSL format is simple and intuitive, and can help you write tests with the minimum amount of effort and code. This can be encouraging and less frustrating for developers who write tests. 

With Karate, it is easy to copy and reuse close test scenarios because the changes required to adapt tests are minimal.  

# Language agnostic

Karate is written in java and targets java projects mainly, but this does not mean it should be used exclusively for java. Karate offers a way to run tests in standalone mode without the need for a testing framework like JUnit. Using the standalone jar, the tests can be run in any environment, and regardless of what languages the APIs are written in, using a simple `java` command:

```
java -jar -Dkarate.config.dir=test/ -Dlogback.configurationFile=test/log-config.xml  test/karate.jar -T 4 -e dev -o test_results test/integration/mytest-service.feature
```

# Tests can be easily parallelizable

When the number of tests gets big, you know that you have no choice but running tests in parrallel to save time and reduce the CI/CD cycles duration. This is important for making quick fixes and releases. Trying to write tests that can be parralelized is not an easy task because often tests share some parameters and variables. Using Karate relieves you from this task, as the test scenario are independent from one another and thus running tests in parralel can be done out of the box. The `-T 4` argument in previous example denotes the number of threads used to launch tests. 

# Tests are more readable

When it comes to the data sent to or received from Rest APIs, the most used format nowadays is JSON (xml is also supported). When you write your tests in Java (or any other language), most of time developpers use a POJO and convert it to JSON using a serializer, which is not always optimal for readability. Karate offers many ways to create and declare JSON as-is, and in the same way as sent from the calling side. This improves readability and can help in error shooting. For example: 

```
Scenario: request with valid payload succeeds 

* def payload = 
"""
{
    "url": "http://www.zakariaamine.com/logo.png",
    "alpha": 0.5,
    "targetFormat": "png",
    "mode": "rtl"
}
"""

Given url 'http://myimageConverterServer/convertImage'
And request payload
And header Content-Type = 'application/json'
And header X-Imager-Key = apiKey
When method post
Then status 200
Then match header Content-Type == 'image/png'
```

# Wrap up

With this being said, I barely scratched the surface of Karate can do. Karate has more to offer and may be the new alternative for making integration tests simple and efficient. With the growing community around it, Karate is growing fast and may become a standard for testing Rest APIs.   