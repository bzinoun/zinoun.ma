---
title: "[ 🇬🇧 ] IBM AppId integration with Springboot 2"
date: 2019-12-01
publishdate: 2019-12-01
dr  aft: false
---

####  What we gonna do 

through this article we will focus on how to integrate the OpenId Connect Server, AppID, proposed by IBM Cloud with a Java application under Springboot.

#### Prerequisites

- An IBM Cloud Account: The free version is sufficient
- Knowledge about OAuth2 flow
- Knowledge knowledge of the Java ecosystem - Springboot.

####  IBM AppId in few lines 
 App ID is the OpenId Connect service from IbmCloud . It helps developers to easily add authentication to their web or mobile apps with few lines of code, and it helps them secure their cloud-native applications and services on IBM Cloud. AppID propose a connection trought Facebook Google etc ... 

App ID also helps manage user-specific data that developers can use to build personalized app experiences.

> /!\  We assume that you know how to create a basic Java Spring Boot App 

_Okay , let's show some code !_

####  Configure AppId with Spring Boot
- Add the dependencies to pom.xml
```
       <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <!-- spring-security-oauth2-autoconfigure tells the framework to use OAuth2, instead of basic http security -->
        <dependency>
            <groupId>org.springframework.security.oauth.boot</groupId>
            <artifactId>spring-security-oauth2-autoconfigure</artifactId>
            <version>2.0.0.RELEASE</version>
        </dependency>
```

- Create a ResourceSecurityConfiguration Java class to enable OAuth2, and Rest capabilities. Also, extend from ResourceServerConfigurerAdapter class, to later configure security access.

```
@Configuration
@RestController
@EnableResourceServer
public class ResourceSecurityConfiguration extends ResourceServerConfigurerAdapter {

}
```

- Override security configuration by adding a configure method.

```
@Configuration
@RestController
@EnableResourceServer
public class ResourceSecurityConfiguration extends ResourceServerConfigurerAdapter {
    @Override
    public void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
        .antMatchers("/api**", "/user", "/userInfo").authenticated()
        .and().logout().logoutSuccessUrl("/").permitAll()
        .and().csrf().disable();
    }
}
```

- Add two endpoints to the ResourceSecurityConfiguration Java class that will be accessible when you perform GET Calls from the front end. The /user endpoint gives us the logged-in user object (principal) and the /userInfo endpoint returns a string with the details.

```
@Configuration
@RestController
@EnableResourceServer
public class ResourceSecurityConfiguration extends ResourceServerConfigurerAdapter {
    @Override
    public void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
        .antMatchers("/api**", "/user", "/userInfo").authenticated()
        .and().logout().logoutSuccessUrl("/").permitAll()
        .and().csrf().disable();
    }
	// for open id
	@RequestMapping("/user")
    public Principal user(Principal principal) {
        //Principal holds the logged in user information.
        // Spring automatically populates this principal object after login.
        return principal;
    }

	// for open id
    @RequestMapping("/userInfo")
    public String userInfo(Principal principal){
        final OAuth2Authentication oAuth2Authentication = (OAuth2Authentication) principal;
        final Authentication authentication = oAuth2Authentication.getUserAuthentication();
        //Manually getting the details from the authentication, and returning them as String.
        return authentication.getDetails().toString();
    }

}
```

- Add necessary AppId config values from information that you'll get from your service credentials / IBM Cloud account to application.properties to complete your configuration
you can find these information in:
Ibm cloud account -> services -> AppId -> Applications -> Add Application (if not done before) -> View Credentials

```
spring.main.allow-bean-definition-overriding=true

env.urlAppid= // OAuth server URL from IBM AppID account

# SECURITY OAUTH2 CLIENT (OAuth2ClientProperties)

security.oauth2.client.accessTokenUri=${env.urlAppid}/token
security.oauth2.client.userAuthorizationUri=${env.urlAppid}/authorization


# SECURITY OAUTH2 RESOURCES (ResourceServerProperties)
security.oauth2.resource.userInfoUri=${env.urlAppid}/userinfo


## Setting to change in case of changing app id dev instance
security.oauth2.client.client-id= // IBM AppId account client-id
security.oauth2.client.client-secret= // IBM AppId secret
```

6. Finally to Sign In and Signup you to make an HTTP Request to these endpoints:
- Sign In: `https://appid-oauth.eu-gb.bluemix.net/oauth/v3/{TenantId}/token`

``` js
Request: {
username: '',
password: '',
grant_type: 'password' // don't change
}
RequestHeader:{
Authorization: 'Basic ClientId:Secret'
}

Response:{
access_token:'',
}
```

- Signup: `https://appid-management.eu-gb.bluemix.net/management/v4/{TenantId}/cloud_directory/sign_up`

``` js
Request:{
name: '',
username: '',
password: '',
active: true, // don't change
emails: [
{
value: 'email@example.com',
primary: true
}
]
}

RequestHeader:{
   Authorization: 'Bearer {IAM_TOKEN}'
}

Response:{
HttpStatus: 200
}
```

That's all folks !