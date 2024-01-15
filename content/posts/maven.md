---
title: maven配置分享
date: 2020-03-05 23:48:49
tags: 
- maven
---

最近在发布组件到中央仓库的过程中，总结了以下最佳实践

# 打包过程

# 最佳实践
一个项目，为了让配置更通用些。我觉得要有两个要素

- parent-pom
- dependencies-bom

这里使用maven提供的插件，处理版本占位符revision
https://www.mojohaus.org/flatten-maven-plugin/

## enode-parent

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xmlns="http://maven.apache.org/POM/4.0.0"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>org.enodeframework</groupId>
    <artifactId>enode-parent</artifactId>
    <version>${revision}</version>
    <modules>
        <module>enode</module>
        <module>bom</module>
        <module>mysql</module>
        <module>kafka</module>
        <module>rocketmq</module>
        <module>tests</module>
        <module>samples</module>
    </modules>
    <packaging>pom</packaging>

    <properties>
        <revision>1.0.2-SNAPSHOT</revision>
        <maven.skip.deploy>false</maven.skip.deploy>
        <maven.jar.version>3.0.2</maven.jar.version>
        <maven.surefire.version>3.0.0-M4</maven.surefire.version>
        <maven.deploy.version>3.0.0-M1</maven.deploy.version>
        <maven.compiler.version>3.8.1</maven.compiler.version>
        <maven.source.version>3.2.0</maven.source.version>
        <maven.war.version>3.2.3</maven.war.version>
        <maven.javadoc.version>3.1.1</maven.javadoc.version>
        <maven.jetty.version>9.4.11.v20180605</maven.jetty.version>
        <nexus.staging.version>1.6.8</nexus.staging.version>
        <maven.gpg.version>1.6</maven.gpg.version>
        <maven.flatten.version>1.1.0</maven.flatten.version>
        <maven.enforce.version>3.0.0-M2</maven.enforce.version>
    </properties>

    <name>${project.artifactId}</name>
    <description>The enodeframework is devoted to helping engineers develop scalable applications.</description>
    <url>http://www.enodeframework.org</url>

    <licenses>
        <license>
            <name>MIT License</name>
            <url>http://www.opensource.org/licenses/mit-license.php</url>
        </license>
    </licenses>
    <developers>
        <developer>
            <name>anruence</name>
            <email>anruence@gmail.com</email>
            <organizationUrl>http://www.enodeframework.org</organizationUrl>
        </developer>
    </developers>
    <scm>
        <tag>master</tag>
        <url>https://github.com/anruence/enode</url>
        <connection>scm:git:git@github.com:anruence/enode.git</connection>
    </scm>

    <distributionManagement>
        <snapshotRepository>
            <id>ossrh</id>
            <url>https://oss.sonatype.org/content/repositories/snapshots</url>
        </snapshotRepository>
        <repository>
            <id>ossrh</id>
            <url>https://oss.sonatype.org/service/local/staging/deploy/maven2/</url>
        </repository>
    </distributionManagement>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.enodeframework</groupId>
                <artifactId>enode-dependencies-bom</artifactId>
                <version>${revision}</version>
                <scope>import</scope>
                <type>pom</type>
            </dependency>
        </dependencies>
    </dependencyManagement>
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>${maven.compiler.version}</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                    <encoding>UTF-8</encoding>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-war-plugin</artifactId>
                <version>${maven.war.version}</version>
                <configuration>
                    <warName>${project.artifactId}-${project.version}</warName>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-deploy-plugin</artifactId>
                <version>${maven.deploy.version}</version>
                <configuration>
                    <skip>${maven.skip.deploy}</skip>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>${maven.surefire.version}</version>
                <configuration>
                    <skipTests>true</skipTests>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.codehaus.mojo</groupId>
                <artifactId>flatten-maven-plugin</artifactId>
                <version>${maven.flatten.version}</version>
                <configuration>
                    <updatePomFile>true</updatePomFile>
                    <flattenMode>resolveCiFriendliesOnly</flattenMode>
                </configuration>
                <executions>
                    <execution>
                        <id>flatten</id>
                        <phase>process-resources</phase>
                        <goals>
                            <goal>flatten</goal>
                        </goals>
                    </execution>
                    <execution>
                        <id>flatten.clean</id>
                        <phase>clean</phase>
                        <goals>
                            <goal>clean</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-source-plugin</artifactId>
                <version>${maven.source.version}</version>
                <executions>
                    <execution>
                        <id>attach-sources</id>
                        <goals>
                            <goal>jar-no-fork</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-javadoc-plugin</artifactId>
                <version>${maven.javadoc.version}</version>
                <executions>
                    <execution>
                        <id>attach-javadocs</id>
                        <goals>
                            <goal>jar</goal>
                        </goals>
                        <configuration>
                            <doclint>none</doclint>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

    <profiles>
        <profile>
            <id>sonatype-oss-release</id>
            <build>
                <plugins>
                    <plugin>
                        <groupId>org.apache.maven.plugins</groupId>
                        <artifactId>maven-gpg-plugin</artifactId>
                        <version>${maven.gpg.version}</version>
                        <executions>
                            <execution>
                                <id>sign-artifacts</id>
                                <phase>verify</phase>
                                <goals>
                                    <goal>sign</goal>
                                </goals>
                            </execution>
                        </executions>
                    </plugin>
                </plugins>
            </build>
        </profile>
    </profiles>
</project>

```


## enode-dependencies-bom
需要对项目依赖的项目进行整体管理，参见spring系统的bom

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xmlns="http://maven.apache.org/POM/4.0.0"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>org.enodeframework</groupId>
    <artifactId>enode-dependencies-bom</artifactId>
    <version>${revision}</version>
    <packaging>pom</packaging>
    <properties>
        <revision>1.0.2-SNAPSHOT</revision>
        <springboot.version>2.2.5.RELEASE</springboot.version>
        <vertx.version>3.8.5</vertx.version>
        <slfj4.version>1.7.30</slfj4.version>
        <guava.version>28.2-jre</guava.version>
        <gson.version>2.8.6</gson.version>
        <junit.version>4.13</junit.version>
        <rocketmq.version>4.6.1</rocketmq.version>
        <mysql.version>8.0.19</mysql.version>
        <reflections.version>0.9.12</reflections.version>
        <maven.flatten.version>1.1.0</maven.flatten.version>
        <maven.gpg.version>1.6</maven.gpg.version>
    </properties>

    <name>${project.artifactId}</name>
    <description>The enodeframework is devoted to helping engineers develop scalable applications.</description>
    <url>http://www.enodeframework.org</url>

    <licenses>
        <license>
            <name>MIT License</name>
            <url>http://www.opensource.org/licenses/mit-license.php</url>
        </license>
    </licenses>
    <developers>
        <developer>
            <name>anruence</name>
            <email>anruence@gmail.com</email>
            <organizationUrl>http://www.enodeframework.org</organizationUrl>
        </developer>
    </developers>
    <scm>
        <tag>master</tag>
        <url>https://github.com/anruence/enode</url>
        <connection>scm:git:git@github.com:anruence/enode.git</connection>
    </scm>

    <distributionManagement>
        <snapshotRepository>
            <id>ossrh</id>
            <url>https://oss.sonatype.org/content/repositories/snapshots</url>
        </snapshotRepository>
        <repository>
            <id>ossrh</id>
            <url>https://oss.sonatype.org/service/local/staging/deploy/maven2/</url>
        </repository>
    </distributionManagement>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${springboot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>io.vertx</groupId>
                <artifactId>vertx-stack-depchain</artifactId>
                <version>${vertx.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>org.enodeframework</groupId>
                <artifactId>enode</artifactId>
                <version>${revision}</version>
            </dependency>
            <dependency>
                <groupId>org.enodeframework</groupId>
                <artifactId>enode-mysql</artifactId>
                <version>${revision}</version>
            </dependency>
            <dependency>
                <groupId>org.enodeframework</groupId>
                <artifactId>enode-rocketmq</artifactId>
                <version>${revision}</version>
            </dependency>
            <dependency>
                <groupId>org.enodeframework</groupId>
                <artifactId>enode-kafka</artifactId>
                <version>${revision}</version>
            </dependency>
            <dependency>
                <groupId>org.reflections</groupId>
                <artifactId>reflections</artifactId>
                <version>${reflections.version}</version>
            </dependency>
            <dependency>
                <groupId>org.slf4j</groupId>
                <artifactId>slf4j-api</artifactId>
                <version>${slfj4.version}</version>
            </dependency>
            <dependency>
                <groupId>junit</groupId>
                <artifactId>junit</artifactId>
                <version>${junit.version}</version>
            </dependency>
            <dependency>
                <groupId>com.google.guava</groupId>
                <artifactId>guava</artifactId>
                <version>${guava.version}</version>
            </dependency>
            <dependency>
                <groupId>com.google.code.gson</groupId>
                <artifactId>gson</artifactId>
                <version>${gson.version}</version>
            </dependency>
            <dependency>
                <groupId>mysql</groupId>
                <artifactId>mysql-connector-java</artifactId>
                <version>${mysql.version}</version>
            </dependency>
            <dependency>
                <groupId>org.apache.rocketmq</groupId>
                <artifactId>rocketmq-client</artifactId>
                <version>${rocketmq.version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>
    <build>
        <plugins>
            <plugin>
                <groupId>org.codehaus.mojo</groupId>
                <artifactId>flatten-maven-plugin</artifactId>
                <version>${maven.flatten.version}</version>
                <configuration>
                    <updatePomFile>true</updatePomFile>
                    <flattenMode>resolveCiFriendliesOnly</flattenMode>
                </configuration>
                <executions>
                    <execution>
                        <id>flatten</id>
                        <phase>process-resources</phase>
                        <goals>
                            <goal>flatten</goal>
                        </goals>
                    </execution>
                    <execution>
                        <id>flatten.clean</id>
                        <phase>clean</phase>
                        <goals>
                            <goal>clean</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
    <profiles>
        <profile>
            <id>sonatype-oss-release</id>
            <build>
                <plugins>
                    <plugin>
                        <groupId>org.apache.maven.plugins</groupId>
                        <artifactId>maven-gpg-plugin</artifactId>
                        <version>${maven.gpg.version}</version>
                        <executions>
                            <execution>
                                <id>sign-artifacts</id>
                                <phase>verify</phase>
                                <goals>
                                    <goal>sign</goal>
                                </goals>
                            </execution>
                        </executions>
                    </plugin>
                </plugins>
            </build>
        </profile>
    </profiles>
</project>

```
