---
title: "Integration test with Docker"
category: test
tags:
 - test
 - integration
 - MySQL
 - docker
 - dao
---

## How should I test a Data Access Object (DAO)? ##

The answer to this question has always been a big doubt for me in the past. I'm sure everybody sooner or later reached the point where the need to test some code interacting with the DB arises. Unit testing code that talks with other code is quite easy (depending on how well you architect your application) but when you reach the borders, testing the interaction with other systems gets more complex. For instance let's say you are writing Java code using JDBC, what should you test about a method executing a query on the DB and returning the `ResultSet`? Should you test the `executeQuery` method has been called with the right query? Should you mock the `Statement` object to return the `ResultSet` you expect? All these approaches looks silly to me, both of them are not actually testing the query you're using works, they just make sure you wrote the query as _you suppose_ works and the DB is returning what _you suppose_ it will return. If you're thinking

> "That's very
true but in my method I have a more complex logic to test"

then maybe you are missing an abstraction layer in your code.   
The way to go in my opinion is to test your code against a real instance to make sure that your code does what it is supposed to do.


### Sub-optimal solutions ###

If you search online, you can see a lot of resources telling you to use H2. What I don't like about this approach is that if I want to test apples I don't want to test oranges, I know it's a good approximation but I want to test apples! Writing code is similar to [Rock Balancing](https://en.wikipedia.org/wiki/Rock_balancing), it's an art, get something wrong and everything can collapse.   
Someone, instead likes to write stubs and recreate the behaviour of the external system you are testing against to. This is something that doesn't make sense in my opinion apart for really sporadic cases. You should focus more on implementing something new rather than implementing something that already exists.

## Solution with Docker ##

As the use of containers became more and more invasive I said:

>"Why not using Docker for integration tests?"

It provides everything we need:

- __Repeatable tests__ For each and every build we have a dedicated empty MySQL instance
- __Real environment__ We test against the same version we have in production
- __No dedicated hardware__ The docker image runs on the same machine you execute the tests 
- __Easy integration__ Interacting with docker is quite easy and you need only few lines of code when you understand how it works.

The amazing part of this approach is that it's not limited to MySQL but it can be used to test against any external system you have a docker image of, the configuration is pretty similar.   
Compared to H2 approach, using MySQL directly gives you also the advantage of being able to test SQL procedures/triggers you have in your DB (even though I'm not a big fan of them).

## How to implement it ##

The fastest ways to integrate with docker I found online were the following:

- __Via Maven__ This [plugin](https://github.com/rhuss/docker-maven-plugin) provides features to spin up and shut down a docker image during the building phase.
- __Programmatically__ This [library](https://github.com/shazam/tocker) allows you to interact with docker daemon from your testing code. 

The former has the advantage of keeping all the configuration inside the `pom.xml` instead of having it spreaded across all your test code, on the other side though, it has a coarser grain of control: you can have either all the images running or no one running. Another advantage of using maven plugin is that you can use different profiles for different operating systems and easily configure your code accordingly.

> In OSX, docker daemon starts as a VirtualBox VM where a linux instance is hosted. This will be the place where your images will run, only on linux boxes your container will actually share the host-box resources. You need to consider this when you want to access a system inside an image via network protocol as we will see later.

In my case I'm developing on a OSX machine and the build runs on a linux-box with jenkins installed.   
Now we have all the pipes and we can start looking into how to assemble them to produce some value. You can find the whole code [here](https://github.com/VisualDNA/DockerIT). Needless to say you need [Docker](https://www.docker.com/) installed on both your development and your CI machine ;-).

### The POM ###

Let's start embedding the plugin inside the `pom.xml`. This is how it looks like

{% highlight xml %}

<plugin>
  <groupId>org.jolokia</groupId>
  <artifactId>docker-maven-plugin</artifactId>
  <version>0.13.6</version>
  <configuration>
   	<useColor>true</useColor>
	<verbose>true</verbose>
	<removeVolumes>true</removeVolumes>
	<images>
	  <image>
	    <name>mysql:5.7.9</name>
	    <run>
		  <env>
		  <MYSQL_ROOT_PASSWORD>
		    ${test.mysql.root.password}
		  </MYSQL_ROOT_PASSWORD>
		  </env>
		  <ports>
		    <port>${test.mysql.port}:3306</port>
		  </ports>
		  <wait>
		    <exec>
		      <postStart>
			    sh /tmp/import/create-database.sh /tmp/import/input.sql
			  </postStart>
		    </exec>
	      </wait>
		  <volumes>
		    <bind>
		      <volume>
			    ${project.build.testOutputDirectory}/db-scripts:/tmp/import:ro
			  </volume>
		    </bind>
		  </volumes>
	    </run>
	  </image>
    </images>
  </configuration>
  <executions>
	<execution>
	  <id>start</id>
	  <phase>pre-integration-test</phase>
	  <goals>
	    <goal>start</goal>
	  </goals>
	</execution>
	<execution>
	  <id>stop</id>
	  <phase>post-integration-test</phase>
	  <goals>
	    <goal>stop</goal>
	  </goals>
	</execution>
  </executions>
</plugin>

{% endhighlight %}

This is a typical structure for a maven plugin and we start analyizing all the configurations.   
The first two tags in the `<configuration>` are not mandatory for the result, they just make the logging inside maven more useful and prettier.
