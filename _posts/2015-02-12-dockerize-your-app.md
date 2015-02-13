---
layout: post
title: "dockerize your app"
date: 2015-02-12 00:30:00
summary: "This is an example how to simply dockerize your application with minimal effort."
categories: docker
tags:  docker gradle jersey java
---
I had time to play with [Docker](http://www.docker.com) so I was able to create
an example how to dockerize java application with minimal effort.

Let's dockerize JAX-RS resource served on Grizzly container. This app is simple
as it can be and it is essentially [Jersey Hello World Example]
(https://github.com/jersey/jersey/tree/master/examples/helloworld) with some
small changes.
My only contribution is ```Dockerfile``` [template]
(https://github.com/okosatka/dockerize-gradle-app/blob/master/src/main/dist/Dockerfile)
and [a couple gradle tasks](https://github.com/okosatka/dockerize-gradle-app/blob/master/build.gradle)
which make all distribution files you need to create a docker image.

## configuration

There are only few configuration options centralized into ```build.gradle```
and ```dockerize``` gradle task which replaces tokens like ```BASE_IMAGE```
in build directory.
{% highlight groovy %}
ext {
    // lib version
    jerseyVersion = "2.14"

    // dockerfile template attributes
    baseImage = "dockerfile/ubuntu"
    port = "4004"
}
{% endhighlight %}

{% highlight groovy %}
task dockerize(type: Copy, dependsOn: tasks.fatJar) {
    from 'src/main/dist/Dockerfile'
    into "${buildDir}/dockerize"
    filter(ReplaceTokens, tokens: [APP_NAME   : project.name,
                                   APP_VERSION: project.version,
                                   BASE_IMAGE : baseImage,
                                   PORT       : port])
}
{% endhighlight %}

Tokens used by a [template](https://github.com/okosatka/dockerize-gradle-app/blob/master/src/main/dist/Dockerfile)
will be replaced with values defined in `dockerize` task.

{% highlight dockerfile %}
#
# @APP_NAME@ dockerfile.
#

# Pull base image.
FROM @BASE_IMAGE@

# Install Java.
RUN \
  echo oracle-java8-installer shared/accepted-oracle-license-v1-1 select true | debconf-set-selections && \
  add-apt-repository -y ppa:webupd8team/java && \
  apt-get update && \
  apt-get install -y oracle-java8-installer && \
  rm -rf /var/lib/apt/lists/* && \
  rm -rf /var/cache/oracle-jdk8-installer

# Define commonly used JAVA_HOME variable
ENV JAVA_HOME /usr/lib/jvm/java-8-oracle

COPY ./@APP_NAME@-@APP_VERSION@.jar /@APP_NAME@/

# Define working directory.
WORKDIR /work-dir

ENTRYPOINT ["java", "-jar", "/@APP_NAME@/@APP_NAME@-@APP_VERSION@.jar"]

EXPOSE @PORT@
{% endhighlight %}


## build app

Distribution task makes all resources you need and places them into ```${buildDir}/distribution```.

 - Dockerfile - transformed template located in ```src/main/dist```
 - fat jar - contains all needed classes

```
gradle distribution
```
Distribution task depends on ```dockerize``` task and ```fatjar``` task.

## build a docker image
Directory ```${buildDir}/distribution``` contains all you need to create docker image.
Go to the distribution directory and build a docker image with tag ```jersey\hello```

```
cd build/distribution
docker build -t jersey/hello .
```

## run docker image
Argument -p means publishing exposed port to the host. We configured Grizzly be running on port 4004 and we publish it as 4001 at host.

```
docker run -p 4001:4004 -d --name hello-4001 jersey/hello
```

Then check the container is running

```
docker ps
```

Output should look like this

```
CONTAINER ID        IMAGE                 COMMAND                CREATED             STATUS              PORTS                    NAMES
9656759a854d        jersey/hello:latest   "java -jar /dockeriz   10 minutes ago      Up 8 seconds        0.0.0.0:4001->4004/tcp   hello-4001
```


## test
```
curl localhost:4001/base/helloworld
```
...and here we go

```
Hello World!
```

[Repo](http://github.com/okosatka/dockerize-gradle-app) on GitHub
