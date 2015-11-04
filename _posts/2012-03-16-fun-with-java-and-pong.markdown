---
title: Fun with Java and Pong
---
The last little while I've been working on a side project to develop my own implementation of Pong. I just finished recently and thought I'd blog about some of my experiences while developing it. If you're interested in seeing source code, it's located [here]("https://github.com/jeffcharles/pong" "Pong source code") under a BSD license.

<a href="/images/pong_screenshot_smaller.png"><img src="/images/pong_screenshot_smaller.png" alt="Pong gameplay" title="Pong gameplay" width="486" height="385"></a>

## Motivations and Goals

* Reacquaint myself with Java and the Java ecosystem
* Do some multi-threaded programming
* Improve my design skills by trying to design a game

## Maven

I decided very early on to use Maven as my project's build tool. I wanted a flexible tool and also not to tie myself or anyone else who is interested in this project to a particular IDE. It's also very helpful to have a build tool that can fetch external dependencies as opposed to distributing them inside the project or requiring other developers to have to download and install them in particular places.

Maven simplified my project and the build process by enforcing a convention and by allowing me to use plugins. The two plugins I made use of, other than the ones that ship with Maven, were Failsafe which allowed me to distinguish between unit and integration tests, and Cobertura which generated automated test coverage metrics.

Performing operations with Maven is pretty simple, you just open a terminal to wherever your pom.xml file is located and run mvn with various arguments. For example, mvn compile will compile your code, mvn test will run your unit tests, mvn verify will run your integration tests, and mvn cobertura:cobertura will generate code coverage metrics. If a command has any dependencies, it will run those first. It will also automatically download and install any external dependencies in your pom file.

Just to provide an idea of how simple configuring Maven can be, here's my pom.xml file:

{% highlight xml %}
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.beyondtechnicallycorrect.pong</groupId>
  <artifactId>pong</artifactId>
  <version>1.0-SNAPSHOT</version>
  <packaging>jar</packaging>
  <name>pong</name>
  <url>http://www.beyondtechnicallycorrect.com</url>
  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>
  <dependencies>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>4.10</version>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.mockito</groupId>
      <artifactId>mockito-all</artifactId>
      <version>1.9.0</version>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>com.google.inject</groupId>
      <artifactId>guice</artifactId>
      <version>3.0</version>
      <scope>compile</scope>
    </dependency>
  </dependencies>
  <build>
  <plugins>
    <plugin>
        <artifactId>maven-compiler-plugin</artifactId>
        <configuration>
          <source>1.6</source>
          <target>1.6</target>
        </configuration>
      </plugin>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-failsafe-plugin</artifactId>
        <version>2.11</version>
        <executions>
          <execution>
            <goals>
              <goal>integration-test</goal>
              <goal>verify</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
  <reporting>
    <plugins>
      <plugin>
        <groupId>org.codehaus.mojo</groupId>
        <artifactId>cobertura-maven-plugin</artifactId>
        <version>2.5.1</version>
        <configuration>
          <instrumentation>
            <excludes>
              <exclude>com/beyondtechnicallycorrect/pong/**/*Module.class</exclude>
              <exclude>com/beyondtechnicallycorrect/pong/**/*FactoryImpl.class</exclude>
              <exclude>com/beyondtechnicallycorrect/pong/models/paddle/PaddleInstruction.class</exclude>
            </excludes>
          </instrumentation>
        </configuration>
      </plugin>
    </plugins>
  </reporting>
</project>
{% endhighlight %}

Again, this gives me external dependency resolution, defined unit and integration tests, and code coverage metrics.

The only thing that using Maven made harder was trying to integrate SWT into my project. Unfortunately they don't maintain an official public Maven repository which makes things unnecessarily difficult. In the end, I chose to use Swing instead just because I didn't feel like putting in too much effort to use SWT.

## Automated Tests and Cobertura
Simply put, automated testing is a must with most software. One thing my experience with developing Batarim Logger taught me was it's difficult to predict how your software is going to react to weird inputs without unit tests. A lot of times it's not even input from the user but from operating system API's or other parts of your application that might be unusual. Not testing or not being able to test those cases significantly raises the risk that your application is going to crash or end up in an invalid state.

Luckily, modern languages like Java usually have test frameworks and design patterns that greatly aid in testing code in isolation. For this project, I used JUnit to create test suites and test cases, Mockito for mocking out dependencies, and Google Guice for simplifying constructor dependency injection.

I'm particularly happy with Google Guice. As with most other dependency injection frameworks, you create a public module for your package with bindings between interfaces and classes and designate which constructor in your class is injectable. A very cool feature I discovered was that you could bind two different interfaces to the same singleton, which is particularly helpful when using the publish/subscribe design pattern. An example of this is in [GameModule](https://github.com/jeffcharles/pong/blob/master/src/main/java/com/beyondtechnicallycorrect/pong/models/game/GameModule.java "GameModule"):

{% highlight java %}
bind(MatchStateNotificationService.class).in(Singleton.class);
bind(MatchStatePublisher.class).to(MatchStateNotificationService.class);
bind(MatchStateSubscription.class).to(MatchStateNotificationService.class);
{% endhighlight %}

Each injected instance of MatchStatePublisher and MatchStateSubscription references the same MatchStateNotificationService singleton.

Code coverage is also important to measure. If you have code that is not covered, that could indicate that you are missing test cases or have code you don't need. However, it shouldn't be the only thing considered during automated testing as it will not tell you if you are missing code along with test cases. On top of that, creating tests to cover 100% of your code is typically not worth the effort. For example, I chose not to test my assertions because my tests would then require specific JVM arguments to pass (i.e., enabling assertions) and I would turn off assertions in any binaries I ship. As well, testing most pure user interface and thread management code adds minimal value, especially if you use assertions, as with the right design, failures and their locations become obvious. To me it makes sense to focus my automated testing efforts on components that could fail subtly or cause other components to fail subtly.

## Design Choices

I chose to design the game to use three threads: one for the user interface (the event dispatch thread), one for the frame processing, and one for the artificial intelligence. The threads communicate with each other using message passing to avoid the need for locks and explicit synchronization.

For example, when the player indicates that they want to move their paddle by pressing a key, the view calls a method on the app view model which sets an enum in the player paddle's instruction holder. The paddle's instruction holder is polled during the frame processing loop and the paddle's velocity is set accordingly.

Another example is how the frame processing thread presents a consistent court view model to the user interface and artificial intelligence by only updating that view model once during the frame processing loop and keeping its member objects separate from the actual entities inside the frame processor. A message is published when the court view model is updated.

## Fixing Ugly Swing/AWT Graphics

While creating the user interface for my game, I noticed that the fillOval and fillRoundedRect methods I was using on the AWT Graphics class as part of PaintComponent were not properly rounding corners. A little undocumented trick I discovered is that you can cast Graphics to Graphics2D and turn on anti-aliasing. See here for an example:

{% highlight java %}
Graphics2D g2 = (Graphics2D)g;
g2.setRenderingHint(
        RenderingHints.KEY_ANTIALIASING,
        RenderingHints.VALUE_ANTIALIAS_ON
    );
{% endhighlight %}

This fix helped enormously.

## Java vs C\#

In general, I prefer working with C# compared to Java. Most of my reasons are the same ones that other developers have raised online ad nauseam so I won't repeat them here. That being said, there are some nice language features in Java that C# doesn't have:

* package private accessibility on interfaces and classes; using C#'s internal accessibility isn't very useful if you only have one assembly
* final members cause a compiler error if they're not initialized in every constructor; C#'s readonly only causes a compiler warning by default and it disappears if you initialize the variable in only one constructor

I'm also very appreciative of the more vibrant open source community. It's difficult to find a good open source code coverage tool for .NET but very easy for Java. My impression is that there is more variety to be had in Java community than in the .NET one. Whether that variety is a good thing or bad thing and whether it makes up for the language's shortcomings is up for debate.

If you've read this far, thank you and I hope you've found some value in reading this post.
