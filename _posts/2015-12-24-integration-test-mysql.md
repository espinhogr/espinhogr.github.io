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

The answer to this question has always been a big doubt for me in the past. I'm sure everybody sooner or later reached the point where the need to test some code interacting with the DB arose. Unit testing code that talks with other code is quite easy (depending on how well you architect your application) but when you reach the borders, testing the interaction with other systems gets more complex. For instance let's say you are writing Java code using JDBC, what should you test about a method executing a query on the DB and returning the `ResultSet`? Should you test whether the `executeQuery` method has been called with the right query? Should you mock the `Statement` object to return the `ResultSet` you expect? All these approaches looks silly to me, both of them are not actually testing the query you're using works, they just make sure you wrote the query as _you suppose_ works and the DB is returning what _you suppose_ it will return. If you're thinking

> "That's very true but in my method I have a more complex logic to test"

then maybe you are missing an abstraction layer in your code.   
The way to go in my opinion is to test your code against a real instance to make sure that your code does what it is supposed to do.


### Sub-optimal solutions ###

If you search around, you definitely run into a lot of resources telling you to use H2. What I don't like about this approach is that when I want to test apples I don't want to test oranges, I know it's a good approximation but I want to test apples! Writing code is similar to [Rock Balancing](https://en.wikipedia.org/wiki/Rock_balancing), it's an art, get something wrong and everything can collapse.   
Someone - instead - likes to write stubs and recreate the behavior of the external system you are testing against to. This is something that doesn't make sense in my opinion apart for really sporadic cases. You should focus more on implementing something new rather than implementing something that already exists.

### Solution with Docker ###

As the use of containers became more and more invasive I thought:

>"Why not use [Docker](https://www.docker.com/) for integration tests?"

It provides everything you need:

- __Repeatable tests__ For each and every build we have a dedicated empty MySQL instance
- __Real environment__ We test against the same version we have in production
- __No dedicated hardware__ The docker image runs on the same machine you execute the tests 
- __Easy integration__ Interacting with docker is quite easy and you need only few lines of code when you understand how it works.

The amazing part of this approach is that it's not limited to MySQL but it can be used to test against any external system you have a docker image of, the configuration is pretty similar.   
Compared to H2 approach, using MySQL directly also gives you the advantage of being able to test SQL procedures/triggers you have in your DB (even though I'm not a big fan of them).

## How to implement it ##

The fastest ways to integrate with docker I found online were the following when I wrote this article:

- __Via Maven__ This [plugin](https://github.com/rhuss/docker-maven-plugin) provides features to spin up and shut down a docker image during the building phase.
- __Programmatically__ This [library](https://github.com/shazam/tocker) allows you to interact with docker daemon from your testing code. 

The former has the advantage of keeping all the configuration inside the `pom.xml` instead of having it spread across all your test code, on the other side though, it has a coarser grain of control: you can have either all the images running or no one running. Another advantage of using maven plugin is that you can use different profiles for different operating systems and easily configure your code accordingly.

> In OSX, docker daemon starts as a VirtualBox VM where a linux instance is hosted. This will be the place where your images will run, only on linux boxes your container will actually share the host-box resources. You need to consider this when you want to access a system inside an image via network protocol as we will see later. In my case I'm developing on a OSX machine and the build runs on a linux-box with jenkins installed.

Now we have all the pipes and we can start looking into how to assemble them to produce some value. You can find the whole code [here](https://github.com/VisualDNA/DockerIT). Needless to say you need [Docker](https://www.docker.com/) installed on both your development and your CI machine ;-).

### Starting a Docker Image ###

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
		  <volumes>
		    <bind>
		      <volume>
			    ${project.build.testOutputDirectory}/db-scripts:/tmp/import:ro
			  </volume>
		    </bind>
		  </volumes>
		  <wait>
		    <exec>
		      <postStart>
			    sh /tmp/import/create-database.sh /tmp/import/input.sql
			  </postStart>
		    </exec>
	      </wait>
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

This is a typical structure for a maven plugin and we analyze now all the configurations.

- The first two tags in the `<configuration>` are not mandatory for the result, they just make the logging inside `maven` more useful and prettier.
- After those tags you start listing all the images you want to spin up, in our case we have only one image and it's `mysql` version `5.7.9`. In `<env>` you can find all the variable that will be passed to the image, in our case we are passing here the root password for mysql service as suggested on [DockerHub](https://hub.docker.com/_/mysql/). The reason why we are passing it as a maven property will be more clear later on.
- Under `<ports>` we specify the mapping we want for the MySQL internal service to the external port of our container, in this case we want to map the internal `3306` to the external port specified in the properties. This allows us to connect to the service from outside the container.
- Proceeding, we find the `<volumes>` tag, this allows us to make some file accessible from within the container. Here we let the container access our directory `/db-scripts` from the directory `/tmp/import` inside the container but we give it only read permission (`ro`). With this trick we can pass any script inside the container and we will use it to initialize our database. Something really important to remember is to set `removeVolumes` to `true`, this will delete the volume after every build and it won't clutter your disk.
- As we said before, we need to initialize the database, this is what `<postStart>` command does. It runs a script that we pass to the container via the volume and uses a second `SQL` file as argument.
- The last bit of configuration for the plugin is deciding when it starts and stop the images, we want to use them for integration test therefore we want to start them in the `pre-integration-test` phase and stop them in the `post-integration-test` phase.

Having done this, the configuration for the process is ready, as you can see it's really simple but our `pom.xml` is not ready yet, we still miss some details.

### Sharing the DB configuration ###

In the previous part you saw we used some maven properties to configure our image, we exactly used these:

{% highlight xml %}

<test.mysql.port>3306</test.mysql.port>
<test.mysql.root.password>admin</test.mysql.root.password>
<test.mysql.root.username>root</test.mysql.root.username>

{% endhighlight %}

Their meaning is obvious but the reason why we are using them might not be. We use a `maven-resources-plugin` to replace their occurrences in other files in order to make them available for our code  and for our scripts to init the database.


### Init script ###

When you run a MySQL container you have two main problems: understanding when the service is ready and creating the structure of your database. This task is achieved by the scripts we share with the container in the volume mounting.

This is the `SQL` code we want to run in our MySQL instance to initialize it, it creates a database and a table with few attributes for testing purpose. In our codebase it's in the file `input.sql`.

{% highlight sql %}

CREATE DATABASE testDb;
USE testDb;
CREATE TABLE `sample_entity` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `firstAttribute` text NOT NULL,
  `secondAttribute` text NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1;

{% endhighlight %}

Having the `SQL` we want to run, here is how we're going to run it:

{% highlight bash %}

#!/bin/bash

while ! echo $(mysql -h 127.0.0.1 -P 3306 -u${test.mysql.root.username} \
	                 -p${test.mysql.root.password} -e "SELECT 'ready'") \
                     | grep --quiet "ready"; do
    sleep 2
    echo "====================================="
    echo "Waiting for mysql Server to be ready"
    echo "====================================="

done

mysql -h 127.0.0.1 -P 3306 -u${test.mysql.root.username} \
      -p${test.mysql.root.password} < $1

{% endhighlight %}

Being aware that this script (`create-database.sh`) will be executed from within the container, the first part of the script sends repeatedly to the MySQL server (`localhost`) the query `SELECT 'ready'` with 2 second interval until the server answers. In the meanwhile it prints the message to show it's not stuck. As soon as the server answers and we grep successfully the word "`ready`" we are good to go and we can send the content of our `input.sql` to the server.   
After this the server is ready to be accessed and tested against. Two tasks to go and we are ready to use it.

### Configuration for your code ###

This part really depends on how you're used to pass configurations to your code, for this example we decided to `KISS` hence we just created a `config.properties` to give you an idea. Here is the content of the file:

{% highlight bash %}

db.host = ${test.mysql.host}
db.username = ${test.mysql.root.username}
db.password = ${test.mysql.root.password}
db.port = ${test.mysql.port}
db.database = testDb

{% endhighlight %}

This is the second place where the `maven-resources-plugin` is useful, it replaces this placeholders with whatever we specified in the `pom.xml`, obviously the database name can be parametric as well. We will read this configuration from our test code and we will initialize our driver accordingly.

### Profiles for different OS ###

At the beginning of the article we pointed out that Docker is installed differently on different OSs, this force us to have different configurations. The attentive reader noticed that we didn't specify the `test.mysql.host` variable in the `pom.xml`, we didn't do that on purpose because that's OS dependent. Here is how to define the different profiles:

{% highlight xml %}

<profiles>
  <profile>
    <id>mac</id>
    <build>
	  <plugins>
        <plugin>
	      <groupId>org.codehaus.mojo</groupId>
		  <artifactId>build-helper-maven-plugin</artifactId>
		  <version>1.10</version>
		  <executions>
		    <execution>
              <id>regex-property</id>
              <goals>
                <goal>regex-property</goal>
              </goals>
		      <configuration>
                <name>my.docker.host</name>
		 	    <value>${env.DOCKER_HOST}</value>
			    <regex>\w+:\/\/(.*):\d+</regex>
			    <replacement>$1</replacement>
		      </configuration>
		    </execution>
	      </executions>
        </plugin>
      </plugins>
    </build>
    <activation>
      <activeByDefault>true</activeByDefault>
    </activation>
    <properties>
      <test.mysql.host>${my.docker.host}</test.mysql.host>
    </properties>
  </profile>
  <profile>
    <id>linux</id>
    <properties>
      <test.mysql.host>localhost</test.mysql.host>
    </properties>
  </profile>
</profiles>

{% endhighlight %}

For the linux one the profile is quite easy, you just specify that the DB host is always localhost because the container runs directly on the host machine and this is the default configuration.   
For OSX the profile is more complex given the container runs in the VM. We used `build-helper-maven-plugin` to intercept the environment variable `DOCKER_HOST` and we used a regular expression to extract the host and map it to the `my.docker.host` variable. This variable will be then assigned to the maven property and will be used around in the code.   
Now the profiles are ready, you just have to run `mvn -Plinux verify` or `mvn -Pmac verify` and your test will execute against a real MySQL instance.

### Don't forget ###

First thing you don't have to forget is to add `maven-failsafe-plugin` to your `pom.xml` or your integration tests won't run.   
If you are a OSX user you also don't have to forget to:

- Start the VM for docker.
- Make sure that you exported environment variables for the VM in the console where you're going to run maven.

## Improvements ##

Often, if the integration tests are a lot, you want to speed up the process, I had an idea about that but I didn't have the time to test it yet:

- Make the MySQL image run in-memory. This is quite easy to achieve, an trivial way could be to declare your table as `MEMORY` as described [here](https://dev.mysql.com/doc/refman/5.7/en/memory-storage-engine.html) but this has some limitations hence my aim is to create a new MySQL image writing files on a [RAM disk](https://en.wikipedia.org/wiki/RAM_drive) instead of on the HardDisk. 

This example was really simple, in production systems you may consider something like DbUnit to help managing the database structure and the data, maven properties are your friends whichever framework you choose.

## Conclusions ##

We already started applying this approach in production and it works really well, at the moment we only tested with MySQL but I want to test it also with a more complex system. This doesn't want to be the solution to all your problems but it can be an idea you consider when you test against an external system.   
Thanks to Paolo Rascunà for helping out with this idea, even though it's a small thing we faced a lot of obstacles on the way, working in two people is always faster.
