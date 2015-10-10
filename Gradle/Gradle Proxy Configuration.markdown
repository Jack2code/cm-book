# Gradle Proxy Configuration

标签（空格分隔）： Gradle

---

[TOC]

==> [原文链接][1]

Configuring an HTTP proxy (for downloading dependencies, for example) is done via standard JVM system properties. These properties can be set directly in the build script; for example, setting the proxy host would be done with System.setProperty('http.proxyHost', 'www.somehost.org'). Alternatively, the properties can be specified in a `gradle.properties` file, either in the build's root directory or in the Gradle home directory.

## Configuring an HTTP proxy
```
gradle.properties
systemProp.http.proxyHost=www.somehost.org
systemProp.http.proxyPort=8080
systemProp.http.proxyUser=userid
systemProp.http.proxyPassword=password
systemProp.http.nonProxyHosts=*.nonproxyrepos.com|localhost    
There are separate settings for HTTPS.
```
## Configuring an HTTPS proxy
```
gradle.properties
systemProp.https.proxyHost=www.somehost.org
systemProp.https.proxyPort=8080
systemProp.https.proxyUser=userid
systemProp.https.proxyPassword=password
systemProp.https.nonProxyHosts=*.nonproxyrepos.com|localhost
```
We could not find a good overview for all possible proxy settings. One place to look are the constants in a file from the Ant project. Here's a link to the Subversion view. The other is a Networking Properties page from the JDK docs. If anyone knows of a better overview, please let us know via the mailing list.



  [1]: https://docs.gradle.org/current/userguide/build_environment.html