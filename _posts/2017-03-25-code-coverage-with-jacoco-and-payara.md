---
title: Code Coverage with JaCoCo and Payara
updated: 2017-03-25 18:00
---

I want to have code coverage from JaCoCo (which I can import in SonarQube) for system tests, which are running against an application deployed on Payara server. 

I start the Payara server with Jenkins by using Docker. So first of all I have to download JaCoCo to my container.

```shell
# Download JaCoCo
ENV JACOCO_URL http://search.maven.org/remotecontent?filepath=org/jacoco/jacoco/0.7.9/jacoco-0.7.9.zip
ENV JACOCO_FILENAME jacoco-0.7.9.zip
RUN mkdir /opt/JaCoCo && \
  curl -o /opt/${JACOCO_FILENAME} -L ${JACOCO_URL} && unzip /opt/${JACOCO_FILENAME} -d /opt/JaCoCo && \
  chmod -R 755 /opt/JaCoCo
```

In the next step I will configure the Java agent by adding a JVM property to Payara. The easiest way to get the result is using the TCP server option. 
I specify all addresses instead of the loopback, because I want to get the results from my Jenkins server.

```shell
...
./asadmin --user admin --passwordFile pwdfile create-jvm-options '-javaagent\:/opt/JaCoCo/lib/jacocoagent.jar=output=tcpserver,address=*' && \
```

Now the only thing left to do is to grab to results and append them to the results as a post build step by using the Maven plugin.

```shell
mvn org.jacoco:jacoco-maven-plugin:0.7.9:dump -Djacoco.address=appserver-container -Djacoco.destFile=target/jacoco-it.exec
```
