---
layout: post
title:  "Creating a Spring MongoDBAuthenticationProvider"
date:   2019-12-22 16:00:00 +0000
categories: [mongodb,spring]
---
<style type="text/css" media="screen">
  .highlight {
    width: 900px;
  }
</style>
# Introduction

This short post describes how to implement a spring `AuthenticationProvider` that will authenticate against MongoDB directly.
Users will only be authenticated with the application if they are valid users in MongoDB.

An example use-case for this would be for support applications / tooling on top of the database.

This is an alternative to creating and managing your own `User` entities via JPA 
where user data is usually stored in an application level collection such as `users`.

# Database Setup
Make sure that you have a secure instance of MongoDB running on the default port at `localhost:27017`.<br/>
Create a user for testing via the mongo shell:
```
use admin
db.createUser(
  {
    user: "jason",
    pwd: "jason123",
    roles: [ "readWriteAnyDatabase" ]
  }
)
```

Verify access via the shell:
```
mongo  --authenticationDatabase admin -u jason -p jason123 admin --eval 'db.runCommand({listDatabases:1})'
# databases should be printed
```

# The AuthenticationProvider

Lets jump straight into the code:

```java
@Service
public class MongoDBAuthenticationProvider extends AbstractUserDetailsAuthenticationProvider {
...
public MongoDBAuthenticationProvider(
        @Value("${mongo.connection.string:mongodb://localhost:27017/admin}") String connectionString,
        @Autowired MongoDBClientFactory clientFactory) {
    this.connectionString = connectionString;
    this.clientFactory = clientFactory;
}
```
The constructor has two arguments: the mongoDB connection string and a `MongoDBClientFactory`. 
The `MongoDBClientFactory` is a simple wrapper around the static `MongoClients.create` method to make testing easier:

```java
@Configuration
public class MongoDBClientFactory {

    public MongoClient create(MongoClientSettings settings) {
        return MongoClients.create(settings);
    }

}
```

The interesting code is implemented in the `retrieveUser` method.
- `MongoClientSettings` is first created from the connectionString and credentials (username and password).
- Then a `MongoClient` is created.
- Using the client an attempt is made to list the database names and retrieve the first one. If the user is authenticated this should work.
- Otherwise a `MongoSecurityException` is thrown from the driver. This is caught, wrapped in a `BadCredentialsException` and re-thrown:

```java
@Override
protected UserDetails retrieveUser(String username,
                                   UsernamePasswordAuthenticationToken authentication)
                                   throws AuthenticationException {

    ConnectionString connString = new ConnectionString(connectionString);
    MongoCredential credential = createCredential(username, authentication, connString);

    MongoClientSettings settings = MongoClientSettings.builder()
            .applyConnectionString(connString)
            .credential(credential)
            .build();

    try(MongoClient mongoClient = clientFactory.create(settings)) {
        log.info("Attempting to authenticate user '{}'", username);
        mongoClient.listDatabaseNames().first();
        log.info("Successfully authenticated user '{}'", username);
    } catch(MongoSecurityException mse) {
        String message = String.format("User '%s' was not authenticated.", username);
        log.warn(message, mse);
        throw new BadCredentialsException(message, mse);
    }
    return new User(username, authentication.getCredentials().toString(), Collections.emptyList());
}

private MongoCredential createCredential(String username,
                                         UsernamePasswordAuthenticationToken authentication,
                                         ConnectionString connString) {
    char[] chars = authentication.getCredentials().toString().toCharArray();
    return MongoCredential.createCredential(username, connString.getDatabase(), chars);
}
```

- If the list databases command worked, a `User` is created and returned.
- This is cached by the spring framework: so this method is not called upon every request.

# Summary
This short post demonstrated how write a Spring `AuthenticationProvider` where authentication is done against users defined in MongoDB directly.
The code is available as part of a simple web-application on github here: [MongoDBAuthenticationProvider.java](https://github.com/jsonking/secure-web/blob/master/src/main/java/com/jsonking/auth/MongoDBAuthenticationProvider.java)