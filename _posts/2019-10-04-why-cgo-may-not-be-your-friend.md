---
layout: post
title: "Golang tale: why cgo may not be your friend?"
author: "Zakaria"
comments: true
description: some shortcomings of cgo. 
---

cgo is a set of packages and syntactic conventions that allow to call c or c++ code from Golang code. While this may be useful for wrapping c/c++ librairies, it does not come without a price, sometimes. In this post, I wanted to share some shortcomings that I faced while making use of cgo in Golang, namely memory leaks and the inability to timeout routines execution. 

# Memory leaks 

When using cgo, you are often calling code that you have not written yourself. The resources allocated by cgo code are not kept track by the golang garbage collectors, simply because c/c++ programs manage their own resources (heap, stack,..etc), and therefore if the called code does not take care of freeing memory after usage, you are likely to observe an increase of memory allocations that golang garbage collector can do nothing about. This is known as memory leaks. Let's take the program below as an example: 

{% highlight go  %}
package main

//
//  #include <stdio.h>
//  #include <stdlib.h>
//  #include <time.h>
//
//  int getRandomWithBounds(int lowerBound, int upperBound) {
//     return (rand() % (lowerBound - upperBound + 1)) + lowerBound;
//   }
//
// char* returnRandomWord() {
//	srand(time(0));
//    char  alphabet[] = {'a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'i', 'j', 'k', 'l', 'm', 'n', 'o', 'p', 'q', 'r', 's', 't', 'u', 'v', 'w', 'x', 'y', 'z'};
//
//      int length = getRandomWithBounds(0, 100);
//
//       char *word = (char *) malloc(length);
//
//      int i;
//     for (i = 0; i < length; i++) {
//	      int randomIndex = getRandomWithBounds(0, 26);
//	       word[i] = alphabet[randomIndex];
//      }
//
//		return word;
// }
//
//
import "C"
import "fmt"

func main() {
	fmt.Println(C.GoString(C.returnRandomWord()))
}
{% endhighlight %}

We can notice here that the dynamically allocated `char` pointer `char *word = (char *) malloc(length);` is not freed after usage, therefore if our program continues on calling `returnRandomWord()` a number of times (in a single execution), this will create chunks of allocated memory that cannot be accessed by the Golang garbage collector. 

Solution: if you find out that the underlying code does not free or the destroy the allocated resources, you can step in and do it yourself.

{% highlight go  %}
// previous definitions goes here
// void freeResources(char* toBeFreed) {
//   free(toBeFreed);
// }
import "C"
import "fmt"

func main() {
	generatedWord := C.returnRandomWord()
	fmt.Println(C.GoString(generatedWord))
	C.freeResources(generatedWord)
}
{% endhighlight %}

by doing so, you can save your program memory usage from growing indefinitely and eventually crashing your servers. 

# Timing out routines is not possible

Another shortcoming of using cgo is the inability to timeout routines that are known to do long computations or last for an unbound period of time. For example, some c/c++ librairies that do image processing (compression, detecting objects...) can exihibit this behavior. If you have used goroutines, you must probably be familiar with the timeout pattern:

{% highlight go  %}
doneChannel := make(chan bool)

	go func() {
		methodThatDoesHeavyComputations()
		doneChannel <- true
	}()


    select {
        case <-doneChannel:
            //do something or continue the flow of your program
            return;
        case <-time.After(20 * time.Second):
            // if got here it means that your call has timed out
            return;
	}
{% endhighlight %}

Using this pattern, a routine can be wrapped into a goroutine and interrupted using the `select` statement in case it lasts longer than a defined period of time. However, if `methodThatDoesHeavyComputations()` is only a wrapper around some c/c++ code called using cgo, the actual underlying code will not be stopped or interrupted, and therefore one would see unusual increase in cpu cycles and memory usage until the method returns, or even worse, the c/c++ can keep on running in the background forever until the server crashes. 

Solution: one possible solution can be compiling the c/c++ code directly into an executable and running it using the `os/exec` package utilities. In this scenario, one would need to find a suitable way to pass and obtain the data from/to the running program. For example, you can pass a file path to a program that compresses an image, and read the result directly from disk. The below example illustrates the pattern without getting into details: 

{% highlight go  %}
    doneChannel := make(chan bool)

	compressCommand := exec.Command("./someCompressUtility", "/path/to/file/image.png")

	go func() {
		if error := compressCommand.Run(); error {
			commandOutput, _ := compressCommand.CombinedOutput()
			log.Print(commandOutput)
			doneChannel <- false
			return
		}
		doneChannel <- true
	}()

      //same thing, except that now the process will be interrupted indeed and therefore release all the resources
        select {
        case <-doneChannel:
            //do something or continue the flow of your program
            return;
        case <-time.After(20 * time.Second):
            // if got here it means that your call has timed out
            return;
	}
{% endhighlight %}