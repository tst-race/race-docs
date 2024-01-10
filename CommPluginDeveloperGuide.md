# Table of Contents
- [Table of Contents](#table-of-contents)
- [Comm Plugin overview](#comm-plugin-overview)
  - [What is the purpose of a comm plugin?](#what-is-the-purpose-of-a-comm-plugin)
  - [Definitions](#definitions)
  - [The difference between unified and decomposed](#the-difference-between-unified-and-decomposed)
  - [The decomposed interface](#the-decomposed-interface)
  - [How plugins are loaded](#how-plugins-are-loaded)
- [Comm plugin directory outline](#comm-plugin-directory-outline)
  - [The kit directory](#the-kit-directory)
- [Creating a unified comm plugin repo](#creating-a-unified-comm-plugin-repo)
  - [Create an empty initial directory](#create-an-empty-initial-directory)
  - [Create a source directory](#create-a-source-directory)
  - [Add a stub plugin file](#add-a-stub-plugin-file)
  - [Add a manifest.json](#add-a-manifestjson)
  - [Add cmake files](#add-cmake-files)
  - [Add the kit directories](#add-the-kit-directories)
  - [Add generate\_configs.sh](#add-generate_configssh)
  - [Add generate\_configs.py](#add-generate_configspy)
  - [Add channel\_properties.json](#add-channel_propertiesjson)
  - [Building the plugin](#building-the-plugin)
- [Implementing a unified comm plugin](#implementing-a-unified-comm-plugin)
  - [General architecture](#general-architecture)
    - [Threading](#threading)
    - [Callbacks](#callbacks)
    - [Plugin, Channels, Links, Connections, Packages](#plugin-channels-links-connections-packages)
  - [Channel / Link properties](#channel--link-properties)
  - [The comm core interface](#the-comm-core-interface)
  - [The comm plugin interface](#the-comm-plugin-interface)
  - [Implementing the manifest.json file](#implementing-the-manifestjson-file)
- [Creating a decomposed comm plugin repo](#creating-a-decomposed-comm-plugin-repo)
  - [Create an empty initial directory](#create-an-empty-initial-directory-1)
  - [Create a source directory](#create-a-source-directory-1)
  - [Add a stub encoding](#add-a-stub-encoding)
  - [Add a stub transport](#add-a-stub-transport)
  - [Add a stub user model](#add-a-stub-user-model)
  - [Add a manifest.json](#add-a-manifestjson-1)
  - [Add cmake files](#add-cmake-files-1)
  - [Add the kit directories](#add-the-kit-directories-1)
  - [Add generate\_configs.sh](#add-generate_configssh-1)
  - [Add generate\_configs.py](#add-generate_configspy-1)
  - [Add channel\_properties.json](#add-channel_propertiesjson-1)
  - [Building the plugin](#building-the-plugin-1)
- [Implementing a decomposed encoding](#implementing-a-decomposed-encoding)
  - [The Encoding SDK Interface](#the-encoding-sdk-interface)
  - [The Encoding Component Interface](#the-encoding-component-interface)
- [Implementing a decomposed transport](#implementing-a-decomposed-transport)
  - [The Transport SDK Interface](#the-transport-sdk-interface)
  - [The Transport Component Interface](#the-transport-component-interface)
- [Implementing a decomposed usermodel](#implementing-a-decomposed-usermodel)
  - [The User Model SDK Interface](#the-user-model-sdk-interface)
  - [The User Model Component Interface](#the-user-model-component-interface)
- [The manifest.json file](#the-manifestjson-file)
  - [Plugins section](#plugins-section)
  - [Compositions section](#compositions-section)
  - [Channel Properties section](#channel-properties-section)
  - [Channel Parameters section](#channel-parameters-section)
- [Implementing channel configuration](#implementing-channel-configuration)
  - [Arguments](#arguments)
  - [Outputs](#outputs)
    - [genesis-link-addresses.json](#genesis-link-addressesjson)
    - [fulfilled-ta1-request.json](#fulfilled-ta1-requestjson)
    - [user-responses.json](#user-responsesjson)
- [Adding a dependency (via RUNPATH)](#adding-a-dependency-via-runpath)
- [Valid Common User Input](#valid-common-user-input)
- [Running a custom plugin](#running-a-custom-plugin)
- [Debugging](#debugging)
- [Adapting these instructions for other programming languages](#adapting-these-instructions-for-other-programming-languages)


# Comm Plugin overview

## What is the purpose of a comm plugin?

Comms plugins provide one or more channels for covertly communicating. Each channel represents a different method of performing covert communication. Generally it is recommended that each Comms plugin provide a single channel for simplicity, but multiple channels can be provided by a single plugin if there are runtime resource advantages to doing so (e.g. sharing a single instance of an ML model). Comms plugins may be either direct connections that are obfuscated in some way (e.g. they look like a typical http session) or indirect (e.g. posting to a social media hashtag that is being scraped by another node).

## Definitions

See the [race developer guide](./RACE%20developer%20guide.md#Terminology) for definitions. The definitions for Plugins, Channels, Links, and Connections are relevant here.

## The difference between unified and decomposed

There are two possible apis that may be implemented to create a comm plugin. The unified interface is more flexible, but requires more work to implement. The decomposed interface is easier to implement and can potentially be reused by other channels. Much of the logic for handling things such as fragmentation and batching are handled automatically when using the decomposed interface, but must be handled individually when using the unified interface.

The unified interface must also handle rate limiting sending or receiving to blend in, limit the amount of queued data to prevent excess memory usage and buffer bloat, and handle multiplexing connections.

## The decomposed interface

The decomposed interface consists of three separate interfaces: the encoding interface, the transport interface, and the usermodel interface. An object that implements one of the interfaces is called a component. There is a corresponding type of component for each of the three interfaces. A decomposed channel (i.e. a channel created using the decomposed interface) consists of at least one of each of the types of components, although multiple encoding components are allowed. There is a component manager that utilizes the components and handles additional logic such as scheduling what bytes are encoded and when.

An encoding component is responsible for encoding and decoding the bytes sent. The specific encoding component provided will determine how the bytes are encoded/decoded. Some examples are no-op, base64, jpg, or video.

A transport component is responsible for all network related functions. This includes sending bytes over the network, but depending on the implementation may also include stuff such as generating background traffic to legitimate sites, logging into an account, or scraping a web page where another node may have posted data for this node.

A User model component is responsible for scheduling when transport actions occur. The actions the user model creates are most commonly fetching and posting, but may also include other actions if necessary. One concept related to the need for user models is 'behavioral independence'. This is the idea that the behavior of a transport is statistically independent from the behavior of the user that is sending messages.

## How plugins are loaded

Kits are required to have two parts. One is a manifest.json file that declares what plugins to load, and the second is the plugins themselves. The plugins may be either a shared library if they are written in a compiled language, or a python file if they are in python. The rest of this document will assume code is being written in c++. See [Adapting these instructions for other programming languages](#adapting-these-instructions-for-other-programming-languages) to see how things change for python or other compiled languages.

The manifest.json for a kit will list what plugins are supplied by a kit, what channels and components they contain, what channels can be created the components supplied by a kit, as well as additional info about what parameters and properties each channel needs.

Shared libraries are loaded and symbols `createPluginTa2()` and `destroyPluginTa2()` (in the unified case) or `create*` and `destroy*` (in the decomposed case, where * is the type of component) looked up. These symbols are called to create c++ objects that implement the corresponding interface.

# Comm plugin directory outline
```
example-comm-plugin
├── build_artifacts_in_docker_image.sh
├── build_artifacts.sh
├── clean_artifacts.sh
├── CMakeLists.txt
├── CMakePresets.json
├── *kit*
├── LICENSE
├── *source*
└── *test*
```

This is an example of a plugin directory. The only required part is the kit directory, but the others all serve common purposes. 

## The kit directory

The kit directory contains the necessary artifacts to use in a running deployment. These do not need to be committed to the repo (although parts may be), but is what should be produced by the build system. This includes the shared libraries that contain the plugin code itself, but also include config generation code and scripts to start and stop external services. The following is the kit directory of the decomposed comm plugin exemplar. In the case of the exemplar, kit/channels/* is committed to the build system, while artifacts/* is not.

```
kit
├── artifacts
│   ├── android-arm64-v8a-client
│   │   └── PluginTa2TwoSixStubDecomposed
│   │       ├── libPluginTa2TwoSixStubEncoding.so
│   │       ├── libPluginTa2TwoSixStubTransport.so
│   │       ├── libPluginTa2TwoSixStubUserModel.so
│   │       └── manifest.json
│   ├── android-x86_64-client
│   │   └── PluginTa2TwoSixStubDecomposed
│   │       ├── actions.json
│   │       ├── libPluginTa2TwoSixStubEncoding.so
│   │       ├── libPluginTa2TwoSixStubTransport.so
│   │       ├── libPluginTa2TwoSixStubUserModelFile.so
│   │       ├── libPluginTa2TwoSixStubUserModel.so
│   │       └── manifest.json
│   ├── linux-x86_64-client
│   │   └── PluginTa2TwoSixStubDecomposed
│   │       ├── actions.json
│   │       ├── libPluginTa2TwoSixStubEncoding.so
│   │       ├── libPluginTa2TwoSixStubTransport.so
│   │       ├── libPluginTa2TwoSixStubUserModelCsv.so
│   │       ├── libPluginTa2TwoSixStubUserModelFile.so
│   │       ├── libPluginTa2TwoSixStubUserModel.so
│   │       └── manifest.json
│   └── linux-x86_64-server
│       └── PluginTa2TwoSixStubDecomposed
│           ├── actions.json
│           ├── libPluginTa2TwoSixStubEncoding.so
│           ├── libPluginTa2TwoSixStubTransport.so
│           ├── libPluginTa2TwoSixStubUserModelCsv.so
│           ├── libPluginTa2TwoSixStubUserModelFile.so
│           ├── libPluginTa2TwoSixStubUserModel.so
│           └── manifest.json
└── channels
    └── twoSixIndirectComposition
        ├── channel-dependencies.json
        ├── channel_properties.json
        ├── docker-compose.yml
        ├── generate_configs.py
        ├── generate_configs.sh
        ├── get_status_of_external_services.sh
        ├── helper_functions.sh
        ├── README.md
        ├── start_external_services.sh
        └── stop_external_services.sh
```

# Creating a unified comm plugin repo

This section will walk through creating a minimal non-functional plugin. The following section will talk through how an actual implementation should behave.

## Create an empty initial directory
`mkdir example-comm-plugin && cd example-comm-plugin`

## Create a source directory
`mkdir src && cd src`

## Add a stub plugin file
Add plugin.cpp to the source directory, containing:
```c++
#include <IRacePluginTa2.h>

class StubCommPlugin : public IRacePluginTa2 {
public:
    explicit StubCommPlugin(IRaceSdkTa2 *raceSdkIn) {}
    virtual ~StubCommPlugin() {}
    virtual PluginResponse init(const PluginConfig &pluginConfig) {
        return PLUGIN_ERROR;
    }
    virtual PluginResponse shutdown() {
        return PLUGIN_ERROR;
    }
    virtual PluginResponse sendPackage(RaceHandle handle, ConnectionID connectionId, EncPkg pkg,
                                       double timeoutTimestamp, uint64_t batchId) {
        return PLUGIN_ERROR;
    }
    virtual PluginResponse openConnection(RaceHandle handle, LinkType linkType, LinkID linkId,
                                          std::string hints, int32_t sendTimeout) {
        return PLUGIN_ERROR;
    }
    virtual PluginResponse closeConnection(RaceHandle handle, ConnectionID connectionId) {
        return PLUGIN_ERROR;
    }
    virtual PluginResponse destroyLink(RaceHandle handle, LinkID linkId) {
        return PLUGIN_ERROR;
    }
    virtual PluginResponse createLink(RaceHandle handle, std::string channelGid) {
        return PLUGIN_ERROR;
    }
    virtual PluginResponse loadLinkAddress(RaceHandle handle, std::string channelGid,
                                           std::string linkAddress) {
        return PLUGIN_ERROR;
    }
    virtual PluginResponse loadLinkAddresses(RaceHandle handle, std::string channelGid,
                                             std::vector<std::string> linkAddresses) {
        return PLUGIN_ERROR;
    }
    virtual PluginResponse createLinkFromAddress(RaceHandle handle, std::string channelGid,
                                                 std::string linkAddress) {
        return PLUGIN_ERROR;
    }
    virtual PluginResponse activateChannel(RaceHandle handle, std::string channelGid,
                                           std::string roleName) {
        return PLUGIN_ERROR;
    }
    virtual PluginResponse deactivateChannel(RaceHandle handle, std::string channelGid) {
        return PLUGIN_ERROR;
    }
    virtual PluginResponse onUserInputReceived(RaceHandle handle, bool answered,
                                               const std::string &response) {
        return PLUGIN_ERROR;
    }
    PluginResponse onUserAcknowledgementReceived(RaceHandle handle) {
        return PLUGIN_ERROR;
    }
    virtual PluginResponse flushChannel(RaceHandle handle, std::string channelGid,
                                        uint64_t batchId) {
        return PLUGIN_ERROR;
    }
};

IRacePluginTa2 *createPluginTa2(IRaceSdkTa2 *sdk) {
    return new StubCommPlugin(sdk);
}

void destroyPluginTa2(IRacePluginTa2 *plugin) {
    delete static_cast<StubCommPlugin *>(plugin);
}

const RaceVersionInfo raceVersion = RACE_VERSION;
const char *const racePluginId = "StubCommPlugin";
const char *const racePluginDescription = "StubCommPlugin exmplar for creating a minimal non-functional plugin";
```

## Add a manifest.json
Add a manifest.json file to the source directory, containing:
```json
{
    "plugins": [
        {
            "file_path": "StubCommPlugin",
            "plugin_type": "TA2",
            "file_type": "shared_library",
            "node_type": "any",
            "shared_library_path": "libStubCommPlugin.so",
            "channels": ["StubCommPlugin"]
        }
    ]
}
```

## Add cmake files
Add a CMakeLists.txt file to the top-level directory, containing:
```cmake
cmake_minimum_required(VERSION 3.20)
project(stub-comm-plugin LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_BUILD_TYPE Debug)
set(CMAKE_MODULE_PATH /opt/race/race-cmake-modules)

add_subdirectory(src)
```

Add a CMakeLists.txt file to the source directory, containing:
```cmake
add_library(StubCommPlugin SHARED plugin.cpp)

find_library(LIB_RACE_SDK_COMMON raceSdkCommon)
target_link_libraries(StubCommPlugin ${LIB_RACE_SDK_COMMON})

if(ANDROID)
    if ("${TARGET_ARCHITECTURE}" STREQUAL "ANDROID_arm64-v8a")
        set(NODE_TYPE android-arm64-v8a-client)
    else()
        set(NODE_TYPE android-x86_64-client)
    endif()
else()
    if ("${TARGET_ARCHITECTURE}" STREQUAL "LINUX_arm64-v8a")
        set(NODE_TYPE linux-arm64-v8a)
    else()
        set(NODE_TYPE linux-x86_64)
    endif()
endif()

add_custom_command(TARGET StubCommPlugin POST_BUILD
COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_CURRENT_SOURCE_DIR}/../kit/artifacts/${NODE_TYPE}-client/StubCommPlugin
COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_CURRENT_SOURCE_DIR}/../kit/artifacts/${NODE_TYPE}-server/StubCommPlugin
COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_FILE:StubCommPlugin> ${CMAKE_CURRENT_SOURCE_DIR}/../kit/artifacts/${NODE_TYPE}-client/StubCommPlugin/
COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_FILE:StubCommPlugin> ${CMAKE_CURRENT_SOURCE_DIR}/../kit/artifacts/${NODE_TYPE}-server/StubCommPlugin/
COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_SOURCE_DIR}/manifest.json ${CMAKE_CURRENT_SOURCE_DIR}/../kit/artifacts/${NODE_TYPE}-client/StubCommPlugin/
COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_SOURCE_DIR}/manifest.json ${CMAKE_CURRENT_SOURCE_DIR}/../kit/artifacts/${NODE_TYPE}-server/StubCommPlugin/
)

set_property(DIRECTORY PROPERTY ADDITIONAL_MAKE_CLEAN_FILES
${CMAKE_CURRENT_SOURCE_DIR}/../plugin/artifacts/${NODE_TYPE}/
)
```

Add CMakePresets.json to the top-level directory. These presets set correct compile flags and include paths for various architectures.
```json
{
    "version": 4,
    "include": [
        "/opt/race/race-cmake-modules/presets/linux_x86_64.json",
        "/opt/race/race-cmake-modules/presets/linux_arm64-v8a.json",
        "/opt/race/race-cmake-modules/presets/android_x86_64.json",
        "/opt/race/race-cmake-modules/presets/android_arm64-v8a.json"
    ]
}
```

## Add the kit directories
The source directory is complete. Next up is creating config gen. First, create the directories

`cd ..`

`mkdir -p kit/channels/StubCommChannel`

`cd kit/channels/StubCommChannel`

## Add generate_configs.sh
Next, add a minimal script that calls the python file that contains the actual logic for generating configs. generate_configs.sh should contain:

```bash
#!/usr/bin/env bash

set -e
BASE_DIR=$(cd $(dirname ${BASH_SOURCE[0]}) >/dev/null 2>&1 && pwd)
python3 ${BASE_DIR}/generate_configs.py "$@"
```

## Add generate_configs.py
The actual logic of generate configs is a bit more complicated:

```python
#!/usr/bin/env python3

import argparse
import json
import logging
import os
import shutil
import sys


CHANNEL_ID = "StubCommPlugin"


def main():
    cli_args = get_cli_arguments()

    with open(cli_args.range_config_file, "r") as range_config_file:
        range_config = json.load(range_config_file)

    with open(cli_args.ta1_request_file, "r") as ta1_request_file:
        ta1_request = json.load(ta1_request_file)

    if os.path.isdir(cli_args.config_dir) and cli_args.overwrite:
        shutil.rmtree(cli_args.config_dir)

    os.makedirs(cli_args.config_dir, exist_ok=True)

    generate_configs(
        range_config,
        ta1_request,
        cli_args.config_dir,
    )


def generate_configs(range_config, ta1_request, config_dir):
    (link_addresses, fulfilled_ta1_request) = generate_genesis_link_addresses(
        range_config,
        ta1_request,
    )

    with open(f"{config_dir}/genesis-link-addresses.json", "w") as link_addresses_file:
        json.dump({CHANNEL_ID: link_addresses}, link_addresses_file)

    with open(f"{config_dir}/fulfilled-ta1-request.json", "w") as request_file:
        json.dump(fulfilled_ta1_request, request_file)

    with open(f"{config_dir}/user-responses.json", "w") as user_responses_file:
        json.dump({}, user_responses_file)


def generate_genesis_link_addresses(
    range_config,
    ta1_request,
):
    link_addresses = {}
    fulfilled_ta1_request = {"links": []}

    # Loop through requested links and create links
    for link_idx, requested_link in enumerate(ta1_request["links"]):
        # Only create links for request for this channel
        if CHANNEL_ID not in requested_link["channels"]:
            continue
        requested_link["channels"] = [CHANNEL_ID]

        fulfilled_ta1_request["links"].append(requested_link)

        sender_node = requested_link["sender"]
        if sender_node not in link_addresses:
            link_addresses[sender_node] = []
        link_addresses[sender_node].append(
            {
                "role": "loader",
                "personas": requested_link["recipients"],
                "address": f"{{}}",
                "description": "link_type: send",
            }
        )

        for recipient in requested_link["recipients"]:
            if recipient not in link_addresses:
                link_addresses[recipient] = []
            link_addresses[recipient].append(
                {
                    "role": "creator",
                    "personas": [sender_node],
                    "address": f"{{}}",
                    "description": "link_type: receive",
                }
            )

    return (link_addresses, fulfilled_ta1_request)




def get_cli_arguments():
    parser = argparse.ArgumentParser(description="Generate RACE Config Files")
    required = parser.add_argument_group("Required Arguments")
    optional = parser.add_argument_group("Optional Arguments")

    # Required Arguments
    required.add_argument(
        "--range",
        dest="range_config_file",
        help="Range config of the physical network",
        required=True,
        type=str,
    )
    required.add_argument(
        "--config-dir",
        dest="config_dir",
        help="Where should configs be stored",
        required=True,
        type=str,
    )
    required.add_argument(
        "--ta1-request",
        dest="ta1_request_file",
        help="Requested links from TA1",
        required=True,
        type=str,
    )

    # Optional Arguments
    optional.add_argument(
        "--overwrite",
        dest="overwrite",
        help="Overwrite configs if they exist",
        required=False,
        default=False,
        action="store_true",
    )
    optional.add_argument(
        "--local",
        dest="local_override",
        help=(
            "Ignore range config service connectivity, utilize "
            "local configs (e.g. local hostname/port vs range services fields)"
        ),
        required=False,
        default=False,
        action="store_true",
    )

    return parser.parse_args()

if __name__ == "__main__":
    LOG_LEVEL = logging.INFO
    logging.getLogger().setLevel(LOG_LEVEL)
    logging.basicConfig(
        stream=sys.stdout,
        level=LOG_LEVEL,
        format="[generate_stub_configs] %(asctime)s.%(msecs)03d %(levelname)s %(message)s",
        datefmt="%a, %d %b %Y %H:%M:%S",
    )

    main()
```

## Add channel_properties.json
Finally, add a channel_properties.json file,

```json
{
    "bootstrap": false,
    "channelGid": "StubCommPlugin",
    "connectionType": "CT_DIRECT",
    "creatorExpected": {
      "send": {
        "bandwidth_bps": -1,
        "latency_ms": -1,
        "loss": -1.0
      },
      "receive": {
        "bandwidth_bps": 25700000,
        "latency_ms": 16,
        "loss": -1.0
      }
    },
    "description": "non-functional stub plugin",
    "duration_s": -1,
    "linkDirection": "LD_LOADER_TO_CREATOR",
    "loaderExpected": {
      "send": {
        "bandwidth_bps": 25700000,
        "latency_ms": 16,
        "loss": -1.0
      },
      "receive": {
        "bandwidth_bps": -1,
        "latency_ms": -1,
        "loss": -1.0
      }
    },
    "mtu": -1,
    "multiAddressable": false,
    "period_s": -1,
    "reliable": false,
    "isFlushable": false,
    "sendType": "ST_EPHEM_SYNC",
    "supported_hints": [],
    "transmissionType": "TT_UNICAST",
    "maxLinks": 2000,
    "creatorsPerLoader": -1,
    "loadersPerCreator": -1,
    "roles": [
      {
        "roleName": "default",
        "mechanicalTags": [],
        "behavioralTags": [],
        "linkSide": "LS_BOTH"
      }
    ],
    "maxSendsPerInterval": -1,
    "secondsPerInterval": -1,
    "intervalEndTime": 0,
    "sendsRemainingInInterval": -1
}
```

## Building the plugin

The plugin should be built in an unmodified race-sdk image. This ensures that the dependencies are limited to the ones expected to be avilable at runtime. See [Adding a dependency (via RUNPATH)](#adding-a-dependency-via-runpath) for how to include additional dependencies.

On host
```bash
docker run -it --rm -v $(pwd):/code/ -w /code ghcr.io/tst-race/race-core/race-sdk:main bash
```

Inside the race-sdk docker container (use arm64-v8a instead of x86_64 if that's your host architecture)
```bash
    cmake --preset=LINUX_x86_64
    cmake --build --preset=LINUX_x86_64
```


# Implementing a unified comm plugin
## General architecture
### Threading
All calls into the plugin are single threaded. All calls other than `init()` will happen on a single plugin thread. The plugin should create threads for other tasks if they block or need to happen asynchronously, e.g. listening on a socket, or intermittently polling a whiteboard. Calls into core may be done on any thread.

Many of the calls into the sdk take a `timeout` parameter. If this is non-zero, the call will block for the specified time (in milliseconds) if the network manager plugin has too many items in its queue. This can be used to add backpressure if the number of messages being sent is too high. The special value `RACE_BLOCKING` can be used to indicate the call should block without ever timing out. Using `RACE_BLOCKING` will generally implement the correct behavior for a comm plugin. The exemplars use that everywhere except for `shutdown()` logic.

An alternative to blocking is to check the queueUtilization field of the returned `SdkResponse` object. The field contains the proportion of the queue currently utilized. It will start blocking at 1.0.


### Callbacks
As many functions are expected to be done on separate threads, if a response is expected from one of the calls it cannot be provided in the return value. Instead, many of the calls into the plugin require a response function to later be called in response. An example of this is `activateChannel()` requires `onChannelStatusChanged()` to be called once activation has completed or failed.


### Plugin, Channels, Links, Connections, Packages
The data model ends up being very hierarchical. A plugin may have multiple channels. Each channel may have multiple links. Each link may have multiple connections. Each connection may have multiple packages sent out over it.

A Channel represents a method of communicating. For example, a very basic channel would be something like a 'TCP channel', indicating data that was sent via that channel would be transmitted point to point via tcp.

A Link is created by a channel and represents a way of using that channel to communicate with a specific destination. To go along with the 'TCP channel' example, a link would be associated with connected socket.

A Connection is an abstraction that allows multiplexing of communication over a single link. The behavior is poorly defined and many existing channels just merge all traffic coming from any of the connections on a link. Theoretically, there are hints and timeouts that are associated per connection.

A Package is a unit of communication. In contrast to TCP, communication is not done as a stream of bytes. Rather, if packages need to be fragmented on the send side, they should be reassembled before notifying the receive side that a package has been received. The channel must handle fragmenting packages if they are too large to send in one transmission.

## Channel / Link properties
See the Channel and Link properties guide for more information about those structures.

## The comm core interface

This is the interface that the comms plugin uses to communicate with the rest of the race application. The core implements this interface and the plugin is given an object that implements it in the `createPluginTa2()` call.

```c++
SdkResponse onChannelStatusChanged(RaceHandle handle, std::string channelGid, ChannelStatus status, ChannelProperties properties, int32_t timeout);
```

`onChannelStatusChanged()` must be called in response to `activateChannel()` or `deactivateChannel()`. It may also be called to indicate a channel is unavailable for use due to e.g. a website being unreachable.

---

```c++
SdkResponse onLinkStatusChanged(RaceHandle handle, LinkID linkId, LinkStatus status, LinkProperties properties, int32_t timeout);
```

`onLinkStatusChanged()` must be called in response to `createLinkFromAddress()`, `createLink()`, `loadLinkAddress()`, `loadLinkAddresses()`, `destroyLink()`. It may also be called with `LINK_DESTROYED` if the link fails and cannot be reestablished separately from those any of those calls. If the status is `LINK_DESTROYED` then properties may be the default properties object, otherwise it should be the properties object for the link.

---

```c++
SdkResponse onConnectionStatusChanged(RaceHandle handle, ConnectionID connId, ConnectionStatus status, LinkProperties properties, int32_t timeout);
```

`onConnectionStatusChanged()` must be called in response to `openConnection()`, `closeConnection()` or `shutdown()`. It may also be called with status set to `CONNECTION_CLOSED` separately from either of those two, e.g. due to the link failing. If the status is `CONNECTION_CLOSED` then properties may be the default properties object, otherwise it should be the properties object for the link corresponding to this connection.

---
```c++
SdkResponse onPackageStatusChanged(RaceHandle handle, PackageStatus status, int32_t timeout);
```
`onPackageStatusChanged()` must be called in response to `sendPackage()`. If a channel is marked as reliable, then there should be two calls. One call with the status set to `PACKAGE_SENT` when the package is sent, and one with the status set to `PACKAGE_RECEIVED` when the package is received on the other end.


---
```c++
SdkResponse updateLinkProperties(LinkID linkId, LinkProperties properties, int32_t timeout);
```
Deprecated. Use `onLinkStatusChanged` instead.

---
```c++
LinkID generateLinkId(std::string channelGid);
```
Generates a valid link id. This must be called in `createLinkFromAddress()`, `createLink()`, `loadLinkAddress()`, or `loadLinkAddresses()` to get the link id to use in the corresponding `onLinkStatusChanged()` call.

---
```c++
ConnectionID generateConnectionId(LinkID linkId);
```

Generates a valid connection id. This must be called in `openConnection()` to get the connection id to use in the corresponding `onConnectionStatusChanged()` call.

---
```c++
SdkResponse receiveEncPkg(const EncPkg &pkg, const std::vector<ConnectionID> &connIDs, int32_t timeout);
```
`receiveEncPkg()` must be called when a package is received. The received package must be byte for byte identical to a package supplied in `sendPackage()` - it cannot just be a fragment of one, or more than one combined together. The list of connection ids may be all the connection ids associated with a single link. It is not allowed to be all of the connection ids associated with all links on the channel.


---
```c++
SdkResponse requestPluginUserInput(const std::string &key, const std::string &prompt, bool cache);
```
`requestPluginUserInput()` is used to get user modifiable setting / configuration values. This is commonly called during channel activation and completion of channel activation delayed until the response is received. The return value of this function will contain a handle that matches a future `onUserInputReceived()` call into the plugin.

These calls should correspond to the `channel_parameters` section of the manifest file.

This is for plugin specific parameters.

---
```c++
SdkResponse requestCommonUserInput(const std::string &key);
```
`requestCommonUserInput()` is used to get user modifiable setting / configuration values. This is commonly called during channel activation and completion of channel activation delayed until the response is received. The return value of this function will contain a handle that matches a future `onUserInputReceived()` call into the plugin.

These calls should correspond to the `channel_parameters` section of the manifest file.

This is for some common parameters that are used by multiple plugins.

See [Valid Common User Input](#valid-common-user-input) for valid keys


---
```c++
SdkResponse displayInfoToUser(const std::string &data, RaceEnums::UserDisplayType displayType);
```
`displayInfoToUser()` displays information to the user in the specified format

--
```c++
SdkResponse displayBootstrapInfoToUser(const std::string &data, RaceEnums::UserDisplayType displayType, RaceEnums::BootstrapActionType actionType);
```
`displayBootstrapInfoToUser()` displays information to the user in the specified format. This also notifies the app of updates to the bootstrapping process. 

---
```c++
SdkResponse unblockQueue(ConnectionID connId);
```
`unblockQueue()` can be used to indicate that a connection is no longer blocked. A connection can be blocked by a return value of `PLUGIN_TEMP_ERROR` from `sendPackage()`. If a connection is blocked, no `sendPackage()` calls into the plugin will be called for that connection. Upon unblocking, the last `sendPackage()` call for that connection is retried.

---


## The comm plugin interface

This is the interface that the rest of the RACE application uses to communicate with the comm plugin. The comm plugin must implement this interface and return the implementing object in the `createPluginTa2()` call.

---
```c++
PluginResponse init(const PluginConfig &pluginConfig);
```
Initialize the plugin. The pluginConfig object will contain various platform / application specific paths.

---
```c++
PluginResponse shutdown();
```
Stop all activities for the plugin. This should cause the plugin to stop all background threads. A call to `onConnectionStatusChanged()` with status equal to `CONNECTION_CLOSED` should be issued for each open connection. A call to `onLinkStatusChanged()` with status equal to `LINK_DESTROYED` should be issued for each existing link. A call to `onChannelStatusChanged()` with status equal to `CHANNEL_DISABLED` should be issued for each channel.

---
```c++
PluginResponse activateChannel(RaceHandle handle, std::string channelGid, std::string roleName);
```
The plugin will receive an activate channel call when the channel is chosen for use. Any memory or processing intensive initialization should be done here. Any calls to `requestPluginUserInput()` or `requestCommonUserInput()` should also be done here. This call must result in a call to `onChannelStatusChanged()`, though that may happen after this call returns.

---
```c++
PluginResponse deactivateChannel(RaceHandle handle, std::string channelGid);
```
The plugin will receive an deactivate channel call when the channel is not expected to be used any more. The plugin should clean up any resources it has open. This includes any connections and links. A call to `onConnectionStatusChanged()` with status equal to `CONNECTION_CLOSED` should be issued for each open connection on this channel. A call to `onLinkStatusChanged()` with status equal to `LINK_DESTROYED` should be issued for each existing link on this channel. A call to `onChannelStatusChanged()` with status equal to `CHANNEL_DISABLED` should be issued this channel. These calls into the core should happen before this function returns.

---
```c++
PluginResponse createLink(RaceHandle handle, std::string channelGid);
PluginResponse createLinkFromAddress(RaceHandle handle, std::string channelGid, std::string linkAddress);
```
Create a new link or create a new link with the specified link address on the specified channel. This call must use `generateLinkId()` to generate a link id. `onLinkStatusChanged()` must be called with either `LINK_CREATED` if creating the link was successful, or `LINK_DESTROYED` if creating the link failed and the resulting link is unusable. The handle supplied to `onLinkStatusChanged()` must match the handle passed in to `createLink*()`.

---
```c++
PluginResponse loadLinkAddress(RaceHandle handle, std::string channelGid, std::string linkAddress);
```
Load a new link with the specified link address on the specified channel. This call must use `generateLinkId()` to generate a link id. `onLinkStatusChanged()` must be called with either `LINK_LOADED` if creating the link was successful, or `LINK_DESTROYED` if creating the link failed and the resulting link is unusable. The handle supplied to `onLinkStatusChanged()` must match the handle passed in to `loadLinkAddress()`.

---
```c++
PluginResponse loadLinkAddresses(RaceHandle handle, std::string channelGid, std::vector<std::string> linkAddresses);
```
If the channel supports loading multicast addresses, this should behave similarly to `loadLinkAddress()`. If this channel does not support multiple addresses, `onLinkStatusChanged()` must be called with the passed in handle and status set to `LINK_DESTROYED`.

---
```c++
PluginResponse destroyLink(RaceHandle handle, LinkID linkId);
```
This call should clean up all resources related to the specified link. A call to `onConnectionStatusChanged()` with status equal to `CONNECTION_CLOSED` should be issued for each open connection on this link. A call to `onLinkStatusChanged()` with status equal to `LINK_DESTROYED` must be issued for this link.

---
```c++
PluginResponse openConnection(RaceHandle handle, LinkType linkType, LinkID linkId, std::string linkHints, int32_t sendTimeout);
```
Create a new connection for the specified link. This call must use `generateConnectionId()` to generate a connection id. `onConnectionStatusChanged()` must be called with either `CONNECTION_OPEN` if creating the link was successful, or `CONNECTION_CLOSED` if creating the link failed and the resulting link is unusable. The handle supplied to `onConnectionStatusChanged()` must match the handle passed in to `openConnection()`.

`linkHints` is a json object (or empty string) containing implementation specific hints for a connection. Supported hints should be listed in channel properties.

sendTimeout is the length of time in seconds that a package on this connection is valid for. If a channel will be unavailable for longer than this amount of time `onChannelStatusChanged()` with status `CHANNEL_UNAVAILABLE` may be used to indicate that.

---
```c++
PluginResponse closeConnection(RaceHandle handle, ConnectionID connectionId);
```
This call should clean up all resources related to the specified connection. A call to `onConnectionStatusChanged()` with handle matching the supplied handle and status equal to `CONNECTION_CLOSED` must be issued for each open connection on this link.

---
```c++
PluginResponse sendPackage(RaceHandle handle, ConnectionID connectionId, EncPkg pkg, double timeoutTimestamp, uint64_t batchId);
```
Send a package over the specified connection. `onPackageStatusChanged()` must be called with handle matching the supplied handle and status equal either `PACKAGE_SENT` or `PACKAGE_FAILED_GENERIC` depending on whether the call was successful or not. If this involves blocking network calls, consider doing it on a background thread.

`timeoutTimestamp` is a timestamp indicating the package is no longer useful after a certain time. If the package cannot be sent before then, it can be dropped. `PACKAGE_FAILED_TIMEOUT` should be used in this case.

`batchId` can be used along with `flushChannel()` calls to indicated that a series of calls to `sendPackage()` has completed and packages can be sent immediately without waiting for more batching.

If the channel is marked as reliable in ChannelProperties, then `PACKAGE_RECEIVED`. With TCP, this would be once the ACK has been received.

---
```c++
PluginResponse onUserInputReceived(RaceHandle handle, bool answered, const std::string &response);
```
Receive a response to a previous `requestPluginUserInput()` or `requestCommonUserInput()` call. The handle will match the handle returned in the `SdkResponse` object of the call that this is responding to.

---
```c++
PluginResponse onUserAcknowledgementReceived(RaceHandle handle);
```
Receive a response to a previous `displayInfoToUser()` or `displayBootstrapInfoToUser()` call. The handle will match the handle returned in the `SdkResponse` object of the call that this is responding to.

---
```c++
PluginResponse createBootstrapLink(RaceHandle handle, std::string channelGid, std::string passphrase);
PluginResponse serveFiles(LinkID linkId, std::string path)
```
Optional calls to be implemented if a channel support bootstrap links.

Channels that support bootstrap links must support transferring files to a user without a race application. These are generally local channels (e.g. wifi hotspot or nfc). `createBootstrapLink()` should open a link that the new node will connect to once the app is installed. `serveFiles()` should make the file at path available to the recipient.

displayInfoToUser / displayBootstrapInfoToUSer should be used for any user interaction required.

---
```c++
PluginResponse flushChannel(RaceHandle handle, std::string channelGid, uint64_t batchId);
```
If messages are not sent out immediately by default, and instead are queued into batches for efficiency, this call may be used to indicate all messages in the batch have been supplied and that they should be sent out as soon as possible.



## Implementing the manifest.json file
The manifest.json file is the same for both unified and decomposed plugins. See [The manifest.json file](#the-manifestjson-file) for details.

# Creating a decomposed comm plugin repo
## Create an empty initial directory
`mkdir example-decomposed-comm-plugin && cd example-decomposed-comm-plugin`
## Create a source directory
`mkdir src && cd src`
## Add a stub encoding
```c++
#include <IEncodingComponent.h>

class StubEncoding : public IEncodingComponent {
public:
    ComponentStatus onUserInputReceived(RaceHandle handle, bool answered,
                                        const std::string &response) override {
        return COMPONENT_ERROR;
    }
    EncodingProperties getEncodingProperties() override {
        return {};
    }
    SpecificEncodingProperties getEncodingPropertiesForParameters(
        const EncodingParameters &params) override {
        return {};
    }
    ComponentStatus encodeBytes(RaceHandle handle, const EncodingParameters &params,
                                const std::vector<uint8_t> &bytes) override {
        return COMPONENT_ERROR;
    }
    ComponentStatus decodeBytes(RaceHandle handle, const EncodingParameters &params,
                                const std::vector<uint8_t> &bytes) override {
        return COMPONENT_ERROR;
    }
};

IEncodingComponent *createEncoding(const std::string &encoding, IEncodingSdk *sdk,
                                   const std::string &roleName, const PluginConfig &pluginConfig) {
    return new StubEncoding();
}

void destroyEncoding(IEncodingComponent *component) {
    delete component;
}

const RaceVersionInfo raceVersion = RACE_VERSION;
```
## Add a stub transport
```c++
#include <ITransportComponent.h>

class StubTransport : public ITransportComponent {
public:
    ComponentStatus onUserInputReceived(RaceHandle handle, bool answered,
                                        const std::string &response) override {
        return COMPONENT_ERROR;
    }
    TransportProperties getTransportProperties() override {
        return {};
    }
    LinkProperties getLinkProperties(const LinkID &linkId) override {
        return {};
    }
    ComponentStatus createLink(RaceHandle handle, const LinkID &linkId) override {
        return COMPONENT_ERROR;
    }
    ComponentStatus loadLinkAddress(RaceHandle handle, const LinkID &linkId,
                                    const std::string &linkAddress) override {
        return COMPONENT_ERROR;
    }
    ComponentStatus loadLinkAddresses(RaceHandle handle, const LinkID &linkId,
                                      const std::vector<std::string> &linkAddress) override {
        return COMPONENT_ERROR;
    }
    ComponentStatus createLinkFromAddress(RaceHandle handle, const LinkID &linkId,
                                          const std::string &linkAddress) override {
        return COMPONENT_ERROR;
    }
    ComponentStatus destroyLink(RaceHandle handle, const LinkID &linkId) override {
        return COMPONENT_ERROR;
    }
    std::vector<EncodingParameters> getActionParams(const Action &action) override {
        return {};
    }
    ComponentStatus enqueueContent(const EncodingParameters &params, const Action &action,
                                   const std::vector<uint8_t> &content) override {
        return COMPONENT_ERROR;
    }
    ComponentStatus dequeueContent(const Action &action) override {
        return COMPONENT_ERROR;
    }
    ComponentStatus doAction(const std::vector<RaceHandle> &handles,
                             const Action &action) override {
        return COMPONENT_ERROR;
    }
};

ITransportComponent *createTransport(const std::string &transport, ITransportSdk *sdk,
                                     const std::string &roleName,
                                     const PluginConfig &pluginConfig) {
    return new StubTransport();
}

void destroyTransport(ITransportComponent *component) {
    delete component;
}

const RaceVersionInfo raceVersion = RACE_VERSION;
```
## Add a stub user model
```c++
#include <IUserModelComponent.h>

class StubUserModel : public IUserModelComponent {
public:
    ComponentStatus onUserInputReceived(RaceHandle handle, bool answered,
                                        const std::string &response) override {
        return COMPONENT_ERROR;
    }
    UserModelProperties getUserModelProperties() override {
        return {};
    }
    ComponentStatus addLink(const LinkID &link, const LinkParameters &params) override {
        return COMPONENT_ERROR;
    }
    ComponentStatus removeLink(const LinkID &link) override {
        return COMPONENT_ERROR;
    }
    ActionTimeline getTimeline(Timestamp start, Timestamp end) override {
        return {};
    }
    ComponentStatus onTransportEvent(const Event &event) override {
        return COMPONENT_ERROR;
    }
};

IUserModelComponent *createUserModel(const std::string &usermodel, IUserModelSdk *sdk,
                                     const std::string &roleName,
                                     const PluginConfig &pluginConfig) {
    return new StubUserModel();
}
void destroyUserModel(IUserModelComponent *component) {
    delete component;
}

const RaceVersionInfo raceVersion = RACE_VERSION;
```
## Add a manifest.json
```json
{
    "plugins": [
        {
            "file_path": "StubDecomposedPlugin",
            "plugin_type": "TA2",
            "file_type": "shared_library",
            "node_type": "any",
            "shared_library_path": "libStubEncoding.so",
            "encodings": ["StubEncoding"]
        },
        {
            "file_path": "StubDecomposedPlugin",
            "plugin_type": "TA2",
            "file_type": "shared_library",
            "node_type": "any",
            "shared_library_path": "libStubTransport.so",
            "transports": ["StubTransport"]
        },
        {
            "file_path": "StubDecomposedPlugin",
            "plugin_type": "TA2",
            "file_type": "shared_library",
            "node_type": "any",
            "shared_library_path": "libStubUserModel.so",
            "usermodels": ["StubUserModel"]
        }
    ],
    "compositions": [
        {
            "id": "StubComposition",
            "transport": "StubTransport",
            "usermodel": "StubUserModel",
            "encodings": ["StubEncoding"]
        }
    ]
}

```
## Add cmake files
Add a CMakeLists.txt file to the top-level directory, containing:
```cmake
cmake_minimum_required(VERSION 3.20)
project(stub-comm-plugin LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_BUILD_TYPE Debug)
set(CMAKE_MODULE_PATH /opt/race/race-cmake-modules)

add_subdirectory(src)
```

Add a CMakeLists.txt file to the source directory, containing:
```cmake
set(CHANNEL_NAME StubDecomposedPlugin)

add_library(StubEncoding SHARED encoding.cpp)
target_link_libraries(StubEncoding raceSdkCommon)
target_include_directories(StubEncoding PRIVATE /linux/x86_64/include/)

add_library(StubTransport SHARED transport.cpp)
target_link_libraries(StubTransport raceSdkCommon)
target_include_directories(StubTransport PRIVATE /linux/x86_64/include/)

add_library(StubUserModel SHARED user-model.cpp)
target_link_libraries(StubUserModel raceSdkCommon)
target_include_directories(StubUserModel PRIVATE /linux/x86_64/include/)

if(ANDROID)
    if ("${TARGET_ARCHITECTURE}" STREQUAL "ANDROID_arm64-v8a")
        set(NODE_TYPE android-arm64-v8a-client)
    else()
        set(NODE_TYPE android-x86_64-client)
    endif()
else()
    if ("${TARGET_ARCHITECTURE}" STREQUAL "LINUX_arm64-v8a")
        set(NODE_TYPE linux-arm64-v8a)
    else()
        set(NODE_TYPE linux-x86_64)
    endif()
endif()

add_custom_command(TARGET StubEncoding POST_BUILD
COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_CURRENT_SOURCE_DIR}/../kit/artifacts/${NODE_TYPE}-client/${CHANNEL_NAME}
COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_CURRENT_SOURCE_DIR}/../kit/artifacts/${NODE_TYPE}-server/${CHANNEL_NAME}
COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_FILE:StubEncoding> ${CMAKE_CURRENT_SOURCE_DIR}/../kit/artifacts/${NODE_TYPE}-client/${CHANNEL_NAME}/
COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_FILE:StubEncoding> ${CMAKE_CURRENT_SOURCE_DIR}/../kit/artifacts/${NODE_TYPE}-server/${CHANNEL_NAME}/
COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_SOURCE_DIR}/manifest.json ${CMAKE_CURRENT_SOURCE_DIR}/../kit/artifacts/${NODE_TYPE}-client/${CHANNEL_NAME}/
COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_SOURCE_DIR}/manifest.json ${CMAKE_CURRENT_SOURCE_DIR}/../kit/artifacts/${NODE_TYPE}-server/${CHANNEL_NAME}/
)

add_custom_command(TARGET StubTransport POST_BUILD
COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_CURRENT_SOURCE_DIR}/../kit/artifacts/${NODE_TYPE}-client/${CHANNEL_NAME}
COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_CURRENT_SOURCE_DIR}/../kit/artifacts/${NODE_TYPE}-server/${CHANNEL_NAME}
COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_FILE:StubTransport> ${CMAKE_CURRENT_SOURCE_DIR}/../kit/artifacts/${NODE_TYPE}-client/${CHANNEL_NAME}/
COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_FILE:StubTransport> ${CMAKE_CURRENT_SOURCE_DIR}/../kit/artifacts/${NODE_TYPE}-server/${CHANNEL_NAME}/
COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_SOURCE_DIR}/manifest.json ${CMAKE_CURRENT_SOURCE_DIR}/../kit/artifacts/${NODE_TYPE}-client/${CHANNEL_NAME}/
COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_SOURCE_DIR}/manifest.json ${CMAKE_CURRENT_SOURCE_DIR}/../kit/artifacts/${NODE_TYPE}-server/${CHANNEL_NAME}/
)

add_custom_command(TARGET StubUserModel POST_BUILD
COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_CURRENT_SOURCE_DIR}/../kit/artifacts/${NODE_TYPE}-client/${CHANNEL_NAME}
COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_CURRENT_SOURCE_DIR}/../kit/artifacts/${NODE_TYPE}-server/${CHANNEL_NAME}
COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_FILE:StubUserModel> ${CMAKE_CURRENT_SOURCE_DIR}/../kit/artifacts/${NODE_TYPE}-client/${CHANNEL_NAME}/
COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_FILE:StubUserModel> ${CMAKE_CURRENT_SOURCE_DIR}/../kit/artifacts/${NODE_TYPE}-server/${CHANNEL_NAME}/
COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_SOURCE_DIR}/manifest.json ${CMAKE_CURRENT_SOURCE_DIR}/../kit/artifacts/${NODE_TYPE}-client/${CHANNEL_NAME}/
COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_SOURCE_DIR}/manifest.json ${CMAKE_CURRENT_SOURCE_DIR}/../kit/artifacts/${NODE_TYPE}-server/${CHANNEL_NAME}/
)

set_property(DIRECTORY PROPERTY ADDITIONAL_MAKE_CLEAN_FILES
${CMAKE_CURRENT_SOURCE_DIR}/../plugin/artifacts/${NODE_TYPE}/
)
```

Add CMakePresets.json to the top-level directory. These presets set correct compile flags and include paths for various architectures.
```json
{
    "version": 4,
    "include": [
        "/opt/race/race-cmake-modules/presets/linux_x86_64.json",
        "/opt/race/race-cmake-modules/presets/linux_arm64-v8a.json",
        "/opt/race/race-cmake-modules/presets/android_x86_64.json",
        "/opt/race/race-cmake-modules/presets/android_arm64-v8a.json"
    ]
}
```

## Add the kit directories
The source directory is complete. Next up is creating config gen. First, create the directories

`cd ..`

`mkdir kit`

`mkdir kit/channels`

`mkdir kit/channels/StubCommChannel`

`cd kit/channels/StubCommChannel`

## Add generate_configs.sh
Next, add a minimal script that calls the python file that contains the actual logic for generating configs. generate_configs.sh should contain:

```bash
#!/usr/bin/env bash

set -e
BASE_DIR=$(cd $(dirname ${BASH_SOURCE[0]}) >/dev/null 2>&1 && pwd)
python3 ${BASE_DIR}/generate_configs.py "$@"
```

## Add generate_configs.py
The actual logic of generate configs is a bit more complicated:

```python
#!/usr/bin/env python3

import argparse
import json
import logging
import os
import shutil
import sys


CHANNEL_ID = "StubComposition"


def main():
    cli_args = get_cli_arguments()

    with open(cli_args.range_config_file, "r") as range_config_file:
        range_config = json.load(range_config_file)

    with open(cli_args.ta1_request_file, "r") as ta1_request_file:
        ta1_request = json.load(ta1_request_file)

    if os.path.isdir(cli_args.config_dir) and cli_args.overwrite:
        shutil.rmtree(cli_args.config_dir)

    os.makedirs(cli_args.config_dir, exist_ok=True)

    generate_configs(
        range_config,
        ta1_request,
        cli_args.config_dir,
    )


def generate_configs(range_config, ta1_request, config_dir):
    (link_addresses, fulfilled_ta1_request) = generate_genesis_link_addresses(
        range_config,
        ta1_request,
    )

    with open(f"{config_dir}/genesis-link-addresses.json", "w") as link_addresses_file:
        json.dump({CHANNEL_ID: link_addresses}, link_addresses_file)

    with open(f"{config_dir}/fulfilled-ta1-request.json", "w") as request_file:
        json.dump(fulfilled_ta1_request, request_file)

    with open(f"{config_dir}/user-responses.json", "w") as user_responses_file:
        json.dump({}, user_responses_file)


def generate_genesis_link_addresses(
    range_config,
    ta1_request,
):
    link_addresses = {}
    fulfilled_ta1_request = {"links": []}

    # Loop through requested links and create links
    for link_idx, requested_link in enumerate(ta1_request["links"]):
        # Only create links for request for this channel
        if CHANNEL_ID not in requested_link["channels"]:
            continue
        requested_link["channels"] = [CHANNEL_ID]

        fulfilled_ta1_request["links"].append(requested_link)

        sender_node = requested_link["sender"]
        if sender_node not in link_addresses:
            link_addresses[sender_node] = []
        link_addresses[sender_node].append(
            {
                "role": "loader",
                "personas": requested_link["recipients"],
                "address": f"{{}}",
                "description": "link_type: send",
            }
        )

        for recipient in requested_link["recipients"]:
            if recipient not in link_addresses:
                link_addresses[recipient] = []
            link_addresses[recipient].append(
                {
                    "role": "creator",
                    "personas": [sender_node],
                    "address": f"{{}}",
                    "description": "link_type: receive",
                }
            )

    return (link_addresses, fulfilled_ta1_request)




def get_cli_arguments():
    parser = argparse.ArgumentParser(description="Generate RACE Config Files")
    required = parser.add_argument_group("Required Arguments")
    optional = parser.add_argument_group("Optional Arguments")

    # Required Arguments
    required.add_argument(
        "--range",
        dest="range_config_file",
        help="Range config of the physical network",
        required=True,
        type=str,
    )
    required.add_argument(
        "--config-dir",
        dest="config_dir",
        help="Where should configs be stored",
        required=True,
        type=str,
    )
    required.add_argument(
        "--ta1-request",
        dest="ta1_request_file",
        help="Requested links from TA1",
        required=True,
        type=str,
    )

    # Optional Arguments
    optional.add_argument(
        "--overwrite",
        dest="overwrite",
        help="Overwrite configs if they exist",
        required=False,
        default=False,
        action="store_true",
    )
    optional.add_argument(
        "--local",
        dest="local_override",
        help=(
            "Ignore range config service connectivity, utilize "
            "local configs (e.g. local hostname/port vs range services fields)"
        ),
        required=False,
        default=False,
        action="store_true",
    )

    return parser.parse_args()

if __name__ == "__main__":
    LOG_LEVEL = logging.INFO
    logging.getLogger().setLevel(LOG_LEVEL)
    logging.basicConfig(
        stream=sys.stdout,
        level=LOG_LEVEL,
        format="[generate_stub_configs] %(asctime)s.%(msecs)03d %(levelname)s %(message)s",
        datefmt="%a, %d %b %Y %H:%M:%S",
    )

    main()
```

## Add channel_properties.json
Finally, add a channel_properties.json file,

```json
{
    "bootstrap": false,
    "channelGid": "StubComposition",
    "connectionType": "CT_DIRECT",
    "creatorExpected": {
      "send": {
        "bandwidth_bps": -1,
        "latency_ms": -1,
        "loss": -1.0
      },
      "receive": {
        "bandwidth_bps": 25700000,
        "latency_ms": 16,
        "loss": -1.0
      }
    },
    "description": "non-functional stub plugin",
    "duration_s": -1,
    "linkDirection": "LD_LOADER_TO_CREATOR",
    "loaderExpected": {
      "send": {
        "bandwidth_bps": 25700000,
        "latency_ms": 16,
        "loss": -1.0
      },
      "receive": {
        "bandwidth_bps": -1,
        "latency_ms": -1,
        "loss": -1.0
      }
    },
    "mtu": -1,
    "multiAddressable": false,
    "period_s": -1,
    "reliable": false,
    "isFlushable": false,
    "sendType": "ST_EPHEM_SYNC",
    "supported_hints": [],
    "transmissionType": "TT_UNICAST",
    "maxLinks": 2000,
    "creatorsPerLoader": -1,
    "loadersPerCreator": -1,
    "roles": [
      {
        "roleName": "default",
        "mechanicalTags": [],
        "behavioralTags": [],
        "linkSide": "LS_BOTH"
      }
    ],
    "maxSendsPerInterval": -1,
    "secondsPerInterval": -1,
    "intervalEndTime": 0,
    "sendsRemainingInInterval": -1
}
```
## Building the plugin

The components should be built in an unmodified race-sdk image. This ensures that the dependencies are limited to the ones expected to be avilable at runtime. See [Adding a dependency (via RUNPATH)](#adding-a-dependency-via-runpath) for how to include additional dependencies.

On host
```bash
docker run -it --rm -v $(pwd):/code/ -w /code ghcr.io/tst-race/race-core/race-sdk:main bash
```

Inside the race-sdk docker container (use arm64-v8a instead of x86_64 if that's your host architecture)
```bash
    cmake --preset=LINUX_x86_64
    cmake --build --preset=LINUX_x86_64
```


# Implementing a decomposed encoding

## The Encoding SDK Interface
This is the interface that the encoding uses to communicate with the rest of the race application. The core implements this interface and the encoding is given an object that implements it in the `createPluginEncoding()` call.

---
```c++
ChannelResponse updateState(ComponentState state);
```
Inform the component manager of a change in state.

NOTE: This MUST be called with the state `COMPONENT_STARTED` before the component can be used. This should be done in either the constructor or `onUserInputReceived()` if the component requires user input before starting.

---
```c++
ChannelResponse requestPluginUserInput(const std::string &key, const std::string &prompt, bool cache);
```
`requestPluginUserInput()` is used to get user modifiable setting / configuration values. This is commonly called during component activation and completion of component activation delayed until the response is received. The return value of this function will contain a handle that matches a future `onUserInputReceived()` call into the plugin.

These calls should correspond to the `channel_parameters` section of the manifest file.

This is for component specific parameters.

---
```c++
ChannelResponse requestCommonUserInput(const std::string &key);
```
`requestCommonUserInput()` is used to get user modifiable setting / configuration values. This is commonly called during component activation and completion of component activation delayed until the response is received. The return value of this function will contain a handle that matches a future `onUserInputReceived()` call into the plugin.

These calls should correspond to the `channel_parameters` section of the manifest file.

This is for some common parameters that are used by multiple plugins or components.

See [Valid Common User Input](#valid-common-user-input) for valid keys.

---
```c++
ChannelResponse onBytesEncoded(RaceHandle handle, const std::vector<uint8_t> &bytes, EncodingStatus status);
```
Tell the component manager that encoding has completed. This must be called in response to a `encodeBytes()` call. Status may be either `ENCODE_OK`, indicating the encoding succeeded, or `ENCODE_FAILED`, indicating that encoding was not successful. `handle` must match the handle supplied to the corresponding `encodeBytes()` call. `bytes` must contain the encoded bytes if status is `ENCODE_OK`.

---
```c++
ChannelResponse onBytesDecoded(RaceHandle handle, const std::vector<uint8_t> &bytes, EncodingStatus status);
```
Tell the component manager that decoding has completed. This must be called in response to a `decodeBytes()` call. Status may be either `ENCODE_OK` (yes, it's 'encode' even though this is decoding), indicating the decoding succeeded, or `ENCODE_FAILED`, indicating that decoding was not successful. `handle` must match the handle supplied to the corresponding `decodeBytes()` call. `bytes` must contain the decoded bytes if status is `ENCODE_OK`.

---


## The Encoding Component Interface

This is the interface that the rest of the RACE application uses to communicate with the user model. The user model must implement this interface and return the implementing object in the `createUserModel()` call.

---
```c++
ComponentStatus onUserInputReceived(RaceHandle handle, bool answered, const std::string &response);
```
Receive a response to a previous `requestPluginUserInput()` or `requestCommonUserInput()` call. The handle will match the handle returned in the `SdkResponse` object of the call that this is responding to.


---
```c++
EncodingProperties getEncodingProperties();
```
The component manager will call this to get some properties of the encoder. The properties contain the time taken to encode, and what type of encoding it is.

The time taken to encode should be a pessimistic estimate. It is used to calculate how long before an action is taken will the encoding start. The units are seconds and may be fractional.

The type of encoding should be a mime-type string. Examples include: "text/plain", "application/octet-stream", and "image/png".


---
```c++
SpecificEncodingProperties getEncodingPropertiesForParameters(const EncodingParameters &params);
```
The component manager will call this to get some properties of the encoder given some parameters. The encoding parameters may contain encoder specific information if the transport is compatible. The properties returned should contain the number of bytes that can be encoded into the content that will be returned for the supplied parameters.


---
```c++
ComponentStatus encodeBytes(RaceHandle handle, const EncodingParameters &params, const std::vector<uint8_t> &bytes);
```
Encode bytes into a message. `onBytesEncoded()` should be called once the encoding is complete.


---
```c++
ComponentStatus decodeBytes(RaceHandle handle, const EncodingParameters &params, const std::vector<uint8_t> &bytes);
```
Decode bytes from a message. `onBytesDecoded()` should be called once the decoding is complete.

---
# Implementing a decomposed transport

## The Transport SDK Interface
This is the interface that the transport uses to communicate with the rest of the race application. The core implements this interface and the transport is given an object that implements it in the `createPluginTransport()` call.

---
```c++
ChannelResponse updateState(ComponentState state);
```
Inform the component manager of a change in state.

NOTE: This MUST be called with the state `COMPONENT_STARTED` before the component can be used. This should be done in either the constructor or `onUserInputReceived()` if the component requires user input before starting.


---
```c++
ChannelResponse requestPluginUserInput(const std::string &key, const std::string &prompt, bool cache);
```
`requestPluginUserInput()` is used to get user modifiable setting / configuration values. This is commonly called during component activation and completion of component activation delayed until the response is received. The return value of this function will contain a handle that matches a future `onUserInputReceived()` call into the plugin.

These calls should correspond to the `channel_parameters` section of the manifest file.

This is for component specific parameters.

---
```c++
ChannelResponse requestCommonUserInput(const std::string &key);
```
`requestCommonUserInput()` is used to get user modifiable setting / configuration values. This is commonly called during component activation and completion of component activation delayed until the response is received. The return value of this function will contain a handle that matches a future `onUserInputReceived()` call into the plugin.

These calls should correspond to the `channel_parameters` section of the manifest file.

This is for some common parameters that are used by multiple plugins or components.

See [Valid Common User Input](#valid-common-user-input) for valid keys.

---
```c++
ChannelProperties getChannelProperties();
```
Get the channel properties listed in the channel properties json.

See [channel properties documentation](./ChannelAndLinkProperties.md) for details about channel properties.

---
```c++
ChannelResponse onLinkStatusChanged(RaceHandle handle, const LinkID &linkId, LinkStatus status, const LinkParameters &params);
```
Update the status of a link. `onLinkStatusChanged()` must be called in response to `createLinkFromAddress()`, `createLink()`, `loadLinkAddress()`, `loadLinkAddresses()`, or `destroyLink()`. It may also be called with `LINK_DESTROYED` if the link fails and cannot be reestablished separately from those any of those calls.

If the call is a response to a previous call, the handle should be match the handle supplied in the original call, otherwise it should be `NULL_RACE_HANDLE`.

`params` may be used when creating or loading to communicate information about this link to the user model.

---
```c++
ChannelResponse onPackageStatusChanged(RaceHandle handle, PackageStatus status);
```
`onPackageStatusChanged()` must be called in response to `doAction()`. If the package was successfully sent, the status should be `PACKAGE_SENT`. Otherwise, `PACKAGE_FAILED_GENERIC` must be called.

The component manager currently does not support `PACKAGE_RECEIVED`.

---
```c++
ChannelResponse onEvent(const Event &event);
```
Inform the user model of some implementation defined event that occurred.

---
```c++
ChannelResponse onReceive(const LinkID &linkId, const EncodingParameters &params, const std::vector<uint8_t> &bytes);
```
`onReceive()` must be called when a package is received on the specified link. The received package must be byte for byte identical to a package supplied in `enqueueContent()`.

Fields of the `params` object other than link id must also match the properties supplied in `enqueueContent()`. This should be handled by the user model informing the transport of any relevant properties.

---

## The Transport Component Interface

This is the interface that the rest of the RACE application uses to communicate with the transport. The transport must implement this interface and return the implementing object in the `createUserModel()` call.

---
```c++
ComponentStatus onUserInputReceived(RaceHandle handle, bool answered, const std::string &response);
```
Receive a response to a previous `requestPluginUserInput()` or `requestCommonUserInput()` call. The handle will match the handle returned in the `SdkResponse` object of the call that this is responding to.

---
```c++
TransportProperties getTransportProperties();
```
The transport properties must contain all supported actions and what type of encodings are necessary for each action.

---
```c++
LinkProperties getLinkProperties(const LinkID &linkId);
```

Returns the link properties for the specified link.

See the Channel and link properties documentation for more details about link properties.


---
```c++
ComponentStatus createLink(RaceHandle handle, const LinkID &linkId);
```
Create a new link. `onLinkStatusChanged()` must be called with either `LINK_CREATED` if creating the link was successful, or `LINK_DESTROYED` if creating the link failed and the resulting link is unusable. The handle supplied to `onLinkStatusChanged()` must match the handle passed in to `createLink()`.

---
```c++
ComponentStatus createLinkFromAddress(RaceHandle handle, const LinkID &linkId, const std::string &linkAddress);
```
Create a new link with the specified link address. `onLinkStatusChanged()` must be called with either `LINK_CREATED` if creating the link was successful, or `LINK_DESTROYED` if creating the link failed and the resulting link is unusable. The handle supplied to `onLinkStatusChanged()` must match the handle passed in to `createLinkFromAddress()`.

---
```c++
ComponentStatus loadLinkAddress(RaceHandle handle, const LinkID &linkId, const std::string &linkAddress);
```
Load a new link with the specified link address. `onLinkStatusChanged()` must be called with either `LINK_LOADED` if creating the link was successful, or `LINK_DESTROYED` if creating the link failed and the resulting link is unusable. The handle supplied to `onLinkStatusChanged()` must match the handle passed in to `loadLinkAddress()`.

---
```c++
ComponentStatus loadLinkAddresses(RaceHandle handle, const LinkID &linkId, const std::vector<std::string> &linkAddress);
```
If the channel supports loading multicast addresses, this should behave similarly to `loadLinkAddress()`. If this channel does not support multiple addresses, `onLinkStatusChanged()` must be called with the passed in handle and status set to `LINK_DESTROYED`.

---
```c++
ComponentStatus destroyLink(RaceHandle handle, const LinkID &linkId);
```
This call should clean up all resources related to the specified link. A call to `onLinkStatusChanged()` with status equal to `LINK_DESTROYED` must be issued for this link.

---
```c++
std::vector<EncodingParameters> getActionParams(const Action &action);
```
Get encoding parameters for each encoding associated with an action. The encodings associated with the action should match the list returned in `getTransportProperties()`

---
```c++
ComponentStatus enqueueContent(const EncodingParameters &params, const Action &action, const std::vector<uint8_t> &content);
```
Supply content to be utilized when performing an action.

---
```c++
ComponentStatus dequeueContent(const Action &action);
```
Remove previously enqueued content for the specified action

---
```c++
ComponentStatus doAction(const std::vector<RaceHandle> &handles, const Action &action);
```
Perform the action and send any content associated with the action. Once the action has been performed, call `onPackageStatusChanged()` for each of the handles provided.

---

# Implementing a decomposed usermodel

## The User Model SDK Interface

This is the interface that the user model uses to communicate with the rest of the race application. The core implements this interface and the user model is given an object that implements it in the `createPluginUserModel()` call.

---
```c++
ChannelResponse updateState(ComponentState state);
```
Inform the component manager of a change in state.

NOTE: This MUST be called with the state `COMPONENT_STARTED` before the component can be used. This should be done in either the constructor or `onUserInputReceived()` if the component requires user input before starting.


---
```c++
ChannelResponse requestPluginUserInput(const std::string &key, const std::string &prompt, bool cache);
```
`requestPluginUserInput()` is used to get user modifiable setting / configuration values. This is commonly called during component activation and completion of component activation delayed until the response is received. The return value of this function will contain a handle that matches a future `onUserInputReceived()` call into the plugin.

These calls should correspond to the `channel_parameters` section of the manifest file.

This is for component specific parameters.

---
```c++
ChannelResponse requestCommonUserInput(const std::string &key);
```
`requestCommonUserInput()` is used to get user modifiable setting / configuration values. This is commonly called during component activation and completion of component activation delayed until the response is received. The return value of this function will contain a handle that matches a future `onUserInputReceived()` call into the plugin.

These calls should correspond to the `channel_parameters` section of the manifest file.

This is for some common parameters that are used by multiple plugins or components.

See [Valid Common User Input](#valid-common-user-input) for valid keys.

---
```c++
ChannelResponse onTimelineUpdated();
```
Inform the component manager that the timeline of actions has been updated. The component manager will call `getTimeline()` to get the updated timeline. This is commonly called during `addLink()` or `removeLink()` to adjust actions so that packages can be sent or received during them.

---
## The User Model Component Interface

This is the interface that the rest of the RACE application uses to communicate with the user model. The user model must implement this interface and return the implementing object in the `createUserModel()` call.

---
```c++
ComponentStatus onUserInputReceived(RaceHandle handle, bool answered, const std::string &response);
```
Receive a response to a previous `requestPluginUserInput()` or `requestCommonUserInput()` call. The handle will match the handle returned in the `SdkResponse` object of the call that this is responding to.

---
```c++
UserModelProperties getUserModelProperties();
```
The component manager will call this to get some properties of the user model. The fields of the UserModelProperties object returned controls how long the timeline is expected to be and how often it should be updated independent of `onTimelineUpdated()` calls. A usermodel which produces many actions per time period should have a shorter timeline period and one which is very sparse should have a longer one. The fetch period should be approximately half the timeline period.

---
```c++
ComponentStatus addLink(const LinkID &link, const LinkParameters &params);
```
This is called to inform the user model that a link has been created with the specified parameters. The link id will match a subsequent call to `removeLink()`. `params` contains a string with implementation defined meaning. It can be used to communicate information from the transport with regards to this link.

---
```c++
ComponentStatus removeLink(const LinkID &link);
```
This is called to inform the user model that a link has been removed. The link id will match the corresponding call to `addLink()`.

---
```c++
ActionTimeline getTimeline(Timestamp start, Timestamp end);
```
The component manager calls this periodically to get a list of actions for the transport to execute. Each action should have a timestamp between the start timestamp and end timestamp. The component manager will never go back in time. If it has asked for actions after some `start` timestamp, all future `getTimestamp()` calls will have `start` equal to or greater than that value.

---
```c++
ComponentStatus onTransportEvent(const Event &event);
```
This gets called in response to the transport calling `onEvent()`. Details about the event are transport specific.

---
```c++
ActionTimeline onSendPackage(const LinkID &linkId, int bytes);
```
An optional call to react to send package events. Returned events are added to the component manager's timeline. Actions with a timestamp of 0 are done immediately. This may be used to implement a usermodel which sends messages out as soon as it gets them, but be careful about transports and encodings with size limits. A common issue is that a single action is produced in response to a `sendPackage()` call, but that package takes more than one action to send. This can be worked around with the `bytes` parameter if the size limit is known. The `bytes` parameter does not include fragmentation overhead per action which can be 30 bytes or higher if fragments of several packages are packed into a single action.

---

# The manifest.json file
The manifest.json file is a json file that informs the core of properties of each plugin before it gets loaded.  This allows the core to know if it needs to be loaded and how to load it. There must be a manifest.json in each kit. The format is as follows:

```json
{
    "plugins": [
        {
            "file_path": "<kit name>",
            "plugin_type": "TA2",
            "file_type": "<shared_library|python>",
            "node_type": "any",
            "shared_library_path": "<shared library path>",
            "encodings": ["<encoding component name>", ...],
            "transports": ["<transport component name>", ...],
            "usermodels": ["<usermodel component name>", ...],
            "channels": [],
        },
    ],
    "compositions": [
        {
            "id": "<composition component name>",
            "transport": "<transport component name>",
            "usermodel": "<usermodel component name>",
            "encodings": ["<encoding component name>", ...]
        }
    ],
    "channel_properties": [
        <omitted>
    ],
    "channel_parameters": [
      {
          "key": "<key value>",
          "plugin": "<plugin name>",
          "required": true|false,
          "type": "<string|int|float>",
          "default": <value of type matching value of type field>
      },
      ...
    ]
}
```

The channel properties section is rather large and has been omitted for brevity. See the [channel properties documentation](ChannelAndLinkProperties.md) for details

The `compositions`, `channel_properties`, and `channel_parameters` sections are currently optional. There are plans for the `channel_properties` and `channel_parameters` sections to become required and it is suggested to implement them. The `compositions` section is necessary if the kit supplies a decomposed channel. 

---
## Plugins section
This section contains a list of plugin definitions. Each definition has the following entries.

```
file_path
```
The name of the kit containing this plugin

---
```
plugin_type
```
This should be "TA2" for a comm plugin

---
```
file_type
```
The allowed values are `shared_library` and `python`

---
```
node_type
```
The allowed values are `client`, `server`, and `any`. This should be `any` for a comm plugin.

---
```
shared_library_path
```
If `file_type` is `shared_library`, this should contain the path of the plugin's shared object. This path is relative to the kit directory. If the .so file is at the top level of the directory, this should just be the file name.

This key may be omitted if the file type is not `shared_library`

---
```
encodings
```
A list of encoding components contained by the plugin.

This may be omitted if this plugin does not contain any encoding components.

---
```
transports
```
A list of transport components contained by the plugin.

This may be omitted if this plugin does not contain any transport components.


---
```
usermodels
```
A list of user model components contained by the plugin.

This may be omitted if this plugin does not contain any user model components.


---
```
channels
```
A list of unified channels contained by the plugin.

This may be omitted if this plugin does not contain any unified channels.

---

## Compositions section
This section contains a list of compositions. Each definition has the following entries.

```
id
```
The id of the composition. This also acts as the channel id for the composition.

---
```
transport
```
The transport component to be used for the composition. This component may refer to a component supplied by a different kit instead of being supplied by one of the plugins in this kit.

If this component is from a different kit, the kit and where to get it should be listed in the README as a dependency. If creating or running a deployment, compositions are checked to make sure all components are available, but they are not automatically loaded.

---
```
usermodel
```
The usermodel component to be used for the composition. This component may refer to a component supplied by a different kit instead of being supplied by one of the plugins in this kit.

If this component is from a different kit, the kit and where to get it should be listed in the README as a dependency. If creating or running a deployment, compositions are checked to make sure all components are available, but they are not automatically loaded.

---
```
encodings
```
A list of encoding components to be used for the composition. These components may refer to a components supplied by a different kit instead of being supplied by the plugins in this kit.

If this component is from a different kit, the kit and where to get it should be listed in the README as a dependency. If creating or running a deployment, compositions are checked to make sure all components are available, but they are not automatically loaded.

---

## Channel Properties section

The channel properties contains a list of channel properties objects. This is very verbose, and so is omitted for brevity. Refer to the [channel property documentation](ChannelAndLinkProperties.md) for details.

## Channel Parameters section
This section contains a list of channel parameter entries. These should correspond to calls to `requestPluginUserInput()`. Each definition has the following entries.

```
key
```
The key that's used when calling `requestPluginUserInput()`.

---
```
plugin
```
What plugin calls `requestPluginUserInput()` with the specified key.

This is optional if there is only one plugin in the kit

---
```
required
```
Whether the plugin requires this key to run.

---
```
type
```
What is the type of the value associated with this key. This controls the user input screen. All values will be passed as strings.

This field is optional. The default is "string".

---
```
default
```
If this value is not supplied, this will be supplied instead. the type of this field should match the value of the type field.

This field is optional.

---



# Implementing channel configuration

Channel configuration is used to create the initial link for node that exist at the [genesis of a network](./RACE%20developer%20guide.md#genesis-node). The links that are formed depend on the network manager plugin. The network manager plugin will request links from specific channels, the config gen for each of those channels will fulfill as many links of those requested as they can and will inform network manager of which ones were fulfilled. The network manager can then either request new links or succeed / fail.

The comm plugin component of this is a generate_configs.sh script. In practice, this shell script will call into python or some other language to perform logic. This script is provided a few arguments.

The output of the script is several json files containing which requests were accepted, link addresses for those requests, and sample user input.

## Arguments
```
--range <filepath>
```
The `range` argument contains information about the services available in the network.

```json
{
    "range": {
        "RACE_nodes": [
            {
                "enclave": "<enclave name>",
                "environment": "<node environment>",
                "genesis": true,
                "identities": [],
                "name": "<node name>",
                "nat": true,
                "platform": "<platform>",
                "type": "<client|server>"
            },
            ...
        ],
        "enclaves": [
            {
                "hosts": [
                    "<node name>",
                    ...
                ],
                "name": "<enclave name>",
                "port_mapping": {
                    "*": {
                        "hosts": [
                            "<node name>",
                            ...
                        ],
                        "port": "*"
                    }
                }
            },
            ...
        ],
        "name": "<deployment name>",
        "services": [
            {
                "access": [
                    {
                        "password": "<password>",
                        "protocol": "<protocol>",
                        "url": "<service url>",
                        "userFmt": "<format string>"
                    }
                ],
                "name": "<service name>",
                "type": "<service type>"
            },
        ],
        "real_world_accounts": {
            "<node name>": {
                "<service name>": [
                    {
                        "<service credential>": "<service credential value>",
                        ...
                    },
                    ...
                ],
                ...
            },
            ...
        }
    }
}
```

---
```
--config-dir <filepath>
```
The `config-dir` argument contains the location that output files should be written to.

---
```
--ta1-request <filepath>
```
The `ta1-request` file contains the information about what links should be created. The file is of the format:


```json
{
    "links": [
        {
            "channels": ["<channel 1>", ...],
            "sender": "<node #>",
            "recipients": ["<node #>", ...],
        },
        ...
    ]
}
```

Each channel should only try to fulfill the links that contain the channel's channel id in channels.

---
```
--overwrite
```
The `overwrite` flag indicates whether config gen should overwrite existing configs.

---
```
--local
```
The `local` flag indicates whether this is a local deployment with sample services or a deployment that should use real services.

---

## Outputs

### genesis-link-addresses.json
The genesis link address file contains link addresses for all the links that should get created when each node starts up. The file is a json file of the following format:

```json
{
    "<channel_id>": {
        "<node 1>": [
            {
                "role": "loader|creator",
                "personas": ["<node #>", ...],
                "address": "<channel specific link address>",
                "description": "...",
            },
            ...
        ],
        ...
    }
}
```

The address field must contain all the information necessary to create or load the link.

### fulfilled-ta1-request.json
The fulfilled-ta1-request file contains which links were fulfilled. The format is identical to the ta1-request format

```json
{
    "links": [
        {
            "channels": ["<channel #>", ...],
            "sender": "<node #>",
            "recipients": ["<node #>", ...],
        },
        ...
    ]
}
```

### user-responses.json

The user responses file contains user responses that can be used to run deployments. This is useful if running locally for testing.

```json
{
    "<node #>": {
        "<user-input-key-1>": "...",
        ...
    }
}
```

Where user-input-key-1 matches the key argument of `requestPluginUserInput()`

# Adding a dependency (via RUNPATH)

The RACE system bundles some dependencies along with the system. These dependencies are part of the racesdk image. If a comm plugin needs one of these dependencies then including it is as simple as linking against it.

If a comm plugin requires additional shared libraries that are not included by the race system, a different approach must be taken. Additional dependencies must be bundled and built with the plugin and must be part of the kit that gets installed. e.g. they should sit within the plugin directory of the kit in this tree view:

```
kit
├── artifacts
│   ├── android-arm64-v8a-client
│   │   └── PluginTa2TwoSixStubDecomposed
│   │       ├── libPluginTa2TwoSixStubEncoding.so
|   |       └── manifest.json
...
```

Linking may be done as normal, but the RPATH must be set on the plugin library in order to find the dependency at runtime.

```cmake
set_target_properties(PluginTa2TwoSixStubEncoding PROPERTIES
    BUILD_RPATH "\${ORIGIN}/lib"
    INSTALL_RPATH "\${ORIGIN}/lib"
)
```
This is an example of setting the RPATH of a target using CMAKE. ${ORIGIN} must be used to get an RPATH relative to the library the RPATH is set on. In this case, it would now be able to find anything in a `lib` directory next to the plugin shared library.

The equivalent gcc/clang argument is:
```
-Wl,-rpath,${ORIGIN}/lib
```

# Valid Common User Input

The key argument to `requestCommonUserInput()` must take one of a few possible values. Valid values are `hostname`, for the publicly available host name (or ip address) of the current node if it has now, and `env`, for the type of environment the node is running on. Valid responses for `env` include "any", "dmz", "home", "phone", "service", and "user".

# Running a custom plugin

Follow the directions for running a race deployment with [RIB](https://github.com/tst-race/race-in-the-box). Make sure to mount a directory that contains your local kit. To use a locally built kit, use the flags `--comms-kit core=<path to local kit>` and `--comms-channel=<channel id>` when creating a deployment. Here is an example deployment create command that works with the example unified plugin created before:

```
rib deployment local create \
    --name github-test3 \
    --comms-kit core=plugin-ta2-twosix-cpp \
    --comms-kit local=/code/example-decomposed-comm-plugin/kit/ \
    --comms-channel StubComposition \
    --comms-channel twoSixDirectCpp \
    --comms-channel twoSixIndirectCpp \
    --linux-client-count=3 --linux-server-count=3
```

# Debugging

Logs from a local deployment are available in the deployment logs directory. This is located at `~/.race/rib/deployments/local/<deployment name>/logs`. Each node will have its own log directory. Core logs are written to `race.log`. `race.log` may also be written to by any plugin using the `RaceLog::log*`functions. The plugin config supplied to each plugin contains a path that may be used to write plugin specific logs.

Additionally, OpenTracing is used to monitor traces across the system. The opentracing containers are spun up along with a deployment. To access opentracing logs, visit http://localhost:16686/.

Rib contains message, test, and verification commands. See Rib documentation or `rib deployment local test --help` and `rib deployment local message --help` for more info.

It is possible to use gdb to debug the a linux-based race node as well. Using `docker exec -it <node name> bash` should get a shell from which you can run `gdb racetestapp`. If you need to debug a running instance the instructions are slightly different.

If the host is linux-based, it's possible to connect to the running process from outside the docker container once you know the pid. The pid can figured out by getting the container id containing the process you want to inspect and running `systemd-cgls` to see the running processes. The executable and libraries may need to be copied out of the container to allow gdb access to debug info.

One solution that works for both Mac and Linux is to start the containers with privileged (or at least the SYS_PTRACE capability). This can be done by modifying the docker compose yaml file located at `~/.race/rib/deployments/local/<deployment name>/docker_compose.yml`. This must be done before starting the container and running the process. Once the container is privileged, gdb from inside the container using the target's pid.


# Adapting these instructions for other programming languages

Language shims are provided via SWIG for python and golang. Language shims for rust are built inside the rust exemplar. The apis are generally the same, adapted for the language types. View the exemplar code in [race-core](https://github.com/tst-race/race-core) for examples.

One area of difference is for python manifest.json files. As python is not compiled, a .so file is not created. Instead, a python interpreter is used and python objects created by it. For unified plugins, the `python_module` and `python_class` keys should be set for the unified plugin. For a decomposed plugin only the `python_module` key is necessary, however an additional `create(Encoding|Transport|UserModel)` function should be present. This `create*` function has the same arguments and serves the same purpose as the `create*` C function given in the example code above.

Here is an example manifest.json for a unified plugin:

```json
{
    "plugins": [
        {
            "file_path": "PluginTa2TwoSixPython",
            "plugin_type": "TA2",
            "file_type": "python",
            "node_type": "any",
            "python_module": "PluginTa2TwoSixPython.PluginTA2TwoSixPython",
            "python_class": "PluginTA2TwoSixPython",
            "channels": ["twoSixDirectPython", "twoSixIndirectPython"]
        }
    ],
    "channel_properties": {
        <omitted>
    },
    "channel_parameters": [
    ]
}
```