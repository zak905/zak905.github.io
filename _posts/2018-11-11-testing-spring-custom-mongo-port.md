---
layout: post
title: "Testing a Spring Data Mongo repository with embedded Mongo on a custom port"
author: "Zakaria"
comments: true
---


# Background:

Since version 1.4, Spring Boot introduced the concept of test slices. Spring context can be costly to bootstrap with every test, especially in large applications. Test slices allows Spring to cherry pick the useful parts for a particular test. Spring provides the possibily to create custom slices annotations, but there are also several ready made test slices annotations provided by `spring-boot-starter-test` like `@WebMvcTest` which allows to test the web layer only (http calls), `@DataJpaTest` which bootstraps the jpa repositories only,  `@DataMongoTest` which bootraps Spring data mongo repositories and launches the [de.flapdoodle.embed.mongo](https://github.com/flapdoodle-oss/de.flapdoodle.embed.mongo)  embedded mongo database (should be on the test classpath), and many others. While `@DataMongoTest` provides an auto-configuration for Mongo related beans and repositories, it's also possible to customize the configurations such as the port on which Mongo runs. This post shows an example.   

# Testing a Mongo Repository:

Let's suppose we want to test the new query method that we added to our Mongo repository:

{% highlight java  %}
public interface TransactionRepository extends MongoRepository<Transaction, String> { 
     Page<Transaction> findBySuccessIsTrueAndCreatedLessThanEqualAndUserIdOrderByCreatedDesc(long created, String userId, Pageable pageRequest);
}
{% endhighlight %}


With `Transaction` being: 

{% highlight java  %}
@Document
public class Transaction {

   @Id 
   String id;

   @Indexed
   boolean success;

   @Indexed
   String userId;

   @Indexed
   long created;


   //constructors, getters and setters..etc
}
{% endhighlight %}

Our test looks like:

{% highlight java  %}

@ExtendWith(SpringExtension.class)
@DataMongoTest
public class TransactionRepositoryTest {

    @Autowired
    private TransactionRepository transactionRepository;

    private final static List<String> USER_ID_LIST = Arrays.asList("b2b1f340-cba2-11e8-ad5d-873445c542a2", "bd5dd3a4-cba2-11e8-9594-3356a2e7ef10");

    private static final Random RANDOM = new Random();


    @BeforeEach
    public void dataSetup() {
        Transaction transaction;
        for (int i = 0; i < 10; i++) {
            String requestId = UUID.randomUUID().toString();
            if (i % 2 == 0) {
                transaction = new Transaction(requestId, true, USER_ID_LIST.get(RANDOM.nextInt(2)), System.currentTimeMillis());
            } else {
                transaction = new Transaction(requestId, false, USER_ID_LIST.get(RANDOM.nextInt(2)), System.currentTimeMillis());
            }

            transactionRepository.save(transaction);
        }
    }


    @Test
    public void findSuccessfullOperationsForUserWithCreatedDateLessThanNowTest() {
        long now = System.currentTimeMillis();
        String userId = USER_ID_LIST.get(RANDOM.nextInt(2));
        List<Transaction> resultsPage =  transactionRepository.findBySuccessIsTrueAndCreatedLessThanEqualAndUserIdOrderByCreatedDesc(now, userId, PageRequest.of(0, 5)).getContent();

        assertThat(resultsPage).isNotEmpty();
        assertThat(resultsPage).extracting("userId").allMatch(id -> Objects.equals(id, userId));
        assertThat(resultsPage).extracting("created").isSortedAccordingTo(Collections.reverseOrder());
        assertThat(resultsPage).extracting("created").first().matches(createdTimeStamp -> (Long)createdTimeStamp <= now);
        assertThat(resultsPage).extracting("success").allMatch(sucessfull -> (Boolean)sucessfull == true);
    }
}

{% endhighlight %}


# Customzing the config:

Everything is ok so far. Now, suppose we want to customize our mongo configuration e.g the port from which mongo is launched. We have to exclude the Mongo auto-configuration first: 

{% highlight java  %}
@DataMongoTest(excludeAutoConfiguration= {EmbeddedMongoAutoConfiguration.class})
{% endhighlight %}

And then we can provide our own custom configuration: 


{% highlight java  %}

    @Configuration
    static class MongoConfiguration implements InitializingBean, DisposableBean {

         MongodExecutable executable;

        @Override
        public void afterPropertiesSet() throws Exception {
            String host = "localhost";
            int port = 27019;

            IMongodConfig mongodConfig = new MongodConfigBuilder().version(Version.Main.PRODUCTION)
                    .net(new Net(host, port, Network.localhostIsIPv6()))
                    .build();

            MongodStarter starter = MongodStarter.getDefaultInstance();
            executable = starter.prepare(mongodConfig);
            executable.start();
        }


        @Bean
        public MongoDbFactory factory() {
            // also possible to connect to a remote or real MongoDB instance
            MongoDbFactory mongoDbFactory = new SimpleMongoDbFactory(new MongoClientURI("mongodb://localhost:27019/test_db"));
            return mongoDbFactory;
        }


        @Bean
        public MongoTemplate mongoTemplate(MongoDbFactory mongoDbFactory) {
            MongoTemplate template = new MongoTemplate(mongoDbFactory);
            template.setWriteConcern(WriteConcern.ACKNOWLEDGED);
            return template;
        }

        @Bean
        public MongoRepositoryFactoryBean mongoFactoryRepositoryBean(MongoTemplate template) {
            MongoRepositoryFactoryBean mongoDbFactoryBean = new MongoRepositoryFactoryBean(TransactionRepository.class);
            mongoDbFactoryBean.setMongoOperations(template);

            return mongoDbFactoryBean;
        }

        @Override
        public void destroy() throws Exception {
            executable.stop();
        }
    }

{% endhighlight %}

The `MongoDbFactory`, `MongoTemplate`, `MongoRepositoryFactoryBean` are the building blocks of Spring data configuration for MongoDB. Additionally, we have extended `InitializingBean`, `DisposableBean` classes for convenience to start/stop the Embedded Mongo instance after the `MongoConfiguration` bean creation/disposal. It is also possible to launch/stop the instance in methods annotated with `@BeforeAll`, `@AfterAll` JUnit annotations in the main test class. 

Now, our Mongo will launched at port 27019.  

The full example can be found here: [https://github.com/zak905/mongo-spring-test-demo](https://github.com/zak905/mongo-spring-test-demo)