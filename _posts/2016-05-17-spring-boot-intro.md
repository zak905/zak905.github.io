---
layout: post
title: "Quick overview of Spring boot"
author: "Zakaria"
---


If there is a word they care about at Pivotal, it is production. Deploying apps and services to production effectively is what companies strive for, sometimes without finding the perfect formula. Spring Boot is meant to overcome this challenge, for Spring framework users. Imagine you have a genie that grants your software developement wishes before you even know them,  or, using a more serious definition, a framework that helps you get rid of all the boiler plate code and configuration, and focus on things that matter. It uses your build system (Maven or Gradle) to predict the things that you can possibly do with your application, and auto configure them for you. For example, if you include a Spring Web dependency, Spring Boot gently includes an embedded Tomcat server with an 8080 default port in your jar, no need to deploy, it's already done for you. Another wonder of Spring boot is database configuration. Once you include a database driver, it auto configures the data access layer for you. It also picks up any schemas or data files (.sql) and executes them on start up. Impressive, ha? 

 Spring Boot has some downsides as well. Additional effort is required by the developper to modify the default configuration if  needed, which is not always easy. Also, a good knowledge of how Spring core framework works is required. In any case, let's give it a try.

In this tutorial, we will go through an example of how to quickly build a Rest application with an embeded database using Spring Boot.

We will start first by using [Spring initializr](https://start.spring.io/) to quicky bootstrap our app.

![Spring initializr]({{site.url}}/assets/images/SpringInitialr.png "Spring initializr")

There is a bunch of options to choose from. In our case, we chose: Web, JPA, and H2 database dependencies. We can now click on generate to download the project, and import it into an IDE ( eclipse for this tutorial).

We will then take care of the data access by creating our model object and the corresponding data access object (dao) to perform operations on the database.

{% highlight java  linenos %}
 @Entity(name="users")
public class User {
 
 @Id @GeneratedValue
 int id;
 String userName;
 String firstName;
 String lastName;
 
 public User(){}

//.. getters & setters

}
{% endhighlight %}

Using Spring data, there is no need to programtically implement the operations, all we need to do is to extend [CrudRepository](http://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/repository/CrudRepository.html).

{% highlight java  linenos %}
public interface UserDao extends CrudRepository{
      public User findByUserName(String userName);
}
{% endhighlight %}

Notice that we have added a findByUserName method, which will be automatically implemented (based on query extracted from the name of the method). Spring data provided our data layer with almost no effort. 

Next we can start implementing our Rest endpoints:

{% highlight java  linenos %}
@Controller
public class UserController {
 
 @Autowired UserDao userdao;
 
 @RequestMapping(value="/users", method=RequestMethod.GET)
 public @ResponseBody List<User> getAllUsers(){
  return (List<User>) userdao.findAll();
 }
 
 @RequestMapping(value="/users", method=RequestMethod.POST)
 public ResponseEntity<Void> addUser(@RequestBody User newUser){
     if(newUser != null){
       if( userdao.save(newUser) == null){
         return new ResponseEntity<Void>(HttpStatus.SERVICE_UNAVAILABLE);
       }else{
        return new ResponseEntity<Void>(HttpStatus.CREATED);
       }
     }else{
      return new ResponseEntity<Void>(HttpStatus.CONFLICT);
     }
 }
 
 @RequestMapping(value="/users/{userName}", method=RequestMethod.DELETE)
 public ResponseEntity<Void> deleteUser(@PathVariable String userName){
  User searchedUser = userdao.findByUserName(userName);
  if(searchedUser != null){
   userdao.delete(searchedUser);
   return new ResponseEntity<Void>(HttpStatus.OK);
  }else{
   return new ResponseEntity<Void>(HttpStatus.NOT_FOUND);
   
  }
  
 }

}
{% endhighlight %}

We defined three endpoints: for listing, adding, and deleting. For demonstration purposes, I added some sample data in `/resources` forlder, which is also automatically picked by Spring Boot and applied to the database on startup.

{% highlight java  linenos %}
insert into users values (1, 'user1', 'john', 'doe');
insert into users values (2, 'user2', 'long', 'beard');
insert into users values (3, 'user3', 'zakaria', 'amine');
{% endhighlight %}

Finally, we can launch our app by simply executing the Main class :

{% highlight java  linenos %}
@SpringBootApplication
public class SpringbootDemoApplication {

 public static void main(String[] args) {
  SpringApplication.run(SpringbootDemoApplication.class, args);
 }
}
{% endhighlight %}

Our Rest API is all set. If we go to `/users` on a browser we get:

{% highlight js  linenos %}
[
    {
        "id": 1,
        "userName": "doe",
        "firstName": "user1",
        "lastName": "john"
    },
    {
        "id": 2,
        "userName": "beard",
        "firstName": "user2",
        "lastName": "long"
    },
    {
        "id": 3,
        "userName": "amine",
        "firstName": "user3",
        "lastName": "zakaria"
    }
]

{% endhighlight %}

I created a html page with some Angular Js code to test the operations. Since the page was named "index.html",  Spring Boot automatically picked up the name and defined it as a welcome page (as the `/` context path).

I will not say more on the front end part since it is not the focus of this tutorial. You can find the source code [here](https://github.com/zak905/springboot-demo/blob/master/src/main/resources/static/index.html).

Total time in building all this: less than 10mins.

Spring Boot is a new way of getting services quickly into production, and it keeps on getting better with every release.

The whole application can be found here:  [https://github.com/zak905/springboot-demo.git](https://github.com/zak905/springboot-demo.git)