            <plugin>
                <!-- Run Jetty (like Tomcat but different) for integration tests -->
                <groupId>org.eclipse.jetty</groupId>
                <artifactId>jetty-maven-plugin</artifactId>
                <version>9.3.8.v20160314</version>
                <executions>
                    <!-- Start Jetty before integration tests and shutdown after -->
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
            </plugin>
            
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
            
src/main/config/jetty-https.xml:

<?xml version="1.0"?>
<!DOCTYPE Configure PUBLIC "-//Jetty//Configure//EN" "http://www.eclipse.org/jetty/configure_9_3.dtd">

<!-- Simplest working Jetty SSL configuration serving on https://localhost:8443-->
<Configure id="Server" class="org.eclipse.jetty.server.Server">
    <Call name="addConnector">
        <Arg>
            <New id="sslConnector" class="org.eclipse.jetty.server.ServerConnector">
                <Arg name="server">
                    <Ref refid="Server"/>
                </Arg>
                <Arg name="factories">
                    <Array type="org.eclipse.jetty.server.ConnectionFactory"/>
                </Arg>
                <Set name="port">
                    <Property name="jetty.ssl.port" deprecated="ssl.port" default="8443"/>
                </Set>
                <Call name="addConnectionFactory">
                    <Arg>
                        <New class="org.eclipse.jetty.server.SslConnectionFactory">
                            <Arg name="next">http/1.1</Arg>
                            <Arg name="sslContextFactory">
                                <New class="org.eclipse.jetty.util.ssl.SslContextFactory">
                                    <Set name="keyStorePath">
                                        <Property name="jetty.ssl.keyStorePath"/>
                                    </Set>
                                    <Set name="keyStorePassword">
                                        <Property name="org.eclipse.jetty.ssl.password"/>
                                    </Set>
                                </New>
                            </Arg>
                        </New>
                    </Arg>
                </Call>
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

src/main/scripts/generate-integration-test-ssl-key

#!/usr/bin/env bash

mkdir -p target
cd target
if [ ! -f ssl/jetty.key ]; then
  mkdir -p ssl
  cd ssl
  yes | keytool -genkey -keyalg RSA -alias jetty -noprompt \
      -keystore jetty.p12 -storetype pkcs12 \
      -storepass changeit -keypass changeit
fi

