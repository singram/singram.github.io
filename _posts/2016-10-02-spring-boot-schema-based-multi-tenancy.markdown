---
layout: post
title: Spring-boot Schema based multi tenancy
date: '2016-10-02 13:06:26'
tags:
- springboot
- hibernate
---


>**TLDR;** This article will explain multi tenancy, focusing in on the SCHEMA strategy and how to implement it in two simple steps using Spring Boot and Hibernate.

Multi-tenancy is the sharing of process and infrastructure across multiple customers or tenants efficiently.  The alternative to this is having a siloed application environment per customer.  This brings it's own problems such as;

- Linearly scaling infrastructure costs (assuming equal customers).
- Inefficient use of infrastructure.
- Divergent infrastructure & configuration without strict infrastructure automation and change management.
- High overheads to keep multiple environments up to date and in sync.
- Opens the door to solution forks under business pressure which incurs huge technical debt and operational overhead as teams now must support multiple solution versions.

Ofttimes multi tenancy offers the easiest way to scale customer growth while minimizing infrastructure and operational costs.

There are a few principle approaches to multi tenancy;
####Discriminator Strategy
The discriminator pattern works on a single database service and single schema for all tenants. Constituent tenants are discriminated by a specific strategy such as a `tenant_id` field embedded in tables containing tenant specific data.  Beyond the below pro's/con's this strategy is a non-starter for use case which legally require 'air-space' between tenants.
**Pros**

- Single database and schema instance to manage
- Single schema to backup
- Single schema to archive, upgrade etc.
- Simple reporting across tenants (e.g. `SELECT .... GROUP BY tenant_id`)
- Single database service account to manage per application.
- Single database instance to tune and maintain.

**Cons**

- Tenant data is interwoven meaning backup & restore is an all or nothing proposition.
- Care needs to be taken with every database interaction that the data returned is appropriately scoped.
- If your database goes down, all your customers go down, therefor necessitating a high availability strategy which is generally a good idea but essential in this strategy.
- If a table becomes corrupted it becomes corrupted for all users.
- If a tenant leaves, it can be tricky to extract and archive the information
 - If that tenant comes back it can be trickier to reinsert the data and easier to integrate from scratch.
 - While storage is cheap, performance is not and an inactive tenant in a single schema will take up database buffer pool resources simply by it's existence through indices alone.
- Because it is likely that a single service account will be used to access the schema and all tenants reside in the schema it can be challenging to trace database load to specific tenant usage.
- As a single database service is serving all tenants, performance is subject to "noisy neighbors".

Scaling can be problematic depending on the underlying storage technology chosen due to the monolithic nature of the schema.  If a traditional [RDBMS](https://en.wikipedia.org/wiki/Relational_database_management_system) is chosen replicas can be employed for read scaling and a sharding strategy employed for write scaling.  If using a RDBMS this particular strategy lends itself well to use cases where historic data can be archived leaving just hot data in the primary database system.  These considerations change if using a NoSQL technology such as [AWS Aurora](https://aws.amazon.com/rds/aurora/) or [MongoDB](https://www.mongodb.com/) where r/w scaling is handled transparently as part of the storage service layer and not a concern of the application itself.  In addition to this schema upgrades can be challenging based on the volume of potential data and all customers being affected simultaneously.  Even with a backing technology supporting 'online schema updates' the application may have to consider supporting multiple data schema versions until the schema update is complete.


####Schema Strategy
The schema strategy employs a single database server like the `DISCRIMINATOR` strategy but specifies a schema instance per tenant meaning that each tenant has complete isolation at the data layer from other tenants.
**Pros**

- Tenant data is robustly isolated from other tenant data
 - This in turn means for simpler more robust application development.  However the application must be tenant aware and capable of switching tenants reliably.
 - Schema & table corruption affects only a single tenant 
 - Ad-hoc queries are automatically scoped to a single tenant.
- Granular backups can be taken and restored with ease & in parallel.
- Tenants can be migrated to and from different environments easily.
- Instrumentation is available on a per schema basis allowing the attribution of load and bottlenecks to specific tenant generated load.
- Single database service account to manage per application.
- Single database instance to tune and maintain.

**Cons**

- As a single database service is serving all tenants, performance is subject to noisy neighbors similar to the `DISCRIMINATOR` strategy.  However it is trivial to move problem customers onto dedicated databases should the need arise.
- If your database goes down, all your customers go down, again necessitating a good failover strategy.
- Tooling needs to be built to handle schema updates, backups and restores of the tenant schemas with an environment.
- Reporting across tenants requires additional tooling.
- De-normalization of common reference tables may be necessary or a 'common/admin' schema employed and shared by all tenants.  This in itself can assist in some of the maintenance tooling mentioned.


####Database Strategy
The database strategy takes the `SCHEMA` strategy one step further whereby each tenant has a separate schema instance on a separate database.

**Pros**

- Tenant data is robustly isolated from other tenant data
 - This in turn means for simpler more robust application development.  
 - Schema & table corruption affects only a single tenant
- Granular backups can be taken and restored with ease & in parallel.
- Tenants can be migrated to and from environments easily.
- Instrumentation is available on a per schema basis allowing the attribution of load and bottlenecks to specific tenant generated load.
- "Noisy neighbor" problems are eliminated at the database layer.

**Cons**

- Multiple databases instances to tune and maintain.
- Additional infrastructure cost of the multiple database instances.
- A connection pool per tenant per application is now required (assuming the application layer is multi tenant) which may require additional tuning when considering the number of application instances you need to scale to and the overhead each connection incurs on your storage service.
- Multiple database service accounts to manage per application.
 - This assumes that an application will switch between tenants and therefor need connection credentials to all databases making this strategy equal from a security standpoint to a single service account.
- If a database goes down, only a single tenant is affected.
- Tooling needs to be built to handle schema updates, backups and restores of the entire environment.
- Reporting across tenants requires additional tooling.
 - This may be complicated by the multiple service accounts to connect with each database.

####Concluding strategy thoughts
The pro/cons for each strategy are entirely subjective to the use-case under consideration.  From a general standpoint I personally favor the `SCHEMA` approach having seen it work successfully in production many times.  I also believe it strikes the right balance between pragmatic pros & cons as well as offering architectural escape routes should performance and scaling problems arise.

Further reading can be found [here](https://docs.jboss.org/hibernate/orm/5.2/userguide/html_single/Hibernate_User_Guide.html#multitenacy) on the [Hibernate](http://hibernate.org/) website which is the default ORM for [SpringBoot](https://projects.spring.io/spring-boot/) applications

### Implementing the SCHEMA strategy

So now we've taken a quick high-level tour of the main multi tenant strategies lets run through what it takes to add one to a typical Spring Boot application.  Here we'll be employing the `SCHEMA` strategy.  It's actually surprising how trivial and flexible it is.

As a quick side note, while `SCHEMA` & `DATABASE` strategies are supported as of Hibernate 4.1, support for the `DISCRIMINATOR` pattern was introduced in 5.x (see [here](https://docs.jboss.org/hibernate/orm/4.2/devguide/en-US/html/ch16.html#d5e4780) for more details)

#### Step 1. Tenant awareness

So first thing's first.  For an application to be multi tenant it must have a way to detect and store the correct tenant for the transaction it is serving.

For the purposes of this entry we will assume a simple tenant naming schema where the name of the `tenant id` matches the name of the tenant schema in the database.  We will also assume we are starting from a simple SpringBoot MVC CRUD application with [RESTful](https://en.wikipedia.org/wiki/Representational_state_transfer) API.  A basic example can be found on the SpringBoot guide page [here](http://spring.io/guides/gs/rest-service/) or you can look at the full working example documented here on [github](https://github.com/singram/spring-boot-multitenant/).
The following will serve as our tenant storage interface, storing the tenant as data against the current thread (see [here](http://stackoverflow.com/questions/817856/when-and-how-should-i-use-a-threadlocal-variable) for more information on `ThreadLocal` usage).

```language-java
public class TenantContext {

  final public static String DEFAULT_TENANT = "test";

  private static ThreadLocal<String> currentTenant = new ThreadLocal<String>()
  {
    @Override
    protected String initialValue() {
      return DEFAULT_TENANT;
    }
  };

  public static void setCurrentTenant(String tenant) {
    currentTenant.set(tenant);
  }

  public static String getCurrentTenant() {
    return currentTenant.get();
  }

  public static void clear() {
    currentTenant.remove();
  }
}
```

One thing to note here is the `DEFAULT_TENANT`.  This is necessary from a Spring framework point of view to initialize the connection pool to the database and Hibernate (see later) will complain on initial startup of the application is this is null and a multi-tenant strategy is in place.  This can be implemented much cleaner in Java 8+ than in the code sample above.  The `DEFAULT_TENANT` could be a real tenant but if that makes you uneasy you could use a demo/empty tenant or your architecture may have the concept of a shared 'master' database for centralized tenant and shared dictionary management.

**But how does this get set?**  Our tenant could passed in the header, subdomain (e.g. http://tenantid.myapp.com/....), URI (e.g. http://myapp.com/tenant_id/....) cookie or ideally as part of the authentication strategy such as a property in a [JWT](https://jwt.io/).

For this example we will use a simple http header property (`X-TenantID`).  You should absolutely **not** use this strategy in any production application under any circumstances, this approach is purely to simplify the concepts.

Regardless the vehicle for the tenant data, it is desirable to have the multi tenant mechanics isolated away from, and as invisible to, the main application as much as possible.  For instance, no tenant specific business logic should ever be visible in the controllers.  To this end, the HTTP [HandlerInterceptorAdapter](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/servlet/handler/HandlerInterceptorAdapter.html) class is perfect for this and requires two additions to our application; the interceptor itself and the configuration to hook the interceptor in.

```language-java
@Component
public class TenantInterceptor extends HandlerInterceptorAdapter {

  private static final String TENANT_HEADER = "X-TenantID";

  @Override
  public boolean preHandle(HttpServletRequest req, HttpServletResponse res, Object handler)
      throws Exception {

    String tenant = req.getHeader(TENANT_HEADER);
    boolean tenantSet = false;

    if(StringUtils.isEmpty(tenant)) {
      res.setStatus(HttpServletResponse.SC_BAD_REQUEST);
      res.setContentType(MediaType.APPLICATION_JSON_VALUE);
      res.getWriter().write("{\"error\": \"No tenant supplied\"}");
      res.getWriter().flush();
    } else {
      TenantContext.setCurrentTenant(tenant);
      tenantSet = true;
    }

    return tenantSet;
  }

  @Override
  public void postHandle(
      HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView)
          throws Exception {
    TenantContext.clear();
  }
```

In the interceptor above, note the logic to return an appropriate response code and message body if a tenant is missing.  This logic becomes unnecessary if the `tenant` is part of the authentication schema and securely transmitted in a JWT for instance which, by definition, is generated by a trusted entity.

And finally the configuration to wire the interceptor in;

```language-java
@Configuration
public class WebMvcConfig extends WebMvcConfigurerAdapter {

  @Autowired
  HandlerInterceptor tenantInterceptor;

  @Override
  public void addInterceptors(InterceptorRegistry registry) {
    registry.addInterceptor(tenantInterceptor);
  }
}
```

It's interesting to note that interceptors can be applied to specific URL path patterns which opens up the possibility of different tenant strategies for different parts of the application.  For instance everything under `\admin` could be handled by a different tenant interceptor which could force the tenant id to `ADMIN` and use a schema dedicated to centralized management of all the tenants in the system.

At this point, you can test your progress with `curl` and a simple endpoint responding to `GET`.

Without a `X-TenantID` header
```
$ curl -v localhost:8080/person/1 | jq .
*   Trying 127.0.0.1...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0* Connected to localhost (127.0.0.1) port 8080 (#0)
> GET /person/1 HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.45.0
> Accept: */*
> 
< HTTP/1.1 400 
< X-Application-Context: application
< Content-Type: application/json;charset=ISO-8859-1
< Transfer-Encoding: chunked
< Date: Thu, 29 Sep 2016 15:04:36 GMT
< Connection: close
< 
{ [37 bytes data]
100    31    0    31    0     0   1880      0 --:--:-- --:--:-- --:--:--  2066
* Closing connection 0
{
  "error": "No tenant supplied"
}

```
`X-TenantID` doesn't do anything at this point, we are simply detecting and storing the desired tenant context.  So with any `X-TenantID` header you should see the following
```
$ curl -v -H "X-TenantID:foo" localhost:8080/person/1 | jq .
*   Trying 127.0.0.1...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0* Connected to localhost (127.0.0.1) port 8080 (#0)
> GET /person/1 HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.45.0
> Accept: */*
> X-TenantID:test
> 
< HTTP/1.1 200 
< X-Application-Context: application
< Content-Type: application/hal+json;charset=UTF-8
< Transfer-Encoding: chunked
< Date: Thu, 29 Sep 2016 15:04:30 GMT
< 
{ [244 bytes data]
100   238    0   238    0     0   5803      0 --:--:-- --:--:-- --:--:--  5950
* Connection #0 to host localhost left intact
{
  "_links": {
    "self": {
      "href": "http://localhost:8080/person/1"
    }
  },
  "lastName": "Baggins",
  "firstName": "Frodo",
  "updatedAt": "2016-09-25T23:01:10.000+0000",
  "createdAt": "2016-09-25T23:01:10.000+0000"
}
```

#### Step 2. Hibernate schema changing

So now we have the tenant context we need to change the schema transparently and reliably.  Remember we do not want to burden developers with the concern of interacting with the correct context at the detriment of business logic and feature simplicity and scope.  This is a great example of Aspect Oriented Programming ([AOP](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/aop.html)).  To this end, as mentioned previously, Hibernate natively supports `SCHEMA` based multi tenancy and requires three main components.

- **CurrentTenantIdentifierResolver** - Class responsible for resolving the correct tenant
- **MultiTenantConnectionProvider** - Class responsible for providing and closing tenant connections
- **Configuration** - Wiring up Hibernate correctly

The `CurrentTenantIdentifierResolver` is remarkably straight forward and essentially, in this case, a proxy to our `TenantContext` class.  This would be an appropriate place to handle any transformations necessary between the `tenant id` and the database schema name for the tenant.  In this example there is a one to one match between the tenant id and schema name so no transformation is necessary but that would most likely not be true in a real production app.  Often a naming convention to clearly identify tenant schemas will be useful in a growing production application.
```language-java
@Component
public class CurrentTenantIdentifierResolverImpl implements CurrentTenantIdentifierResolver {

  @Override
  public String resolveCurrentTenantIdentifier() {
    return TenantContext.getCurrentTenant();
  }

  @Override
  public boolean validateExistingCurrentSessions() {
    return true;
  }
}
```

The `MultiTenantConnectionProvider` is again remarkably simple.  Here we are using [Mysql](https://www.mysql.com/) as the backing store and the standard `USE database;` SQL statement to change schemas which is very cheap to use from a database cost/performance standpoint.  Errors such as the tenant database not existing are propagated up the stack in this example.
```language-java
@Component
public class MultiTenantConnectionProviderImpl implements MultiTenantConnectionProvider {
  private static final long serialVersionUID = 6246085840652870138L;

  @Autowired
  private DataSource dataSource;

  @Override
  public Connection getAnyConnection() throws SQLException {
    return dataSource.getConnection();
  }

  @Override
  public void releaseAnyConnection(Connection connection) throws SQLException {
    connection.close();
  }

  @Override
  public Connection getConnection(String tenantIdentifier) throws SQLException {
    final Connection connection = getAnyConnection();
    try {
      connection.createStatement().execute( "USE " + tenantIdentifier );
    }
    catch ( SQLException e ) {
      throw new HibernateException(
          "Could not alter JDBC connection to specified schema [" + tenantIdentifier + "]",
          e
          );
    }
    return connection;
  }

  @Override
  public void releaseConnection(String tenantIdentifier, Connection connection) throws SQLException {
    try {
      connection.createStatement().execute( "USE " + TenantContext.DEFAULT_TENANT );
    }
    catch ( SQLException e ) {
      throw new HibernateException(
          "Could not alter JDBC connection to specified schema [" + tenantIdentifier + "]",
          e
          );
    }
    connection.close();
  }

  @SuppressWarnings("rawtypes")
  @Override
  public boolean isUnwrappableAs(Class unwrapType) {
    return false;
  }

  @Override
  public <T> T unwrap(Class<T> unwrapType) {
    return null;
  }

  @Override
  public boolean supportsAggressiveRelease() {
    return true;
  }

}
```

And finally the configuration class to wire Hibernate correctly.
```language-java
@Configuration
public class HibernateConfig {

  @Autowired
  private JpaProperties jpaProperties;

  @Bean
  public JpaVendorAdapter jpaVendorAdapter() {
    return new HibernateJpaVendorAdapter();
  }

  @Bean
  public LocalContainerEntityManagerFactoryBean entityManagerFactory(DataSource dataSource,
      MultiTenantConnectionProvider multiTenantConnectionProviderImpl,
      CurrentTenantIdentifierResolver currentTenantIdentifierResolverImpl) {
    Map<String, Object> properties = new HashMap<>();
    properties.putAll(jpaProperties.getHibernateProperties(dataSource));
    properties.put(Environment.MULTI_TENANT, MultiTenancyStrategy.SCHEMA);
    properties.put(Environment.MULTI_TENANT_CONNECTION_PROVIDER, multiTenantConnectionProviderImpl);
    properties.put(Environment.MULTI_TENANT_IDENTIFIER_RESOLVER, currentTenantIdentifierResolverImpl);

    LocalContainerEntityManagerFactoryBean em = new LocalContainerEntityManagerFactoryBean();
    em.setDataSource(dataSource);
    em.setPackagesToScan("com.srai");
    em.setJpaVendorAdapter(jpaVendorAdapter());
    em.setJpaPropertyMap(properties);
    return em;
  }
}
```
Of particular note you will see the multi tenant strategy set to `SCHEMA` and our `multiTenantConnectionProviderImpl` and `currentTenantIdentifierResolverImpl` classes supplied to the configuration to satisfy that strategy's requirements.  You will also note that we are using the default hibernate `jpaProperties` that SpringBoot uses.  This is important to get things like the default naming strategy which converts snake case in database schemas to camel case in the Java entities transparently (see [here](http://stackoverflow.com/questions/25283198/spring-boot-jpa-column-name-annotation-ignored/25293929#25293929))

**And that's really all there is to it.**  When you look at the amount of code to power and how neatly abstracted it is away from your business logic it is hard to imagine a cleaner and simpler implementation for Hibernate & Spring to provide.

A full implementation of the code samples above can be found on github (https://github.com/singram/spring-boot-multitenant)

I hope you found this useful.

If you want to read further around the topic and differing approaches, the following articles may be of interest and were of great use in the development of the code and this article.

- http://anakiou.blogspot.com/2015/08/multi-tenant-application-with-spring.html
- http://fizzylogic.nl/2016/01/24/Make-your-Spring-boot-application-multi-tenant-aware-in-2-steps/
- http://www.greggbolinger.com/tenant-per-schema-with-spring-boot/
- http://jannatconsulting.com/blog/?p=41
- http://stackoverflow.com/questions/29928404/internationalization-by-subdomain-in-spring-boot
- https://dzone.com/articles/stateless-session-multi-tenant
- http://publicstaticmain.blogspot.com/2016/05/multitenancy-with-spring-boot.html
