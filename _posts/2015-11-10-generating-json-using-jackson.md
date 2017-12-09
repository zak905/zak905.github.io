---
layout: post
title: "Generating Json using Jackson"
author: "Zakaria"
comments: true
---

JSON is the number one format when it comes to exchanging data over the Web. In this tutorial, we will go through how to generate JSON content, using Jackson library.

First of all, there are two different ways of generating JSON with Jackson:
- by mapping an object
- Using a stream

Let's try out both of them.

The maven dependency for Jackson is :

{% highlight xml %}
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.5.4</version>
</dependency>
{% endhighlight %}

We are using an object such as :

{% highlight java  linenos %}
public class Concept {
 
 private String conceptName;
 private int score;
 private List conceptOwnerList;
 private Map conceptDataDictionary;
 
 
 public Concept(String name){
  this.conceptName = name;
  conceptOwnerList = new ArrayList();
  conceptDataDictionary = new HashMap();
 }
 
 public String getConceptName() {
  return conceptName;
 }

 public void setConceptName(String conceptName) {
  this.conceptName = conceptName;
 }
 
 public int getScore() {
  return score;
 }

public void setScore(int score) {
 this.score = score;
}
 public List getConceptOwner() {
  return conceptOwnerList;
 }

 public void addConceptOwner(String conceptOwner) {
  conceptOwnerList.add(conceptOwner);
 }
 public Map getConceptDataDictionary() {
  return conceptDataDictionary;
 }
 public void addConceptToDataDictionary(String entry, String definition) {
  conceptDataDictionary.put(entry, definition);
 } 

}
{% endhighlight %}


Let's populate our object with some sample data, and serialize it to JSON using [ObjectMapper](https://fasterxml.github.io/jackson-databind/javadoc/2.0.0/com/fasterxml/jackson/databind/ObjectMapper.html).

{% highlight java  linenos %}
Concept concept = new Concept("MyConcept");
concept.setScore(50);
concept.addConceptOwner("tester1");
concept.addConceptOwner("tester2");
concept.addConceptOwner("tester3");
        
concept.addConceptToDataDictionary("Peniciline", "group of antibiotics");
concept.addConceptToDataDictionary("Amoxiline", "antibiotic");
concept.addConceptToDataDictionary("Morphine", "pain killer");
        
ObjectMapper mapper = new ObjectMapper();
mapper.configure(SerializationFeature.INDENT_OUTPUT, true);
  
String json = mapper.writeValueAsString(concept);
  
System.out.println(json);
{% endhighlight %}

Result:

{% highlight js %}
{
  "conceptName": "MyConcept",
  "score": 50,
  "conceptOwner": ["tester1", "tester2", "tester3"],
  "Peniciline": "group of antibiotics",
  "Amoxiline": "antibiotic",
  "Morphine": "pain killer"
}
{% endhighlight %}


Some useful annotations for object properties:

[@JsonAnyGetter](http://fasterxml.github.io/jackson-annotations/javadoc/2.1.0/com/fasterxml/jackson/annotation/JsonAnyGetter.html): Annotation to add to the getter of a Map. It serializes the content of the map instead of using the map as an object itself.

[@JsonIgnore](http://fasterxml.github.io/jackson-annotations/javadoc/2.0.2/com/fasterxml/jackson/annotation/JsonIgnore.html): Ignores a property when serializing.

[@JsonProperty](http://fasterxml.github.io/jackson-annotations/javadoc/2.1.0/com/fasterxml/jackson/annotation/JsonProperty.html): uses "someproperty" as key value instead of the property name.

A JSON stream can be useful when the developper needs to build the content interactively:

{% highlight java  linenos %}
JsonFactory factory = new JsonFactory();
JsonGenerator generator = factory.createGenerator(new PrintWriter(System.out));
generator.writeStartObject();
generator.writeFieldName("JsonTest");
generator.writeString("using a stream json generator");
generator.writeFieldName("ArrayKey");
        // start an array
generator.writeStartArray();   
generator.writeString("object 1");
generator.writeString("object 2");
generator.writeString("object 3");
        
generator.writeEndArray();
generator.writeEndObject();
 
generator.close();
{% endhighlight %}

Result:

{% highlight js %}
{
  "JsonTest": "using a stream json generator",
  "ArrayKey": ["object 1", "object 2", "object 3"]
}
{% endhighlight %}