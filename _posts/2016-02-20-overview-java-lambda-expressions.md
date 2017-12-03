---
layout: post
title: "Overview of Java lambda expressions"
author: "Zakaria"
---

Lambdas are considered one of the major changes introduced to the Java language constructs since its release. Sooner or later, Java developers will need to get accustomed to using and manipulating lambdas. To put it a simple way, Lambda expressions can be refered to as a method without a name that can be declared anywhere. We know that the essence of Java is object orientation, so normally methods can only live inside an object or an interface. Lambdas were introduced to deal with this shortcoming. They add a flavor of functional programming to one of the most traditional object oriented languages.
In this tutorial, we will go through some use cases where lambda expressions may be useful.

A quick reminder of the lambdas syntax:


    Arguments declaration               Method body

    (a, b, c, ....)             ->   {expressions;...;...;}   

   or 

    (a, b, c, ....)             ->   1 expression that returns the type of the method  


## Case 1: Implementing an interface with one method (Functional interface)

{% highlight java  linenos %}
 public interface Bank {
  double convertAmountFromEuroToDollar(double amount);
 }
     //.....
  Bank bank = (b) ->   b * 1.11;
  System.out.println(bank.convertAmountFromEuroToDollar(10));
  //11.100000000000001 
  {% endhighlight %}

## Case 2: UI events (e.g callbacks)

{% highlight java  linenos %}
  JButton button = new JButton();
  button.addActionListener(
    (e) -> {System.out.println("Action performed");
     });
{% endhighlight %}

## Case 3: Streams

{% highlight java  linenos %}
  List cars = Arrays.asList("Mercedes","Volvo","Volswagen","Audi","Peugeot");
  cars.stream().
  filter(c -> c.startsWith("V"))
  .forEach(c -> System.out.println(c));

  //Result
  // Volvo
  //Volswagen

  {% endhighlight %}

## Case 4: Using Java provided functional interfaces

   {% highlight java  linenos %}  

 //Takes two String arguments and return one Boolean result
  BiFunction<String, String, Boolean> checkAnagram = (e1, e2) -> { 
      e1.toLowerCase(); e2.toLowerCase();
      char[] e1Chars = e1.toCharArray(); char[] e2Chars = e2.toCharArray(); 
      Arrays.sort(e1Chars);
      Arrays.sort(e2Chars);
     return String.valueOf(e1Chars).equals(String.valueOf(e2Chars));
   };
 
     //Takes one String arguments and return one Boolean result 
    Function<String, Boolean> checkPalindrome = (p) -> { 
 char[] toChar = p.toCharArray();
     CharBuffer buffer = CharBuffer.allocate(toChar.length);
  for(int i = toChar.length - 1; i >= 0; i--)
        buffer.put(toChar[i]);
   
           return p.equals(String.valueOf(buffer.array())); 
  };
  

  System.out.println(checkAnagram.apply("test", "estt"));
  System.out.println(checkPalindrome.apply("rotator"));
 
  //Result
  // true
  // true

  {% endhighlight %}

Full code available at:  [https://github.com/zak905/java8features/tree/master/src/opencode/java8features/lambdas](https://github.com/zak905/java8features/tree/master/src/opencode/java8features/lambdas)

Interesting reads: 

[http://zeroturnaround.com/rebellabs/java-8-best-practices-cheat-sheet/](http://zeroturnaround.com/rebellabs/java-8-best-practices-cheat-sheet/)
[http://viralpatel.net/blogs/lambda-expressions-java-tutorial/](http://viralpatel.net/blogs/lambda-expressions-java-tutorial/)



