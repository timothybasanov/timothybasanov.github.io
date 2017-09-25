---
title:  "Maven Integration Tests and Jetty with SSL Enabled"
---

Running integration tests with *Jetty* using *maven* usually is plain and awesome. At least not until you try to enable *SSL*. And then suddenly everything goes to hell. As I have not found a definitive source of a simple working *Jetty* config and I'm sharing my findings.

To run integration tests with Jetty under maven with SSL you'll need to

 - Start and stop Jetty before and after integration tests
 - Generate *Jetty* SSL keys in a *PKCS12* format using `keytool`
 - Config Jetty to enable SSL and use generated keys

<!--more-->

## Starting and stopping Jetty before and after integration tests

This part is easy and is well known. Just add this to your `pom.xml`:

```xml
<plugin>
    <groupId>org.eclipse.jetty</groupId>
    <artifactId>jetty-maven-plugin</artifactId>
    <version>9.3.8.v20160314</version>
    <executions>
        <execution>
            <id>jetty-start</id>
            <phase>pre-integration-test</phase>
            <goals>
                <goal>start</goal>
            </goals>
        </execution>
        <execution>
            <id>jetty-stop</id>
            <phase>post-integration-test</phase>
            <goals>
                <goal>stop</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

It executes *Jetty* plugin before and after integration tests to start and stop local *Jetty* instance.

## Generating *Jetty* SSL keys in a *PKCS12* format using `keytool`

> Using PKCS12 format for keystore allows you to load same keys in other programs like Wireshark.

Use script to generate an RSA keypair to use for SSL `src/main/scripts/generate-integration-test-ssl-key`:

```sh
##!/usr/bin/env bash

mkdir -p target
cd target
if [ ! -f ssl/jetty.key ]; then
  mkdir -p ssl
  cd ssl
  yes | keytool -genkey -keyalg RSA -alias jetty -noprompt \
      -keystore jetty.p12 -storetype pkcs12 \
      -storepass changeit -keypass changeit
fi
```

You may run it once manually or automatically. To automate this script has to be run before *Jetty* is started, add this to your `pom.xml`:

```xml
<plugin>
    <!-- Execute external shell scripts for integration tests -->
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>exec-maven-plugin</artifactId>
    <version>1.5.0</version>
    <executions>
        <!-- Generate SSL keys -->
        <execution>
            <id>ssl-keys</id>
            <phase>pre-integration-test</phase>
            <goals>
                <goal>exec</goal>
            </goals>
            <configuration>
                <executable>${build.scriptSourceDirectory}/generate-integration-test-ssl-key</executable>
            </configuration>
        </execution>
    </executions>
</plugin>
```

## Configuring Jetty to enable SSL and use generated keys

Recent versions of *Jetty* do not support *SSL/HTTPS* configuration through the `pom.xml`, it has to be a separate property file. Add this config to your *Jetty*'s section in `pom.xml`:

```xml
<configuration>
    <jettyXml>${project.basedir}/src/main/config/jetty-https.xml</jettyXml>
    <systemProperties>
        <systemProperty>
            <name>jetty.ssl.keyStorePath</name>
            <value>${project.build.directory}/ssl/jetty.p12</value>
        </systemProperty>
        <systemProperty>
            <name>org.eclipse.jetty.ssl.password</name>
            <value>changeit</value>
        </systemProperty>
    </systemProperties>
</configuration>
```

And here is the simplest `jetty.xml` possible that enables *SSL/HTTPS* `src/main/config/jetty-https.xml`:

```xml
<?xml version="1.0"?>
<!DOCTYPE Configure PUBLIC "-//Jetty//Configure//EN" "http://www.eclipse.org/jetty/configure_9_3.dtd">

<!-- Simplest working Jetty SSL configuration serving on https://localhost:8443-->
<Configure id="Server" class="org.eclipse.jetty.server.Server">
    <Call name="addConnector">
        <Arg>
            <!-- Setting up a new connector that serves HTTPS -->
            <New class="org.eclipse.jetty.server.ServerConnector">
                <Arg name="server">
                    <Ref refid="Server"/>
                </Arg>
                <Arg name="factories">
                    <Array type="org.eclipse.jetty.server.ConnectionFactory"/>
                </Arg>
                <!-- Port is either default 8443 or supplied via -Djetty.ssl.port=8443 -->
                <Set name="port">
                    <Property name="jetty.ssl.port" deprecated="ssl.port" default="8443"/>
                </Set>
                <Call name="addConnectionFactory">
                    <Arg>
                        <!-- You need to setup SSL-specific connection factory -->
                        <New class="org.eclipse.jetty.server.SslConnectionFactory">
                            <Arg name="next">http/1.1</Arg>
                            <Arg name="sslContextFactory">
                                <New class="org.eclipse.jetty.util.ssl.SslContextFactory">
                                    <!-- Setting up path to a keystore -->
                                    <Set name="keyStorePath">
                                        <Property name="jetty.ssl.keyStorePath"/>
                                    </Set>
                                    <!-- Setting up passworrd for a keystore -->
                                    <Set name="keyStorePassword">
                                        <Property name="org.eclipse.jetty.ssl.password"/>
                                    </Set>
                                    <!-- Disabling Diffie-Hellman key exchange
                                         to simplify traffic decryption -->
                                    <Call name="addExcludeCipherSuites">
                                        <Arg>
                                            <Array type="String">
                                                <Item>.*DHE.*</Item>
                                            </Array>
                                        </Arg>
                                    </Call>
                                </New>
                            </Arg>
                        </New>
                    </Arg>
                </Call>
                <!-- And serve HTTP after SSL decryption -->
                <Call name="addConnectionFactory">
                    <Arg>
                        <New class="org.eclipse.jetty.server.HttpConnectionFactory">
                            <Arg name="config">
                                <New class="org.eclipse.jetty.server.HttpConfiguration"/>
                            </Arg>
                        </New>
                    </Arg>
                </Call>
            </New>
        </Arg>
    </Call>
</Configure>
```

And that's it. Now you can run your local *Jetty* serving from `https://localhost:8443`.
