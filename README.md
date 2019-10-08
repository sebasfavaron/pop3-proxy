
##Making the executables

###Proxy
```
cd server/src/
make
```
generated artifact will be in server/src/

###SPCP Client
```
cd client/src/
make
```
generated artifact will be in client/src/

###Generacion del stripmime
```
cd stripmime/
make
```
generated artifact will be in stripmime/

##Running artifacts and their options

###Proxy

Run in server/src:
```
./proxyPop [POSIX STYLE OPTIONS] <origin-server>
```
 Where POSIX STYLE OPTIONS are:

```-e filter-error-file      specifies error file, /dev/null by default

-h                        prints this help dialogue

-l pop3-address           specifies the address where the proxy is listening, every interface by default

-L config-address         specifies the address where the SPCP service is listening, loopback by default

-m message                message used to replace filtered content 

-M censored-media-types   list of censured media types

-o management-port        port where SPCP service listens, 9090 by default

-p local-port             port where proxy listens for incoming TCP connections, 1110 by default

-P origin-port            TCP port where origin server will be listening, 110 by default

-t cmd                    specifies command to execute for the transformations

-v                        prints proxy version
```
###Client

Run in client/src: 
```
./spcpClient [POSIX STYLE OPTIONS] 
```
asumes SPCP server is 127.0.0.1:9090

POSIX STYLE OPTIONS are:

```-L config-address         dirección donde se encuentra el servidor SPCP

-o management-port        el puerto en donde se encuentra escuchando el servidor SPCP
```

###Stripmime 

Run in stripmime/:
```
./stripmime <optional-input-file>
```

Stripmime uses environment variables to work, these are:

FILTER_MEDIAS the media types to filter, in csv format

FILTER_MSG message to replace censured media types with
