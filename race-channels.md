# RACE Channels
The RACE system is designed to use a variety of different unconventional or weird methods for communication between nodes to avoid being detected or blocked. We use the term ___channel___ to refer to distinct methods. 
We provide a list of channels below, along with the `deployment create` arguments to use to pull them in as part of a testing deployment as described [here](https://github.com/tst-race/race-quickstart/blob/main/README.md). The client support and android support fields are provided to help with channel selection: in order to function, you will need to select _at least one_ channel that supports clients (and supports android, if using android clients) to run a RACE network deployment.


| Channel Name | Client Support? | Android Support? | Description |
|- |- |- |- |
| [twoSixDirectCpp](#twoSixDirectCpp) | :x: | :x: | Example/testing channel, a simple TCP socket |
| [twoSixIndirectCpp](#twoSixIndirectCpp) | :white_check_mark: | :white_check_mark: | Example/testing channel, base64-encodes data and posts to a local redis server | 
| [Obfs](#obfs)  | :x: | :x: | Reliable look-like-nothing TCP connection |
| [Snowflake](#snowflake) | :x: | :x: | Encrypted WebRTC connection |
| [Raven](#raven) | :white_check_mark: | :x: | Sends data as PGP-encrypted emails with realistic sending patterns |
| [ssEmail](#ssEmail) | :white_check_mark: | :white_check_mark: | Encodes data in steganographically _generated_ images attached to emails |
| [ssRedis](#ssRedis) | :white_check_mark: | :x: | Encodes data in steganographically _generated_ images posted to a redis "whiteboard" |
| [destiniPixelfed](#destiniPixelfed) | :white_check_mark: | :white_check_mark: | Steganographically encodes data into existing images posted to a redis "whiteboard" |
| [destiniAvideo](#destiniAvideo) | :white_check_mark: | :x: | Steganographically encodes data into videos posted to a redis "whiteboard" |
| [destiniDash](#destiniDash) | :x: | :x: | Steganographically encodes data into videos streamed from one RACE node to another; uses destiniPixelfed as a signaling mechanism for orchestrating video streams |
| [butkusLocalRedis](#butkus)* | :white_check_mark: | :x: | Encodes data into _generated_ natural language text; *: decomposed Encoding _component_, used in combination with other components, see [Decomposed Comms Plugins]() |



## twoSixDirectCpp
__Deployment Create Argument:__ 
```
--comms-channel=twoSixDirectCpp \
--comms-kit=tag=2.6.0-v1,org=tst-race,repo=race-core,asset=plugin-comms-twosix-cpp.tar.gz
```

__Source Code Repository:__ https://github.com/tst-race/https://github.com/tst-race/race-core/tree/2.6.0/plugin-comms-twosix-cpp


## twoSixIndirectCpp
__Deployment Create Argument:__ 
```
--comms-channel=twoSixIndirectCpp \
--comms-kit=tag=2.6.0-v1,org=tst-race,repo=race-core,asset=plugin-comms-twosix-cpp.tar.gz
```

__Source Code Repository:__ https://github.com/tst-race/https://github.com/tst-race/race-core/tree/2.6.0/plugin-comms-twosix-cpp


## Obfs
__Deployment Create Argument:__ 
```
--comms-channel=obfs \
--comms-kit=tag=2.6.0-v1,org=tst-race,repo=race-obfs
```


__Source Code Repository:__ https://github.com/tst-race/race-obfs


## Snowflake
__Deployment Create Argument:__ 
```
--comms-channel=snowflake \ 
--comms-kit=tag=2.6.0-v1,org=tst-race,repo=race-snowflake
```


__Source Code Repository:__ https://github.com/tst-race/race-snowflake


## Raven
__Deployment Create Argument:__ 
```
--comms-channel=raven \ 
--comms-kit=tag=2.6.0-v1,org=tst-race,repo=race-raven
```


__Source Code Repository:__ https://github.com/tst-race/race-raven


## ssEmail
__Deployment Create Argument:__ 
```
--comms-channel=ssEmail \
--comms-kit=tag=2.6.0-v1,org=tst-race,repo=race-semanticsteg
```


__Source Code Repository:__ https://github.com/tst-race/race-semanticsteg

## ssRedis
__Deployment Create Argument:__ 
```
--comms-channel=ssRedis \
--comms-kit=tag=2.6.0-v1,org=tst-race,repo=race-semanticsteg
```


__Source Code Repository:__ https://github.com/tst-race/race-semanticsteg

## destiniPixelfed
__Deployment Create Argument:__ 
```
--comms-channel=destiniPixelfed \
--comms-kit=tag=2.6.0-v1,org=tst-race,repo=race-destini,asset=race-destini-pixelfed.tar.gz
```


__Source Code Repository:__ https://github.com/tst-race/race-destini

## destiniAvideo
__Deployment Create Argument:__ 

```

--comms-channel=destiniAvideo \
--comms-kit=tag=2.6.0-v1,org=tst-race,repo=race-destini,asset=race-destini-avideo.tar.gz

```


__Source Code Repository:__ https://github.com/tst-race/race-destini

## destiniDash
__Deployment Create Argument:__ 
```
--comms-channel=destiniDash \
--comms-kit=tag=2.6.0-v1,org=tst-race,repo=race-destini,asset=race-destini-dash.tar.gz
```


__Source Code Repository:__ https://github.com/tst-race/race-destini


## Butkus
__Deployment Create Argument:__ 

```
--comms-channel=butkusLocalRedis --comms-kit=tag=2.6.0-v1,org=tst-race,repo=race-butkus \ 
--comms-kit=tag=2.6.0-v1,org=tst-race,repo=race-core,asset=plugin-comms-twosix-decomposed-cpp.tar.gz
```

__Source Code Repository:__ https://github.com/tst-race/race-butkus

