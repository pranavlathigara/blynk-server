# What is Blynk?
Blynk is a platform with iOS and Android apps to control Arduino, Raspberry Pi and the likes over the Internet.  
You can easily build graphic interfaces for all your projects by simply dragging and dropping widgets.
If you need more information, please follow these links:
* [Kickstarter](https://www.kickstarter.com/projects/167134865/blynk-build-an-app-for-your-arduino-project-in-5-m/description).
* [Blynk downloads, docs, tutorials](http://www.blynk.cc)
* [Blynk community](http://community.blynk.cc)
* [Facebook](http://www.fb.com/blynkapp)
* [Twitter](http://twitter.com/blynk_app)

# Blynk server
Blynk Server is an Open-Source [Netty](https://github.com/netty/netty) based Java server, responsible for forwarding messages between Blynk mobile application and various microcontroller boards (i.e. Arduino, Raspberry Pi. etc).
**Take latest build [here](https://github.com/blynkkk/blynk-server/releases).**

[ ![Build Status](https://travis-ci.org/blynkkk/blynk-server.svg?branch=master)](https://travis-ci.org/blynkkk/blynk-server)

# Requirements
Java 8 required. (OpenJDK, Oracle)

# GETTING STARTED
Right now Blynk server uses 2 ports. 1 port is used for hardware and second one is used for the mobile apps. This is done due to the lack of security mechanism and low resources on microcontroller boards (e.g. Arduino UNO).
By default, mobile application uses 8443 port and is based on SSL/TLS sockets. Default hardware port is 8442 and is based on plain TCP/IP sockets.

## Quick local server setup

+ Make sure you are using Java 8

        java -version
        Output: java version "1.8.0_40"

+ Run the server on default 'hardware port 8442' and default 'application port 8443' (SSL port)

        java -jar server-{PUT_LATEST_VERSION_HERE}.jar

+ Run the server on custom ports

        java -jar server-{PUT_LATEST_VERSION_HERE}.jar -hardPort 8442 -appPort 8443

## Advanced local server setup
If you need more flexibility, you can extend server with more options by creating server.properties file in same folder as server.jar. Example could be found [here](https://github.com/blynkkk/blynk-server/blob/master/server/tcp-server/src/main/resources/server.properties).
server.properties options:

+ Application port

        app.ssl.port=8443
        
+ For simplicity Blynk already provides server jar with build-in SSL certificates, so you have working server out of the box via SSL/TLS sockets. But as certificate and it's private key are in public this is totally not secure. So in order to fix that you need to provide your own certificates. And change below properties with path to your cert. and private key and it's password. See how to generate self-signed certificates [here](https://github.com/blynkkk/blynk-server#generate-ssl-certificates)

        server.ssl.cert=/server_embedded.crt
        server.ssl.key=/server_embedded.pem
        server.ssl.key.pass=pupkin123

+ Hardware port

        server.default.port=8442

+ User profiles folder. Folder in which all users profiles will be stored. By default System.getProperty("java.io.tmpdir")/blynk used. Will be created if not exists

        data.folder=/tmp/blynk

+ Folder for all application logs. Will be created if it doesn't exist

        logs.folder=./logs

+ Log debug level. Possible values: trace|debug|info|error. Defines how precise logging will be. From left to right -> maximum logging to minimum

        log.level=trace

+ Maximum allowed number of user dashboards. This value can be changed without restarting the server. ("Reloadable" below)

        user.dashboard.max.limit=10

+ 100 Req/sec rate limit per user. Reloadable

        user.message.quota.limit=100

+ In case user exceeds quota limit - response error returned only once in specified period (in Millis). Reloadable

        user.message.quota.limit.exceeded.warning.period=60000

+ Maximum allowed user profile size. In Kb's. Reloadable

        user.profile.max.size=128

+ In-memory storage limit for storing *read* values from hardware

        user.in.memory.storage.limit=1000
        
+ Maximum allowed number of notification queue. Queue responsible for processing email, pushes, twits sending. Because of performance issue - those queue is processed in separate thread, this is required due to blocking nature of all above operations. Usually limit shouldn't be reached
        
        notifications.queue.limit=10000

+ Period for flushing all user DB to disk. In millis

        profile.save.worker.period=60000

+ Specifies maximum period of time when application socket could be idle. After which socket will be closed due to non activity. In seconds. Default value 600 if not provided

        app.socket.idle.timeout=600

+ Specifies maximum period of time when hardware socket could be idle. After which socket will be closed due to non activity. In seconds. Default value 15 if not provided

        hard.socket.idle.timeout=15

### Generate SSL certificates

+ Create key
        
        openssl genrsa -out server.key 2048
        
+ Create new cert request
        
        openssl req -new -out server.csr -key server.key

+ Generate self-signed request

        openssl x509 -req -days 1825 -in server.csr -signkey server.key -out server.crt
        
+ Convert server.key to PKCS#8 private key file in PEM format

        openssl pkcs8 -topk8 -inform PEM -outform PEM -in server.key -out server.pem
        
As output you'll retrieve server.crt and server.pem files that you need to provide for server.ssl properties.

### Behind wifi router
If you want to run Blynk server behind WiFi-router and want it to be accessible from the Internet, you have to add port-forwarding rule on your router. This is required in order to forward all of the requests that come to the router within the local network to Blynk server.

### Performance
Currently server handles 20k req/sec with SSL and 40k req/sec without SSL hardware messages on VM with 2-cores of Intel(R) Xeon(R) CPU E5-2660 @ 2.20GHz. With high load - memory consumption could be up to 1 GB of RAM.

## App Client (emulates Smartphone App)

+ To emulate the Smartphone App client:

        java -jar client-${PUT_LATEST_VERSION_HERE}.jar -mode app -host localhost -port 8443


+ In this client: register new user and/or login with the same credentials

        register username@example.com UserPassword
        login username@example.com UserPassword


+ Save profile with simple dashboard

        saveProfile {"dashBoards":[{"id":1, "name":"My Dashboard", "boardType":"UNO"}]}


+ Get the Auth Token for hardware (e.g Arduino)

        getToken 1

+ Activate dashboard

        activate 1

+ You will get server response similar to this:

    	00:05:18.100 TRACE  - Incomming : GetTokenMessage{id=30825, command=GET_TOKEN, length=32, body='33bcbe756b994a6768494d55d1543c74'}

Where `33bcbe756b994a6768494d55d1543c74` is your Auth Token.

## Hardware Client (emulates Hardware)

+ Start new client and use received Auth Token to login

    	java -jar client-${PUT_LATEST_VERSION_HERE}.jar -mode hardware -host localhost -port 8442
    	login 33bcbe756b994a6768494d55d1543c74
   

You can run as many clients as you want.

Clients with the same credentials and Auth Token are grouped into one Session and can send messages to each other.
All client’s commands are human-friendly, so you don't have to remember the codes.

## Hardware Commands

Before sending any read/write commands to hardware, application must first send “init” command.
"Init" command is a 'hardware' command which sets all the Pin Modes(pm). Here is an example of "init" command:

    	hardware pm 1 in 13 out 9 out 8 in

// TODO: take description about pin modes from Blynk Arduino library readme
// TODO Describe separation with Zeroes in pinmode command

In this example you set pin 1 and pin 8 to 'input’ PIN_MODE. This means this pins will read values from hardware (graph, display, etc).
Pins 13 and 9 have 'output’ PIN_MODE. This means that these pins will we writable (button, slider).

List of hardware commands:

+ Digital write:

    	hardware dw 9 1
    	hardware dw 9 0


+ Digital read:

    	hardware dr 9
    	You should receive response: dw 9 <val>


+ Analog write:

    	hardware aw 14 123


+ Analog read:

    	hardware ar 14
        You should receive response: aw 14 <val>


+ Virtual write:

    	hardware vw 9 1234
        hardware vw 9 string
        hardware vw 9 item1 item2 item3
        hardware vw 9 key1 val1 key2 val2

 
+ Virtual read:

    	hardware vr 9
    	You should receive response: vw 9 <values>


## Licensing
[MIT license](https://github.com/blynkkk/blynk-server/blob/master/license.txt)
