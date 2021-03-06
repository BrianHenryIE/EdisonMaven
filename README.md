# Intel Edison with Java and Maven on OS X

```mvn package wagon:upload-single@edison-upload wagon:sshexec@edison-run```

I tried to run a Java app on an [Intel Edison board](https://software.intel.com/en-us/iot/hardware/edison) using the version of [Eclipse supplied by Intel](https://software.intel.com/en-us/installing-the-eclipse-ide). It didn't copy over any library jars and rather than troubleshoot, I figured it I'd just set things up properly with Maven. I'm using OS X.

I'm just using the IBM Bluemix Java sample for MQTT –
[Explore MQTT and the Internet of Things service on IBM Bluemix](http://www.ibm.com/developerworks/cloud/library/cl-mqtt-bluemix-iot-node-red-app/) – which was trivial to change to a Maven project.

Otherwise, this is just an implementation of the Intel guide, [Running Java IoT Applications outside of Eclipse](https://software.intel.com/en-us/node/596288), through Maven.

## Building a fat jar

To fix the problem with the jars, use the [Maven Shade plugin](http://maven.apache.org/plugins/maven-shade-plugin/) to build an uber-jar. You'll need to [set the main class](http://maven.apache.org/plugins/maven-shade-plugin/examples/executable-jar.html) and [exclude manifest signature files](http://stackoverflow.com/a/6743609) giving you a plugin configuration like so:

```
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-shade-plugin</artifactId>
	<version>2.4.2</version>
	<configuration>
		<filters>
			<filter>
				<artifact>*:*</artifact>
				<excludes>
					<exclude>META-INF/*.SF</exclude>
					<exclude>META-INF/*.DSA</exclude>
					<exclude>META-INF/*.RSA</exclude>
				</excludes>
			</filter>
		</filters>
		<transformers>
			<transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
				<mainClass>ie.brianhenry.testclient.DeviceTest</mainClass>
			</transformer>
		</transformers>
	</configuration>
	<executions>
		<execution>
			<phase>package</phase>
			<goals>
				<goal>shade</goal>
			</goals>
		</execution>
	</executions>
</plugin>
```

Running ```mvn package``` should build a nice jar for you under your ```target``` folder.

## Copying the jar by scp

Add your username/password to your Maven [settings.xml](https://maven.apache.org/settings.html) by ```nano ~/.m2/settings.xml``` 

```
<settings>
	<servers>
		<server>
			<id>bhedison</id>
			<username>root</username>
			<password>password</password>
		</server>   
	</servers>
</settings>
```
Then use ```ctrl O``` to save and ```ctrl X``` to exit.

(We probably should be [encrypting the password](https://maven.apache.org/guides/mini/guide-encryption.html) or using a key)

Add the [servers-maven-extension](https://github.com/shyiko/servers-maven-extension) to your pom's ```build/extensions``` so we can reference the server settings using ```${settings.servers.edison.username}``` notation.

```
<extension>
	<groupId>com.github.shyiko.servers-maven-extension</groupId>
	<artifactId>servers-maven-extension</artifactId>
	<version>1.3.0</version>
</extension>
```
To find your Edison's IP address, connect the USB cables and follow the instructions at [Setting up a serial terminal](https://software.intel.com/en-us/setting-up-serial-terminal-intel-edison-board) to make a serial connection, then type ```ifconfig``` once logged in. If you don't see an IP address beside ```ined addr:``` under ```wlan0``` run ```configure_edison --wifi``` to connect to a network. Add the IP address to your pom's ```properties```, being careful to match the server name throughout with what you used in your settings.xml:

```
<edisonIp>192.168.0.7</edisonIp>
<edisonServer>bhedison</edisonServer>
<edisonUsername>${settings.servers.bhedison.username}</edisonUsername>
<edisonPassword>${settings.servers.bhedison.password}</edisonPassword>
```
Add [Maven Wagon plugin](http://www.mojohaus.org/wagon-maven-plugin/) to your extensions:

```
<extension>
	<groupId>org.apache.maven.wagon</groupId>
	<artifactId>wagon-ssh</artifactId>
	<version>2.10</version>
</extension>
```
then the plugin itself:

```
<plugin>
	<groupId>org.codehaus.mojo</groupId>
		<artifactId>wagon-maven-plugin</artifactId>
		<version>1.0</version>
		<executions>
			<execution>
				<id>edison-upload</id>
				<goals>
					<goal>upload-single</goal>
				</goals>
				<configuration>
					<serverId>${edisonServer}</serverId>
					<fromFile>${project.build.directory}/${project.build.finalName}.jar</fromFile>
					<url>scp://${edisonIp}/home/${edisonUsername}</url>
				</configuration>
			</execution>
		</executions>
</plugin>
```

```mvn wagon:upload-single@edison-upload``` will now copy the jar to your Edison.

## Executing the remote jar

```
<execution>
	<id>edison-run</id>
	<phase>install</phase>
	<goals>
		<goal>sshexec</goal>
	</goals>
	<configuration>
		<serverId>${edisonServer}</serverId>
		<url>scp://${edisonIp}/</url>
		<displayCommandOutputs>true</displayCommandOutputs>
		<commands>
			<command>java -jar ${project.build.finalName}.jar &amp;</command>
		</commands>
	</configuration>
</execution>
```

```mvn wagon:sshexec@edison-run``` will execute the remote jar!
## All together
```mvn package wagon:upload-single@edison-upload wagon:sshexec@edison-run``` will package (i.e. test, compile, package), copy and execute your app.

## Status/Caveats

Currently, this leaves the ```mvn``` process running once the SSH session is opened, without any remote output displayed, until Crtl-C is hit, after which point the application continues to run on the Edison.

I'll hopefully put up a full sample pom and project soon. 

Remote debugging would be nice.

I haven't looked into MRAA yet. Maybe things will just work, or maybe it's just a matter of editing the SSH java command to explicitly state where the libraries are sitting on the Edison. In terms of keeping them synced between the Edison and dev environment, Maven Shade might add it to the fat jar.