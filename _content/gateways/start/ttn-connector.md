---
title: The Things Network Gateway connector protocol
zindex: 100
---

# The Things Network gateway connector protocol

This page describes a protobuf-over-MQTT protocol between TheThingsNetwork gateway and TheThingsNetwork bridge. The connector protocol can be implemented in C language and easily used on bare-metal platfroms, that are not powerful enough to run Linux. The protocol can be seen as a alternative to Semtech protcol and as a simplification of the gRPC protcol, used by default TTN forwarder.

## Is This Necessary?

Yes. This protocol provides the following advantages over [Semtech's packet forwarder](https://github.com/Lora-net/packet_forwarder) reference implementation:

1. When using TCP/IP, there is an open and more reliable connection with The Things Network as compared to UDP which suffers from congestion and firewalls.
1. When using MQTT, we build on top of a proven and well-supported messaging protocol as compared to custom headers and breaking changes on message protocol level
1. When using authentication, we can identify gateways according to their ID and security key as compared to user-defined EUIs which deemed unreliable
1. When using Protocol Buffers, we use proven and well-supported (un)packing and we are binary compatible with The Things Network back-end while minimizing payload and allow for protocol changes

## Existing implementation

There is the official Gateway Connector Library, which implements the protocol. Check it, if you don't want implement protocol from scratch.

## Bridges

TTN bridge is a host that supports the TTN connector protocol. Below is the list of bridges, provided by TTN:  

* `bridge.asia-se.thethings.network`
* `bridge.brazil.thethings.network`
* `bridge.eu.thethings.network`
* `bridge.us-west.thethings.network`
* `router.thethingsnetwork.jp:1883`
* `thethings.meshed.com.au:1882`
* `ttn.opennetworkinfrastructure.org:1882`

```

1. Describe connect/disconnect/send/receive MQTT procedure.
2. Describe connect/disconnect/send/receive protobufs.
3. Add commands examples to generate C structures from given protobufs.
4. Add some code examples.

```

## Protocol flow

### Connection

Briefly, a connection procedure can be performed by following steps:

1. Sending CONNECT MQTT message.
1. Sending PUBLISH MQTT message to the `connect` topic with a protobuf payload.
1. Subscribing to `[id]/down` topic, where `[id]` is a gateway ID.

Refer to sections below for more detailed description of each step.

#### MQTT CONNECT message format

The CONNECT MQTT message in question conforms to the MQTT v3.1 specification.

Following parameters must be used for CONNECT message:

Additionally, TTN client must supply following parameters with MQTT CONNECT:

* Client Identifier: TTN gateway ID.
* User Name: TTN gateway key.
* QoS level: 1

An MQTT password can be ignored. The gateway is authenticated by matching the gateway ID and the gateway key.

Optionally, TTN client can use the MQTT Will Message option when sending MQTT CONNECT. In such case, the will message must be serialized using `DisconnectMessage` protobuf, that can be found in `github.com/TheThingsNetwork/gateway-connector-bridge/types/types.proto`. 

* [`DisconnectMessage` - top-level container for the disconnect message](https://github.com/TheThingsNetwork/gateway-connector-bridge/blob/8a00f170d96aa0867ac43faa6d1c0ad705204a81/types/types.proto#L17-L20):
  * `id` - string - gateway ID.
  * `key` - string - gateway key.

The Will Message must, if being used, must have following parameters:

* QoS level: 1
* RETAIN flag: false

#### Connection message (MQTT PUBLISH) format

The PUBLISH MQTT message in question conforms to the MQTT v3.1 specification.

Additionally, TTN client must supply a protobuf payload, described by the `ConnectMessage` protobuf, that can be found in `github.com/TheThingsNetwork/gateway-connector-bridge/types/types.proto`.

* [`ConnectMessage` - top-level container for the connect message](https://github.com/TheThingsNetwork/gateway-connector-bridge/blob/8a00f170d96aa0867ac43faa6d1c0ad705204a81/types/types.proto#L12-L15):
  * `id` - string - gateway ID.
  * `key` - string - gateway key.

#### MQTT SUBSCRIBE message format

The SUBSCRIBE MQTT message in question conforms to the MQTT v3.1 specification.

### Disconnection

Briefly, a disconnection procedure can be performed by following steps:

1. Unsubscribing from `[id]/down` topic, where `[id]` is a gateway ID.
1. Sending PUBLISH MQTT message to the `disconnect` topic with a protobuf payload.
1. Sending DISCONNECT MQTT message.

Refer to sections below for more detailed description of each step.

#### MQTT UNSUBSCRIBE message format

The UNSUBSCRIBE MQTT message in question conforms to the MQTT v3.1 specification.

#### Disconnection message (MQTT PUBLISH) format

The PUBLISH MQTT message in question conforms to the MQTT v3.1 specification.

Following MQTT parameters are required by TTN for disconnect MQTT message:

* QoS level: 1
* RETAIN flag: false

Additionally, TTN client must supply a protobuf payload, described by the `DisconnectMessage` protobuf, that can be found in `github.com/TheThingsNetwork/gateway-connector-bridge/types/types.proto`. 

* [`DisconnectMessage` - top-level container for the disconnect message](https://github.com/TheThingsNetwork/gateway-connector-bridge/blob/8a00f170d96aa0867ac43faa6d1c0ad705204a81/types/types.proto#L17-L20):
  * `id` - string - gateway ID.
  * `key` - string - gateway key.

#### MQTT DISCONNECT message format

The DISCONNECT MQTT message in question conforms to the MQTT v3.1 specification.

### Sending data

A sending procedure consist of sending the PUBLISH MQTT message to `[id]/up` topic, where `[id]` is a gateway ID with a protobuf payload, as described below.

#### Uplink message (MQTT PUBLISH) format

The PUBLISH MQTT message in question conforms to the MQTT v3.1 specification.

A client must set following fields in PUBLISH message in order to send data to TTN:

* QoS level: 1
* RETAIN flag: false

Additionally, TTN client must supply a protobuf payload, described by the `UplinkMessage` protobuf, that can be found in `github.com/TheThingsNetwork/api/router/router.proto` and in nested protobuffers. Only relevant messages and their fields are described below.

`github.com/TheThingsNetwork/api/router/router.proto`. 

* [`UplinkMessage` - the top-level container for the uplink data](https://github.com/TheThingsNetwork/api/blob/22ad17f21fd52dca346a3bb4d4c9889dd9636f7d/router/router.proto#L25-L31)
  * `payload` - `bytes` - uplink LoRa payload.
  * [`protocol_metadata` - `lorawan.Metadata` - container for uplink metadata](https://github.com/TheThingsNetwork/api/blob/22ad17f21fd52dca346a3bb4d4c9889dd9636f7d/protocol/lorawan/lorawan.proto#L21-L34).
    * [`modulation` - `Modulation` - LoRa of FSK modulation flag](https://github.com/TheThingsNetwork/api/blob/22ad17f21fd52dca346a3bb4d4c9889dd9636f7d/protocol/lorawan/lorawan.proto#L16-L19)
    * `data_rate` - `string` - LoRa data rate in form SF{spreadingfactor}BW{bandwidth}.
    * `bit_rate` - `uint32` - FSK bit rate in bit/s.
    * `coding_rate` - `string` - LoRa coding rate
    * `f_cnt` - `uint32` - the full 32 bit FCnt (deprecated, do not use).
    * [`frequency_plan` - `FrequencyPlan` - frequency plan enumeration.](https://github.com/TheThingsNetwork/api/blob/master/protocol/lorawan/lorawan.proto#L62-L82)
  * `message` - `lorawan.Message` - not used when sending uplink from the gateway to TTN bridge.
  * `trace` - `trace.Trace` - not used.

### Receiving data

The downlink data is published by TTN on `[id]/down` topic, where `[id]` is a gateway ID. The payload is serialized using protobufs, as described in the following section.

#### Downlink message (MQTT PUBLISH) format

The PUBLISH MQTT message in question conforms to the MQTT v3.1 specification.

A protobuf payload is described by the `DownlinkMessage` protobuf, that can be found in `github.com/TheThingsNetwork/api/router/router.proto` and in nested protobuffers. Only relevant messages and their fields are shown below.

* [`DownlinkMessage` - the top-level container for the downlink data](https://github.com/TheThingsNetwork/api/blob/22ad17f21fd52dca346a3bb4d4c9889dd9636f7d/router/router.proto#L33-L39)
  * `payload` - `bytes` - downlink LoRa payload.
  * [`protocol_configuration` - `lorawan.TxConfiguration` - LoRa configuration](https://github.com/TheThingsNetwork/api/blob/22ad17f21fd52dca346a3bb4d4c9889dd9636f7d/protocol/lorawan/lorawan.proto#L36-L47).
    * [`modulation` - `Modulation` - LoRa of FSK modulation flag]()
    * `data_rate` - `string` - LoRa data rate in form SF{spreadingfactor}BW{bandwidth}.
    * `bit_rate` - `uint32` - FSK bit rate in bit/s.
    * `coding_rate` - `string` - LoRa coding rate
  * [`gateway_configuration` - `gateway.TxConfiguration` - gateway configuration](https://github.com/TheThingsNetwork/api/blob/22ad17f21fd52dca346a3bb4d4c9889dd9636f7d/gateway/gateway.proto#L104-L119).
    * `timestamp` - `uint32` - Timestamp (uptime of LoRa module) in microseconds with rollover.
    * `rf_chain` - `uint32` - RF chain.
    * `frequency` - `uint64` - Frequency in Hz.
    * `power` - `int32` - Transmit power in dBm.
    * `polarization_inversion` - `bool` - LoRa polarization inversion (basically always true for messages from gateway to node).
    * `frequency_deviation` - `uint32` - FSK frequency deviation in Hz
  * `message` - `lorawan.Message` - not used when receiving downlink from TTN bridge.
  * `trace` - `trace.Trace` - not used.

### Sending the gateway status message

A sending status procedure consist of sending the PUBLISH MQTT message to `[id]/status` topic, where `[id]` is a gateway ID with a protobuf payload, as described below.

#### Status message (MQTT PUBLISH) format

The PUBLISH MQTT message in question conforms to the MQTT v3.1 specification.

A client must set following fields in PUBLISH message in order to send data to TTN:

* QoS level: 1
* RETAIN flag: false

A protobuf payload is described by the `Status` protobuf, that can be found in `github.com/TheThingsNetwork/api/gateway/gateway.proto` and in nested protobuffers. Only relevant messages and their fields are shown below.

* [`Status` - the top-level container for the status data](https://github.com/TheThingsNetwork/api/blob/22ad17f21fd52dca346a3bb4d4c9889dd9636f7d/router/router.proto#L33-L39)
  * `message` - `lorawan.Message` - not used when receiving downlink from TTN bridge.
  * `trace` - `trace.Trace` - not used.


## Protocol overview

The Gateway Connector Protocol payload is serialized using Google Protocol Buffers (Protobufs for short). 
Protobufs specification
Official documentation of the Protobufs are available through following link: https://developers.google.com/protocol-buffers/

Source proto-files, that describes the payload structure of the Gateway Connector Protocol are gateway.proto and router.proto . 
MQTT specification
The Gateway Connector Protocol works on top of regular MQTT, the specification of which can be found at http://docs.oasis-open.org/mqtt/mqtt/v3.1.1/os/mqtt-v3.1.1-os.html

The protocol does not specify exact The Things Network LoRa Router to connect. Address is depends on a region. Full list of Router addresses are available in the official documentation 

The protocol expects that the Gateway ID and the Gateway Key, obtained when the gateway is registered in The Things Network, are passed with MQTT connect message as username and password, respectively.

The protocol specifies two topics:
[id]/down - topic for downlink protobuf messages, gateway should subscribe to it. [id]is the Gateway ID.
[id]/up - topic for publishing uplink messages from the gateway. [id]is the Gateway ID.
The Things Network connector library
TODO

## Existing implementation
