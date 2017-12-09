---
layout: post
title: "Testing a Rest API in Spring Framework"
author: "Zakaria"
comments: true
---

After you have designed your Rest API and implemented it successfully, you need to test it. Testing a Rest Api is not easy because of the application context. By default, to be able to run tests, the Spring MVC container needs to be launched as it contains all of the components that the application needs to run properly ( request mapping, exception handling, ...). However, launching the container will take a considerable amount of time slowing down tests. Usually, tests need to run quicker than the application.<br>
To solve this issue, Spring test provides the speed of an out of container context with only the needed infrastructure of Spring MVC. All the infrastructure that we need to run our tests is bootstrapped in an out of container process. That is to say, we get a standalone web application that does not need to run inside a servlet, which allows unit and integration tests to run quicker. In this tutorial, we will go through how to use spring-test for testing a Rest API in Spring framework.

Requirements:

- [demo application](https://github.com/zak905/rest-spring-example)

We have the following end points that we want to test:

POST: /person<br>
GET: /person/{id}<br>
GET: /person/{id}/profile<br>
GET: /person/{id}/account<br>

Our test class looks like:

{% highlight java  linenos %}
@RunWith(SpringJUnit4ClassRunner.class)
@WebAppConfiguration
@ContextConfiguration(locations={"classpath:beans.xml"})
public class PersonControllerUnitTest {
	    @Autowired
	    private volatile WebApplicationContext webApplicationContext;

	    private volatile MockMvc mockMvc;

	    @Before
	    public void before() {
	        this.mockMvc = webAppContextSetup(this.webApplicationContext).build();
	    }
	    
	    @Test
	    public void createPersonTest() throws Exception{
	    	this.mockMvc.perform(post("/person"))
	        .andExpect(status().isCreated());
	    }
	    
	    @Test
		public void getPerson() throws Exception{
			this.mockMvc.perform(get("/person/{id}", 1))
			.andExpect(status().isOk())
			.andExpect(content().contentType("application/json;charset=UTF-8"))
			.andExpect(jsonPath("$.firstname").value("John"));
		}
		
		@Test
		public void getProfile() throws  Exception{
			this.mockMvc.perform(get("/person/{id}/profile", 1))
			.andExpect(status().isOk())
			.andExpect(content().contentType("application/json;charset=UTF-8"))
			.andExpect(jsonPath("$.description").value("Profile description"));
		}
		
		@Test
		public void getAccount() throws Exception{
			this.mockMvc.perform(get("/person/{id}/account", 1))
			.andExpect(status().isOk())
			.andExpect(content().contentType("application/json;charset=UTF-8"))
			.andExpect(jsonPath("$.username").value("johnk"));
		}

}

{% endhighlight %}

Before running tests, we have configured the application context and initialized a `MockMvc` object that mimics a Spring MVC infrastructure. Assertions were made on the return HTTP status, the content type, and also on the data returned.
