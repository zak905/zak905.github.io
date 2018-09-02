---
layout: post
title: "Open source contribution note: overview of my latest contribution to Karate (testing framework)"
author: "Zakaria"
comments: true
---

Karate is a testing framework that borrows concepts from behavior based developement testing and applies them to the context of Rest APIs. While getting familiar with the framework recently, I noticed that requests that send files (with `application/octet-stream`) using the `read` utility have a `Content-Length` of `-1`, and therefore the information carried out by the `Content-Length` header was erroneous. This may not be noticeable right away unless the target application does some validation against the `Content-Lenght` header, which was the case. After digging further, I found out that this was due to the usage of the wrong [InputStreamEntity](https://hc.apache.org/httpcomponents-core-ga/httpcore/apidocs/org/apache/http/entity/InputStreamEntity.html) (part of Apache Utils) [constructor](https://github.com/intuit/karate/blob/master/karate-apache/src/main/java/com/intuit/karate/http/apache/ApacheHttpUtils.java#L99).

{% highlight java %}

    public static HttpEntity getEntity(InputStream is, String mediaType, Charset charset) {
        try {
            return new InputStreamEntity(is, getContentType(mediaType, charset));
        } catch (Exception e) {
            throw new RuntimeException(e);
        }         
    } 
 {% endhighlight %}

When the lenght is not used in the constructor arguments, it is set to `-1` underneath. I submitted a pull request that corrects this behavior which was accepted by the Karate team. More details can be found on the PR: [https://github.com/intuit/karate/pull/492/files](https://github.com/intuit/karate/pull/492/files)
The PR is only one line of code, but it took some time to figure out the cause. It feels always good to get a PR merged into a well known os project. The fix will part of the next release. 

