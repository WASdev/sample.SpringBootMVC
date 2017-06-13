# Integrating a Spring Boot MVC Project with Liberty Using Maven [![Build Status](https://travis-ci.org/WASdev/sample.SpringBootMVC.svg?branch=master)](https://travis-ci.org/WASdev/sample.SpringBootMVC)

This tutorial demonstrates the process of integrating a Spring Boot MVC project (which uses the [spring-boot-starter-web](https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-web) artifact or one of its derived artifacts) with Liberty. For the purposes of this example, we'll be building off Spring Boot's `gs-serving-web-content` [sample project](https://github.com/spring-guides/gs-serving-web-content). The Spring Boot website also provides a [guide](https://spring.io/guides/gs/serving-web-content/) which explains their project code in great detail. 

### Table of Contents

* [Getting Started](#start)
* [Modifying the POM](#pom)
* [Server Configuration](#server)
* [Changing the Application Startup Process](#app)
* [Wrapping Up](#conclusion)

## <a name="start"></a>Getting Started

Start by downloading or cloning the code from Spring's `gs-serving-web-content` [sample project](https://github.com/spring-guides/gs-serving-web-content/). All the following modifications will be made on the code in the ["complete" folder](https://github.com/spring-guides/gs-serving-web-content/tree/master/complete) of that project. Thus, we suggest familiarizing yourself with that code and reading the guide before proceeding.

Below, we'll make modifications to our project to deploy it on a Liberty server. Similar to Spring Boot, the application and its containing server are then packaged into a standalone runnable JAR.

## <a name="pom"></a>Modifying the POM

### Change the Packaging Type

To get started, we set our packaging type to `liberty-assembly`. By default, no packaging type is specified in the sample project, so you'll need to add 

```
<packaging>liberty-assembly</packaging>
````
near the top of the project POM, after the `version` parameter. 

### Add POM Properties

Next, add the following properties:

```
<properties>
   ...
    <start-class>hello.Application</start-class>
    <!-- Liberty server properties -->
    <wlpServerName>MVCServer</wlpServerName>
    <testServerHttpPort>9080</testServerHttpPort>
    <testServerHttpsPort>9443</testServerHttpsPort>
    ...
</properties>
```

Note the inclusion of the `start-class` property, which references the fully qualified Spring Boot startup class. For our example, this class is `hello.Application`. This property indicates the main class to use for the runnable JAR.

### Add and Modify Dependencies

The `spring-boot-starter-thymeleaf` artifact includes the `spring-boot-starter-web` dependency we talked about at the beginning of this tutorial. As specified in [Maven Central](http://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-web), `spring-boot-starter-web` uses Tomcat as the default embedded server. However, we don't need Tomcat anymore because we want to deploy our application to a Liberty server. Thus, we add the following exclusion to the `spring-boot-starter-thymeleaf` dependency:
```
<exclusions>
	<exclusion>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-tomcat</artifactId>
	</exclusion>
</exclusions>
```

We'll also need to add two dependencies for our Liberty server:

```
<dependency>
	<groupId>com.ibm.websphere.appserver.runtime</groupId>
	<artifactId>wlp-webProfile7</artifactId>
	<version>16.0.0.4</version>
	<type>zip</type>
</dependency>

<dependency>
	<groupId>net.wasdev.maven.tools.targets</groupId>
	<artifactId>liberty-target</artifactId>
	<version>16.0.0.4</version>
	<type>pom</type>
	<scope>provided</scope>
</dependency>
```

### Add the `liberty-maven-plugin`

Finally, add and configure the `liberty-maven-plugin` as shown below. We can also remove the `spring-boot-maven-plugin` that was included in the POM of the sample project. 

```
<plugin>
	<groupId>net.wasdev.wlp.maven.plugins</groupId>
	<artifactId>liberty-maven-plugin</artifactId>
	<version>2.0</version>
	<extensions>true</extensions>
	<!-- Specify configuration, executions for liberty-maven-plugin -->
	<configuration>
		<serverName>MVCServer</serverName>
		<assemblyArtifact>
			<groupId>com.ibm.websphere.appserver.runtime</groupId>
			<artifactId>wlp-webProfile7</artifactId>
			<version>16.0.0.4</version>
			<type>zip</type>
		</assemblyArtifact>
		<assemblyInstallDirectory>${project.build.directory}</assemblyInstallDirectory>
		<!-- <configFile>src/main/liberty/config/server.xml</configFile> -->
		<packageFile>${project.build.directory}/MVCServerPackage.jar</packageFile>
		<bootstrapProperties>
			<default.http.port>9080</default.http.port>
			<default.https.port>9443</default.https.port>
		</bootstrapProperties>
		<features>
			<acceptLicense>true</acceptLicense>
		</features>
		<include>runnable</include>
		<installAppPackages>all</installAppPackages>
		<appsDirectory>apps</appsDirectory>
		<stripVersion>true</stripVersion>
		<looseApplication>true</looseApplication>
	</configuration>
</plugin>
```

In this plugin, we specify a Liberty `wlp-webProfile7` server called "MVCServer" which our application will be deployed to. We also set `<include>runnable</include>`, to indicate that the application will be packaged into a runnable JAR, which we set to be called `MVCServerPackage.jar` in the `<packageFile>` parameter.

The 2.0 version of the `liberty-maven-plugin` introduces support for a "[loose application](https://www.ibm.com/support/knowledgecenter/en/SSEQTP_liberty/com.ibm.websphere.wlp.zseries.doc/ae/rwlp_loose_applications.html) configuration", as indicated by the `looseApplication` tag. Application code is installed as a loose application XML file if `installAppPackages` is set to `all` or `project` and `looseApplication` is set to `true`. 

The loose application configuration uses XML to loosely bundle application resources, so that they can be modified individually without needing to repackage the entire application. This is useful because it allows for incremental changes to be made to our application, which saves time for tasks such as debugging.

Further documentation of the plugin configuration is provided in the [ci.maven](https://github.com/WASdev/ci.maven/tree/tools-integration) repository.

## <a name="server"></a>Server Configuration

Next, we'll provide the configuration for our Liberty server. Create a file called `server.xml` and place it in the directory specified in the `configFile` configuration parameter in the `liberty-maven-plugin`. If no directory is specified, the default directory is `src/test/resources`. 

Add the following code to `server.xml`:

```
<?xml version="1.0" encoding="UTF-8"?>
<server description="new server">
	<application context-root="/"
		location="gs-serving-web-content.war"></application>

	<!-- Enable features -->
	<featureManager>
		<feature>servlet-3.1</feature>
	</featureManager>

	<!-- To access this server from a remote client add a host attribute to 
		the following element, e.g. host="*" -->
	<httpEndpoint id="defaultHttpEndpoint" httpPort="9080"
		httpsPort="9443" />

	<!-- Automatically expand WAR files and EAR files -->
	<applicationManager autoExpand="true" />
    
    <!-- Automatically load the Spring application endpoint once the server 
		is ready. -->
	<webContainer deferServletLoad="false"/>

</server>
```

In the server configuration, we map our application context root to the server root directory. We also add the Liberty `servlet-3.1` feature needed for running our application, as well as other things which are described in the XML comments. 

## <a name="app"></a>Changing the Application Startup Process

We'll now need to change the startup process for our application. Traditionally, a Spring Boot application running on an embedded server such as Tomcat simply calls `SpringApplication.run(...)` in its main class (in this case `hello.Application`). However, our class needs to extend `SpringBootServletInitializer` instead when being deployed to a Liberty server, as shown below:

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

Also recall that we set the `<start-class>` parameter in the POM properties above to `hello.Application`. This tells the server that our application starts its execution from this class.

## <a name="conclusion"></a>Wrapping Up

Congratulations! You should now be able to run your Spring MVC application on Liberty by executing `mvn install liberty:run-server`. By navigating to `http://localhost:9080/`, you should see the application appear and behave in the same way as it does standalone. Additionally, you'll see that a JAR named `MVCServerPackage.jar` was created in the target directory, which bundles your application with the Liberty server you configured into a standalone runnable JAR.

Please feel free to download our sample code from this repository, or create a Github Issue if you have any additional questions.
