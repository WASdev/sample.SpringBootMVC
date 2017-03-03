 # Integrating a Spring Boot MVC Project with Liberty Using Maven

This tutorial demonstrates the process of integrating a Spring Boot MVC project (which uses the [spring-boot-starter-web](https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-web) artifact, or one of its derived artifacts) with Liberty. For the purposes of this example, we'll be building off Spring Boot's "gs-serving-web-content" [sample project](https://github.com/spring-guides/gs-serving-web-content). Spring Boot also provides a [guide on their website](https://spring.io/guides/gs/serving-web-content/) which explains their project code in great detail.

We'll show the integration of this project in two ways. First, we'll package the application as a WAR and deploy it manually to an external Liberty server. Building on this, we'll then eliminate the need for an external server by packaging a Liberty server with our application, and produce a standalone runnable JAR. 

### Table of Contents

* [Getting Started](#start)
* [Packaging as a WAR](#war)
* [Creating a Standalone Runnable JAR](#jar)
* [Conclusion](#conclusion)

## <a name="start"></a>Getting Started

Start by downloading/cloning the code from Spring's "gs-serving-web-content" [sample project](https://github.com/spring-guides/gs-serving-web-content/). All the proceeding modifications will be made on the code in the ["complete" folder](https://github.com/spring-guides/gs-serving-web-content/tree/master/complete) of that project. Thus, we suggest familiarizing yourself with that code and reading the guide before proceeding.

## <a name="war"></a>Packaging as a WAR

In this section, we'll show how to package your application as a WAR file, which you can then manually deploy to an external Liberty server (running wlp-webProfile7 or something similar) via the dropins folder. 

We break down the process into the following sections. 

### Modifying the POM

Make these changes to `pom.xml`:

1. Set the packaging type to WAR. By default, no packaging type is specified in the sample project, so you'll need to add `<packaging>war</packaging>` near the top of the project, after the `version` parameter. 
2. Add a "start-class" property to your properties list, and reference (including package name) your Spring Boot startup class. For our example, this class is `hello.Application`. We'll explain the importance of this in the next section.
3. Exclude Tomcat from the `spring-boot-starter-thymeleaf` dependency. This artifact includes the `spring-boot-starter-web` dependency we talked about at the beginning of this tutorial. As specified in [Maven Central](http://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-web), `spring-boot-starter-web` uses Tomcat as the default embedded server. However, we don't need Tomcat anymore because we want to deploy our application to a Liberty server. Thus, we add the following exclusion to `spring-boot-starter-thymeleaf`:
```
<exclusions>
	<exclusion>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-tomcat</artifactId>
	</exclusion>
</exclusions>
```
4. Add the servlet dependency in order to package your application as a WAR:
```
<dependency>
	<groupId>javax.servlet</groupId>
	<artifactId>javax.servlet-api</artifactId>
    <scope>provided</scope>
</dependency>
```

### Modifying the Code

We previously briefly mentioned the importance of setting the `<start-class>` parameter in the POM properties. The importance of this is evident now, as we're about to change how our Spring MVC application is launched.

Traditionally, a Spring Boot application running on an embedded server such as Tomcat simply calls `SpringApplication.run(...)` in its main class (in this case `hello.Application`). However, we must change this when exporting our application as a WAR to deploy to a Liberty server. Specifically, we're going to have our start class extend `SpringBootServletInitializer`. Then, by setting this class as our `<start-class>` parameter in `pom.xml`, we specify that our application starts its execution from this class.

To make these changes, replace the original code in `hello/Application.java` with the following:
```
package hello;

import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.builder.SpringApplicationBuilder;
import org.springframework.boot.web.support.SpringBootServletInitializer;

@SpringBootApplication
public class Application extends SpringBootServletInitializer {

	@Override
	protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
		return application.sources(Application.class);
	}
}
```

### Build and Deploy

We've now completed all the changes necessary to allow our application to run as a WAR on an external Liberty server. To build the WAR file, run `mvn clean package`.  You should see the WAR in your "target" directory after the build completes.

We can now start up a Liberty server (I'm using Liberty version 16.0.0.4 with Java EE 7 Web Profile) and move the WAR file we just created to the "dropins" directory (or whatever appropriate directory based on your server configuration). 

## <a name="jar"></a>Creating a Standalone Runnable JAR

