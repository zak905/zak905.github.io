---
layout: post
title: "the Decorator design pattern: a real world java example"
author: "Zakaria"
comments: true
---

The decorator design pattern allows complementing functionalities of an existing interface without altering the original behavior. It is a common pattern, and it can be used, for example, to add logging, caching, notifications to an exisiting interface while preseving modularity and loose coupling. Let's assume we have the following interface: 

  {% highlight java  linenos %}
   public interface UsersService {
     public final static List<User> userDB = Arrays.asList(new User(1, "user1"), new User(2, "user2"));

     User readUserById(int id);

     void registerUser(User user);
   }
 {% endhighlight %}

 with a `User` class:

  {% highlight java  linenos %}
   public class User {
       int id;

       String name;

       public User(int id, String name) {
            this.id = id;
            this.name = name;
       }
       
       public int getId() {
           return this.id;
       }

       public String getName() {
           return this.name;
       }
   }
  {% endhighlight %}

and the following implementation:
  {% highlight java  linenos %}
   public class UsersServiceBasic implements UsersService  { 

    public User readUserById(int id) {
        Optional<User> searchedUser = UsersService.userDB.stream().filter(user -> user.getId() == id).findFirst();
        return searchedUser.isPresent() ? searchedUser.get() : null ;
    }

    public void registerUser(User user) {
        UsersService.userDB.add(user);
    }
   }
  {% endhighlight %}

  We want to add logging and caching to `UsersServiceBasic` without modifying the initial behavior. 

# Adding Logging:

 {% highlight java  linenos %}
 public abstract class AbstractLoggingService {
       //do some logging initialization here
       // for the sake of the example we are just using print

       public void logInfo(String message) {
         System.out.println("INFO " + message);
       }
 } 
 {% endhighlight %}

 We can now implement the `UsersService` and add a logging functionality.

 {% highlight java linenos %}
 public class UsersServiceWithLogging extends AbstractLoggingService implements UsersService {

     UsersService decoratedUsersService;

   public UsersServiceWithLogging(UsersService decoratedUsersService) {
          this.decoratedUsersService = decoratedUsersService;
   }

    public User readUserById(int id) {
        logInfo("looking for user with id " + id);
        User user = decoratedUsersService.readUserById(id);
        String message = user != null ? "Found user" : "Not found, returning null";
        logInfo(message);
        return user;
    }

    public void registerUser(User user) {
        decoratedUsersService.registerUser(user);
        logInfo("registred new user");
    }
 }
 {% endhighlight %}

 Let's do a simple test :

{% highlight java  linenos %}

UsersService basicService = new UsersServiceBasic();

UsersService usersServiceWithLogging = new UsersServiceWithLogging(basicService);

usersServiceWithLogging.readUserById(10);

 {% endhighlight %}

 Result : 

 ```
 INFO looking for user with id 10
 INFO Not found, returning null
 ```

   {% highlight java  linenos %}
   usersServiceWithLogging.readUserById(1);
  {% endhighlight %}

   Result : 

 ```
 INFO looking for user with id 1
 INFO Found user
 ```

# Adding caching:

 {% highlight java  linenos %}
 public abstract class AbstractCachingService {
        
        Map<Integer, User> cache = new HashMap<>();

       public void cache(User user) {
         cache.put(user.getId(), user);
       }

       public User retrieveFromCache(int id) {
           return cache.get(id);
       }
 } 
 {% endhighlight %}


  We can now implement the `UsersService` and add a caching functionality.


  {% highlight java linenos %}
 public class UsersServiceWithCaching extends AbstractCachingService implements UsersService {

     UsersService decoratedUsersService;

   public UsersServiceWithCaching(UsersService decoratedUsersService) {
          this.decoratedUsersService = decoratedUsersService;
   }

    public User readUserById(int id) {
        User user = retrieveFromCache(id);
        if (user != null ) {
         return user;
        }

        return decoratedUsersService.readUserById(id);
    }

    public void registerUser(User user) {
        decoratedUsersService.registerUser(user);
        cache(user);
    }
 }
 {% endhighlight %}


# Combining several decorators:

It is also possible to combine several decorators or, in other words, decorating a decorator. In this example, we can, for example, add logging functionality to the caching service. The caching service can be passed to the logging service or vice versa.

{% highlight java linenos%}

UsersService basicService = new UsersServiceBasic();

UsersService usersServiceWithLogging = new UsersServiceWithLogging(basicService);

UsersService usersServiceWithCaching = new UsersServiceWithCaching(basicService);

UsersService usersServiceWithCachingAndLogging = new UsersServiceWithCaching(usersServiceWithLogging);

 {% endhighlight %}