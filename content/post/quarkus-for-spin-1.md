+++
title = "Quarkus for Spin - Part 1"
date = "2020-06-10"
tags = ["Java", "Quarkus", "Picocli", "Native"]
categories = ["Development"] 
+++

From some time i wanted to try out [Quarkus](https://quarkus.io/). Starting with version 1.4 it bought the functionality of command line support. It was good time to start playing with Quarkus for learning and understanding the ecosystem along with native compilation.

In this blog i will create a cli application using Quarkus picocli extension and natively compile to binary.

#### Getting Started

A started project can be created from [code.quarkus.io](https://code.quarkus.io/) or using the command below

```sh
mvn io.quarkus:quarkus-maven-plugin:1.5.0.Final:create \
      -DprojectGroupId=io.gh.rchve -DprojectArtifactId=qlearn \
      -DprojectVersion=1.0.0 -Dextensions="picocli" \
      -DclassName=io.gh.rchve.qlearn.Main 
```

Delete the generated folder (src/test, src/main/docker).

From pom.xml remove all dependencies except quarkus-picocli.

#### Creating Top level command

Create class for Top level command 

```java
package io.gh.rchve.qlearn.commands;

import io.quarkus.picocli.runtime.annotations.TopCommand;
import picocli.CommandLine;

@TopCommand
@CommandLine.Command(mixinStandardHelpOptions = true, version = "1.0.0")
public class QLearnCommand implements Runnable {
  @Override
  public void run() {}
}
```

#### Build and run the command

```sh
./mvnw clean package

java -jar target/qlearn-1.0.0-runner.jar -h

__  ____  __  _____   ___  __ ____  ______ 
 --/ __ \/ / / / _ | / _ \/ //_/ / / / __/ 
 -/ /_/ / /_/ / __ |/ , _/ ,< / /_/ /\ \   
--\___\_\____/_/ |_/_/|_/_/|_|\____/___/   
2020-06-10 18:45:17,141 INFO  [io.quarkus] (main) qlearn 1.0.0 on JVM (powered by Quarkus 1.5.0.Final) started in 0.333s. 
2020-06-10 18:45:17,162 INFO  [io.quarkus] (main) Profile prod activated. 
2020-06-10 18:45:17,162 INFO  [io.quarkus] (main) Installed features: [cdi, picocli]
Usage: <main class> [-hV]
  -h, --help      Show this help message and exit.
  -V, --version   Print version information and exit.
2020-06-10 18:45:17,275 INFO  [io.quarkus] (main) qlearn stopped in 0.004s
```

The above run displays the Quarkus banner and info logs. To disable this add the following properties to application.properties

```properties
quarkus.banner.enabled=false
quarkus.log.level=ERROR
```

Now build and run again.

```sh
./mvnw clean package

java -jar target/qlearn-1.0.0-runner.jar -h
Usage: <main class> [-hV]
  -h, --help      Show this help message and exit.
  -V, --version   Print version information and exit.
```

Output is similar to a cli application.

#### Build and run the native

Building a native image uses the [GraalVM](https://www.graalvm.org/) to convert the java application to native binary.

Running the below command to create a binary of Quarkus Application.
```sh
./mvnw clean package -Pnative

./target/qlearn-1.0.0-runner -h
Usage: <main class> [-hV]
  -h, --help      Show this help message and exit.
  -V, --version   Print version information and exit.
```

#### Running the application from IDE

The Quarkus application generates the Main class that will be used when running the application. We can provide our own Main class to be able to run from IDE.

All the below Main class

```java
package io.gh.rchve.qlearn;

import io.quarkus.runtime.Quarkus;

public class QLearn {
  public static void main(String[] args) {
    Quarkus.run(args);
  }
}
```

Now we can run the application from the IDE.

The code for this step is available at https://github.com/rchve/qlearn/tree/step-01 

#### Next Steps

Add sub commands to the application.

##### References
- [Quarkus Picocli](https://quarkus.io/guides/picocli)