+++
title = "Quarkus for Spin - Part 2"
date = 2020-06-18
tags = ["Java", "Quarkus", "Picocli", "Native", "Tika", "Camel"]
categories = ["Development"] 
+++

In this post we will be enhancing from the [Part 1]({{< ref "quarkus-for-spin-1" >}}) to add new functionality using quarkus extensions. We will add a watch command to the application that watches a folder for file events and process to identify the type of file.

In order to implement the above functionality we will use the following 
- quarkus picocli to add subcommand "watch"
- quarkus camel integration to watch for file events and react on them
- quarkus tika to process a file and determine its mine type

We will be starting from the code at branch [step-01](https://github.com/rchve/qlearn/tree/step-01)

#### Adding subcommand

To add a new subcommand create a new Java class for command and add Picocli annoations that will be discovered when run as quarkus application

```java
import picocli.CommandLine;

@CommandLine.Command(
    name = "watch",
    mixinStandardHelpOptions = true,
    description = "Watches a path for file changes.")
public class WatchCommand implements Runnable {
  @CommandLine.Option(
      names = {"-p", "--path"},
      description = "Path to watch.",
      defaultValue = ".")
  String path;

  @Override
  public void run() {
    System.out.println("Watch command fired ...");
  }
}
```

The @CommandLine.Command annotation on the class defines the subcommand with name as "watch". As the command needs to accept the path to monitor this is added and an option to the command using the @CommandLoine.Option annotation. The run function is executed whenever the command is run.

##### Building and running

```sh
./mvnw clean package

java -jar target/qlearn-1.0.0-runner.jar watch           
Watch command fired ...
```

#### Adding support to listen to file events

The quarkus camel integration have [file watch component](https://camel.apache.org/camel-quarkus/latest/extensions/file-watch.html) that can be used to monitor and perform action on file events. Guide for camel quarkus integration is available [here](https://camel.apache.org/camel-quarkus/latest/). 

Quarkus looks for @ApplicationScoped RouteBuilder and will setup the route and start it. However for our case we want to setup camel route for the watch command only. This can be achieved by configuring the routes to the CamelContext provided by Quarkus.

Update to pom.xml to add the dependencyManagment and dependencies.
```xml
    <dependencyManagement>
      <dependencies>
          <dependency>
              <groupId>org.apache.camel.quarkus</groupId>
              <artifactId>camel-quarkus-bom</artifactId>
              <version>1.0.0-CR2</version>
              <type>pom</type>
              <scope>import</scope>
          </dependency>
        </dependencies>
    </dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.apache.camel.quarkus</groupId>
            <artifactId>camel-quarkus-file-watch</artifactId>
        </dependency>
    </dependencies>
```

Update the WatchCommand class to setup the camel route to monitor and listen for messages

```java
import io.quarkus.runtime.Quarkus;
import org.apache.camel.CamelContext;
import org.apache.camel.builder.endpoint.EndpointRouteBuilder;
import picocli.CommandLine;

@CommandLine.Command(
    name = "watch",
    mixinStandardHelpOptions = true,
    description = "Watches a path for file changes.")
public class WatchCommand implements Runnable {
  @CommandLine.Option(
      names = {"-p", "--path"},
      description = "Path to watch.",
      defaultValue = ".")
  String path;

  private final CamelContext ctx;
  private final FileProcessor processor;

  public WatchCommand(CamelContext ctx, FileProcessor processor) {
    this.ctx = ctx;
    this.processor = processor;
  }

  @Override
  public void run() {
    try {
      System.out.println("Configuring Watches ... ");
      ctx.addRoutes(
          new EndpointRouteBuilder() {
            @Override
            public void configure() {
              from(fileWatch(path)).process(processor);
            }
          });
      System.out.println("Waiting for file changes ... Press Ctrl+C to exit.");
      Quarkus.waitForExit();
    } catch (Exception e) {
      System.err.println("Unable to setup watch");
      System.exit(-1);
    }
  }
}
```

The above code injects the CamelContext and FileProcessor to the watch command and the setup the camel route to monitor path and process any event using the FileProcessor.

```java
import org.apache.camel.Exchange;
import org.apache.camel.Processor;
import org.apache.camel.component.file.watch.FileWatchComponent;
import org.apache.camel.component.file.watch.constants.FileEventEnum;

import javax.enterprise.context.ApplicationScoped;

@ApplicationScoped
class FileProcessor implements Processor {
  @Override
  public void process(Exchange exchange) {
    System.out.println("----------------------------------------------------");
    final String path = (String) exchange.getMessage().getHeader(Exchange.FILE_PATH);
    final FileEventEnum eventType =
        (FileEventEnum) exchange.getMessage().getHeader(FileWatchComponent.EVENT_TYPE_HEADER);
    System.out.println(String.format("File %s was %s.", path, eventType));
    System.out.println("----------------------------------------------------\n");
  }
}
```

The above file processor just logs the file and event type.

##### Building and running

```sh
./mvnw clean package

java -jar target/qlearn-1.0.0-runner.jar watch -p "./tmp"
Configuring Watches ... 
Waiting for file changes ... Press Ctrl+C to exit.
----------------------------------------------------
File /Users/rahgupta19/.k/gh/qlearn/tmp/abc.txt was CREATE.
----------------------------------------------------
```

The above command monitor the "tmp" directory and processes file events using the camel routes.

#### Identifying the file type using Apache Tika

Quarkus Tika integration uses Apache Tika library to analyse context of the documents. We will use the quarkus-tika extension to parse the file contents using the FileProcessor and identify the Media Type of the document.

Update the pom.xml to include the quarkus-tika dependency.
```xml
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-tika</artifactId>
        </dependency>
```

Update the FileProcessor to parse the files received by Apache Camel and parse files using Apache Tika.

```java
import io.quarkus.tika.TikaMetadata;
import io.quarkus.tika.TikaParser;
import org.apache.camel.Exchange;
import org.apache.camel.Processor;
import org.apache.camel.component.file.watch.FileWatchComponent;
import org.apache.camel.component.file.watch.constants.FileEventEnum;
import org.apache.tika.io.TikaInputStream;

import javax.enterprise.context.ApplicationScoped;
import java.io.File;
import java.io.IOException;

@ApplicationScoped
class FileProcessor implements Processor {
  private final TikaParser parser;

  FileProcessor(TikaParser parser) {
    this.parser = parser;
  }

  @Override
  public void process(Exchange exchange) throws IOException {
    System.out.println("----------------------------------------------------");
    final String path = (String) exchange.getMessage().getHeader(Exchange.FILE_PATH);
    final FileEventEnum eventType =
        (FileEventEnum) exchange.getMessage().getHeader(FileWatchComponent.EVENT_TYPE_HEADER);
    System.out.println(String.format("File %s was %s.", path, eventType));
    final TikaMetadata metadata = parser.getMetadata(TikaInputStream.get(new File(path).toURI()));
    System.out.println("Content Type" + metadata.getValues(Exchange.CONTENT_TYPE));
    System.out.println("----------------------------------------------------\n");
  }
}
```

##### Building and running

```sh
./mvnw clean package

java -jar target/qlearn-1.0.0-runner.jar watch -p "./tmp"
Configuring Watches ... 
Waiting for file changes ... Press Ctrl+C to exit.
----------------------------------------------------
File /Users/rahgupta19/.k/gh/qlearn/tmp/abc.txt was CREATE.
Content Type[text/plain; charset=ISO-8859-1]
----------------------------------------------------

----------------------------------------------------
File /Users/rahgupta19/.k/gh/qlearn/tmp was CREATE.
Content Type[text/plain; charset=ISO-8859-1]
----------------------------------------------------
```

##### Building and running the native

```sh
./mvnw clean package -Pnative

./target/qlearn-1.0.0-runner watch -p ./tmp
Configuring Watches ... 
Waiting for file changes ... Press Ctrl+C to exit.
----------------------------------------------------
File /Users/rahgupta19/.k/gh/qlearn/tmp/abc.txt was CREATE.
Content Type[text/plain; charset=ISO-8859-1]
----------------------------------------------------

----------------------------------------------------
File /Users/rahgupta19/.k/gh/qlearn/tmp/abc.txt was MODIFY.
Content Type[text/plain; charset=ISO-8859-1]
----------------------------------------------------

----------------------------------------------------
File /Users/rahgupta19/.k/gh/qlearn/tmp/1.png was CREATE.
Content Type[image/png]
----------------------------------------------------
```

Trying out with different files and getting same functionality as cli application.

The code for this step is available at https://github.com/rchve/qlearn/tree/step-02

##### References
- [Quarkus Picocli](https://quarkus.io/guides/picocli)
- [Quarkus Camel](https://camel.apache.org/camel-quarkus/latest/)
- [Quarkus Tika](https://quarkus.io/guides/tika)