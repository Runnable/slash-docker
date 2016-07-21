---
title: Advantages of Docker for Java Apps
step: 1
category: java
tags:
- docker
- java
- advantages
excerpt: Docker for developing and deploying Java applications.
---

Docker containers make it quicker and easier to develop, test, and deploy applications within multiple environments. Docker containers provide the flexibility and the isolation you need to create development and test environments to match the many production environments your application is likely to accumulate over its lifetime.

* **Development:**
  Setting up an individual development environment is time consuming, particularly if multiple environments and toolsets are needed across development teams or geographies.

  For example, if you are developing a Java/Maven-based application *without Docker*, you must install the JDK and Maven on your host machine. But *with docker*, you simply obtain a Maven image from the Docker Hub and use it to develop, test, and run your application.

  If your task is complicated by the need for different versions of Maven and Java, you can easily get the matching Docker image for each version and develop/run in isolation.

* **Testing:**
  Test environments can be even more challenging because they will likely represent a larger number of run-time variations - for example, different frameworks and databases, different versions of Java, and so on. And this is only complicated by developing and running in virtualized environments.

  Similar to the development example, the ability to spin up many Docker containers based on specific custom images make it much easier to have many representative test environments. To test your application in different versions of Java, you can simply run your application in multiple containers. You can set up containers for your databases as well. This will dramatically speed up your testing process.

* **Deployment and Packaging:**
  Deployment with Docker is also simple and easy. For example, if your application runs in your local environment using a Docker container, then it will run on the target server as well.

  By packaging your application in a container with all of its configurations and dependencies, you ensure that it will always work locally or on another machine, or in test and production environments. No more worries about having to install the same configurations into different environments.

# Other Benefits

* Docker makes it easier to provide a consistent development environment for your whole team, particularly when your team must frequently shift from one environment to another. Your team can easily use the same OS, the same language run-time environment, and the same system libraries regardless of the host OS.

* Similarly, Docker makes it easier to provide a development environment which is the exact replica of your production environment. You can deploy the development environment that is necessary for a given production target.

* Docker makes it much easier to provide safety and/or isolation when troubleshooting your new applications - i.e., with rapid builds and incremental changes, you can reduce the number of variables that change prior to an application build.

* While using Docker, your specific code base will have everything it needs inside a custom Docker container to compile and run properly. You can use multiple versions of any language without having to resort to all of the "hackarounds" for your language (Java8, Java7, Python, Ruby, Node.js). If you want to compile your Java program with Java 1.7 instead of the 1.8 that is installed on your machine, compile it in a Java 1.7 Docker container. Similarly, if you want to run your Java application, run it inside the appropriate Java Docker container.

* While using Docker for development, you can still use your favorite editor/IDE tools.

# Examples
Provided below are several Java-based examples that illustrate the concepts discussed above. You can extrapolate these ideas to other development or run-time environments.

## Run Simple Java Application using Docker

Create a project folder and create a file inside this folder called *Main.java* with the following content.

```java
public class Main
{
    public static void main(String[] args) {
        System.out.println("Hello, World");
    }
}
```

Now execute the following commands from the current project directory:

* Execute this command to compile your *Main.java* file.

```bash
$ docker run --rm -v $PWD:/app -w /app java:8 javac Main.java
```

* Execute this command to run your compiled *Main.class* file.

```bash
$ docker run --rm -v $PWD:/app -w /app java:8 java Main
```

The following output should be displayed:

```bash
Hello, World
```

In this example, you compiled and executed a Java program without the need to install Java on your machine by using the official Java image from the Docker Hub repository. Now, if you want to instead use Java 7 for your application, all you need to do is use the Java:7 image.

### Running application using Java7

```bash
$ docker run --rm -v $PWD:/app -w /app java:7 javac Main.java
$ docker run --rm -v $PWD:/app -w /app java:7 java Main
```

## Run a Maven-based Java Application Using Docker

If you already have a Maven project, you can apply the concepts below. Otherwise, begin by creating a Maven project using the command below.
**Note:** Remember, you do not need to install Maven or JDK.

```bash
$ docker run -it --rm -v "$PWD":/app -w /app maven:3.3-jdk-8 mvn archetype:generate -DgroupId=com.mycompany.app -DartifactId=my-app -DarchetypeArtifactId=maven-archetype-quickstart -Dinte
```

This will create a Maven project in the current directory under the 'my-app' directory.

* Go to the project directory.

```bash
$ cd  my-app
```

* Build the project.

```bash
$ docker run -it --rm -v "$PWD":/app -w /app maven:3.3-jdk-8 mvn package
```

* Test the newly compiled and packaged JAR with the following command:

```bash
$ docker run -it --rm -v "$PWD":/app -w /app maven:3.3-jdk-8 java -cp target/my-app-1.0-SNAPSHOT.jar com.mycompany.app.App
```

* The following output will be displayed:

```bash
Hello World!
```

The above process uses the official Maven image from the Docker Hub repository. This Maven image uses the tag “maven:3.3-jdk-8” to indicate that it is comprised of Maven 3.3 and Java 8.

If you wanted to use Maven 3.2 (instead of 3.3), simply change the tag of the image to 3.2.

You can see the list of all available tags from the Docker Hub repository at [https://hub.docker.com/r/library/maven/](https://hub.docker.com/r/library/maven/)



## Run Gradle-Based Java Application Using Docker

If you already have a Gradle project, you can apply the concepts below. Otherwise, download the basic Gradle project from here: [https://spring.io/guides/gs/gradle/](https://spring.io/guides/gs/gradle/)

* If you downloaded the project from above URL, go to the 'complete' directory inside the project.

```bash
$ cd  gs-gradle/complete
```

* Build the project.

```bash
$ docker run -it --rm -v "$PWD":/app -w /app stakater/gradle gradle build
```

* Run the project.

```bash
$ docker run -it --rm -v "$PWD":/app -w /app stakater/gradle gradle run
```

# Conclusion

You can compile and run three different types of Java applications without the need to install special tools or environment components. Any Java application can run with Docker without installing JDK, Maven, or Gradle, and without worrying about version conflicts or other complications associated with a non-containerized development, run-time, or test environment.