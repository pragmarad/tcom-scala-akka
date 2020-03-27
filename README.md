Tcom - TCP communication using Akka Streams PoC
===============================================
Inception year: **2020**

# Overview
This project was originated from idea of checking communication technology stack of 2 machines interacting 
via TCP as a transport protocol and using some kind of streaming.  
Idea about particular technical stack itself (Akka streams with scala and **akka.stream.scaladsl.Tcp** support) came from lessons 
for Akka streams in a course ["Reactive Scala"](https://courses.edx.org/courses/course-v1:EPFLx+scala-reactiveX+1T2019/course/).
So, credits and inspiration of this project come to creators of this course (such as Martin Odersky, Roland Kuhn etc).

NOTE: There is a trend of TCP getting replaced with UDP in newer protocols  
      (such as Google UDP based QUIC or remoting Akka Artery that utilizing UDP based Aeron). 
      So, UDP based streaming is likely the next contender for exploring.

## Participants
There are 2 participants of interaction: **client** and **server**. They communicate through host and port pair. Host address can be DNS name or IP.
Their interaction roughly can be shown as:  
**client** - Akka_Stream[*req_message] -> **server**  
**client** <- Akka_Stream[*resp_message] - **server**

In [Plantuml](https://plantuml.com/sequence-diagram) it can be shown as a [model](models/client_server_seq.puml).

# Goals
Components to be developed in this project, should answer to questions:
1. What is latency between nodes in various deployments (see variations below)?
2. How reliable are streams? How they react to network disruptions / slow traffic?
3. How streams deal with errors on both client and server sides (consider simulation of different error/failure cases)?
4. How well TLS TCP works for Akka streams (with mutual authc, hostname restrictions, firewalls, VPNs etc)?
5. How to make both sides configurable on level that allows worry free deployments into various environments?

Answers to these and more imaginable questions should lead app architects/designers to conclusion whether it worth 
to use Akka TCP or it's irrelevant/unsafe etc for particular system. 
NOTE: It's not about direct using as TCP is too low level, but about better understanding, when e.g. 
Akka remoting (via cluster these days) is really good candidate for use.

Deployment variations for client and server:
   - on the same host machine
   - on containers(Docker compose as starter, then K8S) on the same machine
   - on machines distributed this or that way (local / global networks)
   - on machines in the same cloud vendor
   - on machines in different cloud vendors
   - on containers(Docker compose as starter, then K8S) in the same cloud vendor
   - on containers in the same cloud vendor

# Design
At least in first iterations design is to be brutally simple: just create a client that can send configurable message multiple times (to be configurable as well) to 
a server with particular host and port. A server can either return back the same message or perform some stream flow logic.
Akka streams approach on very basic level is about 3 components: 
1. Source 
2. Flow 
3. Sink
They make chain: 
```
<Source> -> <Flow_1> -> .. -> <Flow_n> -> <Sink>
```

NOTE: in reality more complex stream execution graphs can be constructed, but this is topic on its own.  
Both Tcom client and server will follow the same kind of chain as shown above. 
Devil of variations will be in their implementations.   
NOTE: **Tcom** is abbreviation of "Telecom". 

One of essence of design of this particular project is to be as close to "bare metal" as possible. 
System itself should add only minor overhead to TCP Akka streams work. 
Metrics can be (or more like "should") gathered with 3rd party toolkits (nagios, prometheus, influxdb, nmap, wireshark etc). 

## Components
In [Plantuml](https://plantuml.com/component-diagram) components can be shown as a [model](models/client_server_components.puml).

From package prospective, this project split into crosscutting concern modules (1 for now - **commons**) 
and tier ones (for client and server).
Those packages:

### Tcom Crosscut Commons Akka
Name: tcom-crosscut-commons-akka ([Github](https://github.com/pragmarad/tcom-crosscut-commons-akka) , [Bintray](https://bintray.com/pragmarad-tech/tcom-scala-akka/tcom-crosscut-commons-akka))
This module contains utilities(see **HoconConfigUtil** and **StringUtil**) and constans (see **ConfigCommonConstants**) reusable across other modules (tiers).

### Tcom Client Tier
Name: tcom-tier-cli-akka ([Github](https://github.com/pragmarad/tcom-tier-cli-akka) , [Bintray](https://bintray.com/pragmarad-tech/tcom-scala-akka/tcom-tier-cli-akka))
This module contains client tier functionality.

The most important class in this module is:
```scala
class TcpAkkaStreamClient(host: String, port: Int, actorSysName: String) {
  def start(message: String, frequency: FiniteDuration, maxBurstCount: Int): Unit = { .. }
```

TCP support is added into Flow in form:
```scala
	Tcp(system).outgoingConnection(host, port)
```

For dealing with launch of this client (with arguments/options and configs) a class **TcpAkkaStreamClientApp** was created.

### Tcom Server Tier
Name: tcom-tier-srv-akka ([Github](https://github.com/pragmarad/tcom-tier-srv-akka) , [Bintray](https://bintray.com/pragmarad-tech/tcom-scala-akka/tcom-tier-srv-akka))
This module contains server tier functionality.

The most important class in this module is:
```scala
class TcpAkkaStreamServer(host: String, port: Int, actorSysName: String) {
  def start(flowLogic: Flow[ByteString, ByteString, _]): Unit = {
    Tcp(system).bindAndHandle(flowLogic, host, port)
  }	
}
```
Flow parameter allows more flexibility in choosing stream flow cases. For beginning just "mirroring" flow is implemented.

TCP support is added into Flow in form:
```scala
	Tcp(system).outgoingConnection(host, port)
```

For dealing with launch of this server (with arguments/options and configs) a class **TcpAkkaStreamServertApp** was created.

### Dependencies:
 - **tcom-tier-cli-akka** -> **tcom-crosscut-commons-akka**
 - **tcom-tier-srv-akka** -> **tcom-crosscut-commons-akka**
 - **tcom-crosscut-commons-akka** -> {akka-stream, akka-stream-typed, akka-actor-typed, slf4j, com.monovore.decline}

Why **com.monovore.decline**? This is cute [Cats](https://typelevel.org/cats/) based (and who doesn't like cats?) arguments parser. 
NOTE: Yes, it's possible to use scala "match" functionality for parsing app args, but it's more like "reinventing the wheel"
      , which is avoided in this project. Basically, this project is lessen about coding, more - for providing convenient way of launching TCP akka stream tests.

## Configs and args handling
Flexibility of configuring apps is major feature of this project. Devops team should have many ways to configure (or skip at all) apps. 
Apps need to be smootly "blendable" into modern environments.

Common approach of reading args to all tiers is:
1. Get command line arguments (e.g. --srvhost myhost).
2. (if not found in arguments) Get values from env variables (e.g. TCOM_SRV_HOST='myhost').
3. (if not found in env variables) Get values from config (e.g. tcom.cli.akka.srvhost='myhost').

NOTE: Config format is Lightbend' [HOCON](https://github.com/lightbend/config/blob/master/HOCON.md).

### Tcom Client Tier Config and Args
The class **TcpAkkaStreamClientApp** is entry point into client streams from outside world. It accepts args:

| Name       	 | Short name | Example	  | Config name   | Env name 	  | Description          |
| :------------- |:---------- | :-------: | :------------ |:--------------|:---------------------|
| srvhost      	 | h 		  | localhost | host          | TCOM_SRV_HOST | Host name (DNS / IP) |
| srvport     	 | p      	  | 1661      | port          | TCOM_SRV_PORT | Host port 			 |
| message 		 | p      	  | msg       | message 	  | N/A 		  | Message content 	 |
| frequencymsecs | f      	  | 1000      | frequencymsecs| N/A 		  | How often to resend a message, in milliseconds |
| maxburstcount  | m      	  | 10        | maxburstcount | N/A 		  | How many to send in burst |

- For Config name root path is **tcom.cli.akka**. 
- For name '--' needs to be used, whereas for short - '-' (--srvhost myhost vs -h myhost).

Examples of call:
```sh
launch_cli.sh --srvhost localhost --srvport 1661 --message 'msg' --frequencymsecs 1000 --maxburstcount 10
launch_cli.sh -h localhost -p 1661 -m 'msg' -f 1000 -m 10
```
NOTE: All parameters are optional, reasonabale defaults from env/config will be used. 
If to put both client and server deployments on the same server, all should work with just using scripts with all defaults. 

Config class **application.tcom.conf** will be put in classpath during build. Example config content:
```json
tcom.cli.akka {
  host = "127.0.0.1"
  host = ${?TCOM_SRV_HOST}
  port = "1661"
  port = ${?TCOM_SRV_PORT}
  message = "tst"
  frequencymsecs = 1000
  maxburstcount = 10
  actor-system-name = "TcomCliSys"
}
```

### Tcom Server Tier Config and Args
The class **TcpAkkaStreamServerApp** is entry point into server streams from outside world. It accepts args:

| Name       	 | Short name | Example	  | Config name   | Env name 	  | Description          |
| :------------- |:---------- | :-------: | :------------ |:--------------|:---------------------|
| srvhost      	 | h 		  | localhost | host          | TCOM_SRV_HOST | Host name (DNS / IP) |
| srvport     	 | p      	  | 1661      | port          | TCOM_SRV_PORT | Host port 			 |

- For Config name root path is **tcom.srv.akka**. 
- For name '--' needs to be used, whereas for short - '-' (--srvhost myhost vs -h myhost).

Examples of call:
```sh
launch_srv.sh --srvhost 127.0.0.1 --srvport 1661
launch_srv.sh -h 127.0.0.1 -p 1661
```
NOTE: All parameters are optional, reasonabale defaults from env/config will be used. 
If to put both client and server deployments on the same server, all should work with just using scripts with all defaults. 

Config class **application.tcom.conf** will be put in classpath during build. Example config content:

```json
tcom.srv.akka {
  host = "127.0.0.1"
  host = ${?TCOM_SRV_HOST}
  port = "1661"
  port = ${?TCOM_SRV_PORT}
  actor-system-name = "TcomSrvSys"
}
```

## Logging
[SLF4J](http://www.slf4j.org/) with logback impl is used. A file **logback.xml** will be put in classpath while building packages.
Important note about appenders: they are wrapped with async, like this:
```xml
    <appender name="ASYNC_FILE" class="ch.qos.logback.classic.AsyncAppender">
        <queueSize>500</queueSize>
        <discardingThreshold>0</discardingThreshold>
        <appender-ref ref="FILE"/>
    </appender>
``` 
There are minor concerns its async nature may affect work of akka internals.

## Status
NOTE: better source of history is git itself, only basic / major updates are mentioned here.
* 2020-03-26 - [0.1.0] Doc init version added.  

# Roadmap
NOTE: Also, see **Goals** section.  
1. Prepare modules for various deployments (local, containers, clouds).
2. Make config file name configurable for more flexibility.
3. Describe (either in this "codebase" or in tier modules) various deployment configs, such as helm charts, docker compose or K8S yamls.
4. Basic internal metrics for gathering perfromance stats need to be added (but external tools for measure things are just fine for start).
5. Simplified aproach of causing periodical errors/failures on client and server sides is needed. For instance, some symbols in a message could cause server response timeout.
6. Add real streaming from potentially large source (big file etc).