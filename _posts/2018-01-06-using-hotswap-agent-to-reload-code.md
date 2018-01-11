---
layout: post
title: "Using HotSwap agent for hot reloading Java code"
author: "Zakaria"
comments: true
---


Java language and JVM languages in general are known to be simultaneously compiled and interpreted languages. The compiling phase involves compiling the source code to byte code, and then the byte code is interpreted (executed) by the JVM. Unlike interpreted languages (e.g JavaScript, php), which can be instantly reloaded, Java and other JVM languages may take the developer into a tedious cycle of coding-compiling-deploying. Luckily, there are some tools that can help the developers break this cycle and hot reload code changes without relaunching/redeploying the application: [HotSwap Agent](https://github.com/HotswapProjects/HotswapAgent) is one of them. 

HotSwap agent uses [instrumentation](https://docs.oracle.com/javase/7/docs/api/java/lang/instrument/package-summary.html)   (introduced in Java 5) to manipulate the byte code running on the JVM. 

# Installation and setup: 

In order not reinvent the wheel, all the installation and set up can be found in HotSwap agent [starting guide](http://hotswapagent.org/mydoc_quickstart.html).


# Usage examples: 

The most important file when working with HotSwap agent is the `hotswap-agent.properties`. This file tells the agent which resources to watch, and can also provide optional configuration properties like logging. Description of the configuration properties can be found [here](https://github.com/HotswapProjects/HotswapAgent/blob/master/hotswap-agent-core/src/main/resources/hotswap-agent.properties). 

1. <ins>Jersey application deployed on Tomcat</ins>:

    To use HotSwap agent on tomcat, the `hotswap-agent.properties` needs to be on the classpath. The ideal place to put it is `src/main/resources` on a regular java project structure. Then, we need to make HotSwap watch the classes (.class) files deployed on Tomcat. To achieve that, we need to point the classpath property to where the classes are, which is usually `WEB-INF/classes` for webapps: 

    
        extraClasspath=/WEB-INF/classes
        autoHotswap=true 
    

    Then, the HotSwap JVM arguments need to be added to Tomcat. The best way to do it is to create a `setenv.sh` file inside the `bin` Tomcat folder. 

    
       export JAVA_OPTS="$JAVA_OPTS -XXaltjvm=dcevm -javaagent:/your/path/to/agent/hotswap-agent-1.0.jar"
    

    After this setup, the classes can be hot reloaded directly by recompiling the app, and pointing the compile result to where the classes are deployed on the Tomcat, in our case `/WEB-INF/classes`. In Maven for example, setting the `outputDirectory` can do the trick:  

    
        <build>
        <outputDirectory>your/path/to/Tomcat/webapps/app/WEB-INF/classes</outputDirectory>
            <!-- other stuff goes here -->
        </build>
     

    For convenience, it's better to create a separate build profile for hot reloading. 

    After running `mvn compile`, you will see the following message in the console:

     
        HOTSWAP AGENT: 21:06:00.276 RELOAD (org.hotswap.agent.config.PluginManager) - Reloading classes [com.gwidgets.resource.MyResource] (autoHotswap)
        HOTSWAP AGENT: 21:06:00.797 INFO (org.hotswap.agent.plugin.jersey2.Jersey2Plugin) - Reloaded Jersey Containers
     

    This means that your new changes has been reloaded.   

2. <ins>Spring Boot application</ins>: 

In  a Spring Boot application, the task is even easier. We only need (optionally) to set the auto `autoHotswap=true` and put `hotswap-agent.properties` in `src/main/resources`. Finally, we can run the app from the IDE with the HotSwap agent JVM arguments: `-XXaltjvm=dcevm -javaagent:/your/path/to/agent/hotswap-agent-1.0.jar`. The modified `.java` files will be automatically reloaded when saved to disk, without even the need to recompile. 

# Wrap up:

HotSwap agent is a free open source tool that help java developers save precious time, and reduce the development cycle. It does not support all types of code changes (e.g adding/removing class annotations), but it can be an indispensable tool especially when the compile/deploy time is not negligable. The list of the supported code changes is provided by the official page: http://hotswapagent.org/.  HotSwap agent provides plugins for well-know Java librairies and frameworks like Spring, Jersey, Hibernate, JSF and many others. If your project technology stack is not supported then looking at proprietary tools like [JRebel](https://zeroturnaround.com/software/jrebel/) may be an option, but it does come with a price tag. 