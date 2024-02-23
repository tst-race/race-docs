# What is RACE: The Longer Story

RACE is a distributed system developed to provide resilient, secure, anonymous messaging. You can think of RACE in terms of a network two types of nodes: _clients_ and _servers_. The _clients_ are devices run by individual users who want to anonymously message one another; the _servers_ are run by volunteer users or organizations that provide the _infrastructure_ to enable anonymous client messaging. In a typical messaging app (e.g. [Signal](https://signal.org)) clients connect to a single conceptual "server" (typically load-balanced) which directly routes messages to other clients. This means the server knows the _to_ and _from_ of every message (its _metadata_), even if use of end-to-end encryption prevents seeing the actual message content. In contrast, RACE has an entire network of independent servers that use special multi-party computation (MPC) algorithms to route messages without individual servers learning the metadata. The specific goals of the original RACE program were to enable up to 20% of the servers to be malicious and colluding (i.e. a single adversary running a bloc of servers) without any client messaging metadata being leaked. There are multiple ways to achieve this, and there are actually two distinctly different designs of message routing implemented: [Prism](https://github.com/tst-race/race-prism) and [Carma](https://github.com/tst-race/race-carma).

![Two diagrams, the right shows two signal clients messaging via a signal server with the message to and from fields in plaintext and a caption that reads: Centralized Messaging Architecture: single point of failure to deanonymize or disrupt messaging. The left diagram shows two RACE clients messaging via a cloud of RACE servers, with the to-field encrypted when the client sends it, and the from-field encrypted when the other client receives it, and a caption that reads: RACE Messaging Architecture: distributed server network provides resilience and provides metadata anonymity via cryptographic routing.](https://github.com/tst-race/race-docs/blob/main/comparative.png?raw=true)


RACE is designed to not just operate in permissive network environments (i.e. places that also allow free access to services like [Tor](https://torproject.org) and [Signal](https://signal.org)) but also censored network environments, such as the residential networks of many repressive regimes. The anonymous routing described above provides one layer of protection by preventing an adversary from deanonymizing users by just compromising (or maliciously operating) a small number of RACE servers. However, a censoring network could still easily fingerprint and block RACE traffic if it used standard TCP connections or if the IP address of its servers was known (as happens with Tor). Thus, RACE has a layer of resilient network communication channels to enable it to operate _wholly within_ and/or span _across_ a censoring network. These channels are built to be impossible or impractical for an adversary to detect and block connections between RACE nodes. See [the RACE channels page](https://github.com/tst-race/race-docs/blob/main/race-channels.md) for more details about the channels.

In addition to resilience and anonymity, RACE was designed and tested to _scale_ to thousands of clients to enable a significant userbase built on volunteer server infrastructure. The two anonymous routing protocols use different network topologies, but both have been tested with up to 5K clients and 500 servers. Both approaches have also built in resilience mechanisms to handle individual servers or entire chunks of the server network being taken offline, which could occur either because of the volunteer nature of the servers or due to adversarial action.


## Software Stack
RACE was built as a modular plugin-based application for linux and Android environments. For ease of testing (see [RACE Quickstart](https://github.com/tst-race/race-quickstart/)) RACE servers and linux-clients are packaged as dockerized applications with all dependencies pre-installed. For the Android client we have a RACE APK that can be manually installed and is built on Android API-level 29 and currently tested on Android 10 and 11 devices. More information can be found in the [developer docs](https://github.com/tst-race/race-docs/blob/main/RACE%20developer%20guide.md).


## Architecture
RACE is built using a plugin-based architecture. There are three "arms" of RACE functionality provided by different types of _plugins_: 
1. Network-manager plugins manage the overlay network topology and handle routing messages between nodes (these are the anonymous routing algorithms [Carma](https://github.com/tst-race/race-carma) and [Prism](https://github.com/tst-race/race-prism))
2. Comms plugins provide the resilient network channels used to keep RACE undetectable and/or unblockable in hostile network environments (see the [RACE channels page](https://github.com/tst-race/race-docs/blob/main/race-channels.md))
3. The App provides an external interface for RACE, e.g. client-to-client messaging or alternate uses (see #additional-uses)

![A diagram showing the Race SDK / Core connected via APIs to Network Manager Plugins, Comms Plugins, and a Client App](https://github.com/tst-race/race-docs/blob/main/client-node-apis.png?raw=true)


## Alternative Contexts and Uses
RACE's initial goal was to provide private user-to-user messaging of a few kilobytes with a few minutes latency in a manner that could be scaled to thousands of users, operate on a network of volunteer servers, and exist entirely within a censored network environment. However, there are many variations on that goal that the RACE software is flexible enough to readily adapt to. E.g. rather than providing natural language messaging, the RACE app could provide a proxy interface to change arbitrary proxy-supporting applications into anonymously served applications. Separately, a global user and volunteer base could result in a RACE network that mixes censored and permissive network environments, where clients and servers in the latter can use standard network protocols (e.g. simple TCP sockets) for communications with greater performance. The plugin-based architecture of RACE is designed to maximize its flexiblity and minimize the effort necessary to adapt and refine its use for specific cases.

## List of RACE Code Repositories
Owing to RACE's modular nature, many of its pieces were developed by separate teams and are alternative/complementary implementations of functionality (e.g. different communication channels, different routing algorithms). This is a list of the other RACE source code repositories for these subprojects:

* [RACE Core](https://github.com/tst-race/race-core) (SDK and functional example implementations of each plugin in each supported language)
* Network Manager Plugins Implementations
  + [Carma](https://github.com/tst-race/race-carma)
  + [Prism](https://github.com/tst-race/race-prism) 
* Comms Plugin Implementations (also see [RACE Channels](https://github.com/tst-race/race-docs/blob/main/race-channels.md)
  + [Destini](https://github.com/tst-race/race-destini)
  + [Obfs](https://github.com/tst-race/race-obfs) 
  + [Snowflake](https://github.com/tst-race/race-snowflake) 
  + [Raven](https://github.com/tst-race/race-raven) 
  + [SemanticSteg](https://github.com/tst-race/race-semanticsteg) 
  + [Butkus](https://github.com/tst-race/race-butkus) 
* Artifact Manager Plugin Implementations
  + [Slothy](https://github.com/tst-race/race-slothy)
* [Race-in-the-Box](https://github.com/tst-race/race-in-the-box)
* Docker Images and Dependency Building
  + [Docker Images](https://github.com/tst-race/race-images)
  + [External Dependencies](https://github.com/orgs/tst-race/repositories?q=ext&type=all&language=&sort=)
