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

#### MQTT CONNECT

The CONNECT MQTT message in question conforms to the MQTT v3.1 specification.

Following parameters must be used for CONNECT message:

Additionally, TTN client must supply following parameters with MQTT CONNECT:

* Client Identifier: TTN gateway ID.
* User Name: TTN gateway key.
* QoS level: 1

An MQTT password can be ignored. The gateway is authenticated by matching the gateway ID and the gateway key.

Optionally, TTN client can use the MQTT Will Message option when sending MQTT CONNECT. In such case, the will message must be serialized using `DisconnectMessage` protobuf, that can be found in `github.com/TheThingsNetwork/gateway-connector-bridge/types/types.proto`. Relevant message is described below.

```protobuf
syntax = "proto3";

import "github.com/gogo/protobuf/gogoproto/gogo.proto";

package types;

option go_package = "github.com/TheThingsNetwork/gateway-connector-bridge/types";

// ... 

message DisconnectMessage {
  string id = 1 [(gogoproto.customname) = "GatewayID"];
  string key   = 3;
}

```

The Will Message must, if being used, must have following parameters:

* QoS level: 1
* RETAIN flag: false


#### MQTT PUBLISH

The PUBLISH MQTT message in question conforms to the MQTT v3.1 specification.

Additionally, TTN client must supply a protobuf payload, described by the `ConnectMessage` protobuf, that can be found in `github.com/TheThingsNetwork/gateway-connector-bridge/types/types.proto`. Relevant message is described below.

```protobuf
syntax = "proto3";

import "github.com/gogo/protobuf/gogoproto/gogo.proto";

package types;

option go_package = "github.com/TheThingsNetwork/gateway-connector-bridge/types";

message ConnectMessage {
  string id    = 1 [(gogoproto.customname) = "GatewayID"];
  string key   = 3;
}

// ... 

```

#### MQTT SUBSCRIBE

The SUBSCRIBE MQTT message in question conforms to the MQTT v3.1 specification.

### Disconnection

Briefly, a disconnection procedure can be performed by following steps:

1. Unsubscribing from `[id]/down` topic, where `[id]` is a gateway ID.
1. Sending PUBLISH MQTT message to the `disconnect` topic with a protobuf payload.
1. Sending DISCONNECT MQTT message.

Refer to sections below for more detailed description of each step.

#### MQTT UNSUBSCRIBE

The UNSUBSCRIBE MQTT message in question conforms to the MQTT v3.1 specification.

#### MQTT PUBLISH

The PUBLISH MQTT message in question conforms to the MQTT v3.1 specification.

Following MQTT parameters are required by TTN for disconnect MQTT message:

* QoS level: 1
* RETAIN flag: false

Additionally, TTN client must supply a protobuf payload, described by the `DisconnectMessage` protobuf, that can be found in `github.com/TheThingsNetwork/gateway-connector-bridge/types/types.proto`. Relevant message is described below.

```protobuf
syntax = "proto3";

import "github.com/gogo/protobuf/gogoproto/gogo.proto";

package types;

option go_package = "github.com/TheThingsNetwork/gateway-connector-bridge/types";

// ... 

message DisconnectMessage {
  string id = 1 [(gogoproto.customname) = "GatewayID"];
  string key   = 3;
}

```

#### MQTT DISCONNECT

The DISCONNECT MQTT message in question conforms to the MQTT v3.1 specification.

### Sending data

A sending procedure consist of sending the PUBLISH MQTT message to `[id]/up` topic, where `[id]` is a gateway ID with a protobuf payload, as described below.

#### MQTT PUBLISH

The PUBLISH MQTT message in question conforms to the MQTT v3.1 specification.

A client must set following fields in PUBLISH message in order to send data to TTN:

* QoS level: 1
* RETAIN flag: false

Additionally, TTN client must supply a protobuf payload, described by the `UplinkMessage` protobuf, that can be found in `github.com/TheThingsNetwork/api/router/router.proto` and in nested protobuffers. Relevant messages and their fields are described below.

`github.com/TheThingsNetwork/api/router/router.proto`. Only the pop-level container for the uplink data is shown for clarity.

```protobuf

// ... 

message UplinkMessage {
  bytes                payload            = 1;
  protocol.Message     message            = 2;
  protocol.RxMetadata  protocol_metadata  = 11 [(gogoproto.nullable) = false];
  gateway.RxMetadata   gateway_metadata   = 12 [(gogoproto.nullable) = false];
  trace.Trace          trace              = 21;
}

// ...

```

`github.com/TheThingsNetwork/api/protocol/protocol.proto`. Only RX (uplink) metadata proxy type is shown for clarity.

```protobuf

// ...

message RxMetadata {
  oneof protocol {
    lorawan.Metadata lorawan = 1 [(gogoproto.customname) = "LoRaWAN"];
  }
}

// ...

```

`github.com/TheThingsNetwork/api/protocol/lorawan/lorawan.proto`. In the uplink procedure, only `Metadata` and `Modulaion` types are used. 

```protobuf

// ...

enum Modulation {
  LORA = 0;
  FSK  = 1;
}

message Metadata {
  Modulation  modulation   = 11;
  // LoRa data rate - SF{spreadingfactor}BW{bandwidth}
  string      data_rate    = 12;
  // FSK bit rate in bit/s
  uint32      bit_rate     = 13;
  // LoRa coding rate
  string      coding_rate  = 14;

  // Store the full 32 bit FCnt (deprecated; do not use)
  uint32      f_cnt = 15;

  FrequencyPlan frequency_plan = 16;
}

// ...

```

`github.com/TheThingsNetwork/api/gateway/gateway.proto`:

```

message RxMetadata {
  string  gateway_id = 1 [(gogoproto.customname) = "GatewayID"];

  // Indicates whether the gateway is trusted. Components that are able to verify gateway trust MUST do so and set this value accordingly
  bool    gateway_trusted = 2;

  // Timestamp (uptime of LoRa module) in microseconds with rollover
  uint32  timestamp  = 11;
  // Time in Unix nanoseconds
  int64   time       = 12;

  // Encrypted time from the Gateway FPGA
  bytes   encrypted_time = 13;

  uint32  rf_chain   = 21;
  uint32  channel    = 22;

  repeated Antenna antennas = 30;

  // Frequency in Hz
  uint64  frequency  = 31;
  // Received signal strength in dBm
  float   rssi       = 32 [(gogoproto.customname) = "RSSI"];
  // Signal-to-noise-ratio in dB
  float   snr        = 33 [(gogoproto.customname) = "SNR"];

  message Antenna {
    uint32 antenna = 1;
    uint32 channel = 2;

    // Received signal power in dBm
    float  rssi         = 3 [(gogoproto.customname) = "RSSI"];

    // Received channel power in dBm
    float  channel_rssi = 5 [(gogoproto.customname) = "ChannelRSSI"];

    // Standard deviation of the RSSI
    float  rssi_standard_deviation = 6 [(gogoproto.customname) = "RSSIStandardDeviation"];

    // Frequency offset (Hz)
    int64 frequency_offset = 7;

    // Signal-to-noise-ratio in dB
    float  snr     = 4 [(gogoproto.customname) = "SNR"];

    // Encrypted fine timestamp from the Gateway FPGA
    bytes encrypted_time = 10;

    // Fine timestamp from the Gateway FPGA (decrypted)
    int64 fine_time = 11;
  }

  LocationMetadata location = 41;
}

// ...

```

### Receiving data

### Sending status message

## Protobufs in question

### Connection and disconnection payload protobuf

Protobuf path: `github.com/TheThingsNetwork/gateway-connector-bridge/types/types.proto`

```protobuf
// Copyright © 2017 The Things Network
// Use of this source code is governed by the MIT license that can be found in the LICENSE file.

syntax = "proto3";

import "github.com/gogo/protobuf/gogoproto/gogo.proto";

package types;

option go_package = "github.com/TheThingsNetwork/gateway-connector-bridge/types";

message ConnectMessage {
  string id    = 1 [(gogoproto.customname) = "GatewayID"];
  string key   = 3;
}

message DisconnectMessage {
  string id = 1 [(gogoproto.customname) = "GatewayID"];
  string key   = 3;
}
```

### Router payload protobuf

Protobuf path: `github.com/TheThingsNetwork/gateway-connector-bridge/types/types.proto`

```protobuf
// Copyright © 2017 The Things Network
// Use of this source code is governed by the MIT license that can be found in the LICENSE file.

syntax = "proto3";

import "google/protobuf/empty.proto";

import "github.com/gogo/protobuf/gogoproto/gogo.proto";

import "github.com/TheThingsNetwork/api/api.proto";
import "github.com/TheThingsNetwork/api/protocol/protocol.proto";
import "github.com/TheThingsNetwork/api/gateway/gateway.proto";
import "github.com/TheThingsNetwork/api/trace/trace.proto";

package router;

option csharp_namespace = "TheThingsNetwork.API.Router";
option go_package = "github.com/TheThingsNetwork/api/router";
option java_package = "org.thethingsnetwork.api.router";
option java_outer_classname = "RouterProto";
option java_multiple_files = true;

message SubscribeRequest {}

message UplinkMessage {
  bytes                payload            = 1;
  protocol.Message     message            = 2;
  protocol.RxMetadata  protocol_metadata  = 11 [(gogoproto.nullable) = false];
  gateway.RxMetadata   gateway_metadata   = 12 [(gogoproto.nullable) = false];
  trace.Trace          trace              = 21;
}

message DownlinkMessage {
  bytes                     payload                 = 1;
  protocol.Message          message                 = 2;
  protocol.TxConfiguration  protocol_configuration  = 11 [(gogoproto.nullable) = false];
  gateway.TxConfiguration   gateway_configuration   = 12 [(gogoproto.nullable) = false];
  trace.Trace               trace                   = 21;
}

message DeviceActivationRequest {
  bytes                        payload              = 1;
  protocol.Message             message              = 2;
  bytes                        dev_eui              = 11 [(gogoproto.customname) = "DevEUI", (gogoproto.nullable) = false, (gogoproto.customtype) = "github.com/TheThingsNetwork/ttn/core/types.DevEUI"];
  bytes                        app_eui              = 12 [(gogoproto.customname) = "AppEUI", (gogoproto.nullable) = false, (gogoproto.customtype) = "github.com/TheThingsNetwork/ttn/core/types.AppEUI"];
  protocol.RxMetadata          protocol_metadata    = 21 [(gogoproto.nullable) = false];
  gateway.RxMetadata           gateway_metadata     = 22 [(gogoproto.nullable) = false];
  protocol.ActivationMetadata  activation_metadata  = 23;
  trace.Trace                  trace                = 31;
}

message DeviceActivationResponse {
  // NOTE: In LoRaWAN, device activations are accepted with DownlinkMessages, so
  // this message is just an Ack.
  //
  // bytes                     payload                 = 1;
  // protocol.Message          message                 = 2;
  // protocol.TxConfiguration  protocol_configuration  = 11;
  // gateway.TxConfiguration   gateway_configuration   = 12;
  // trace.Trace               trace                   = 21;
}

// The Router service provides pure network functionality
service Router {
  // Gateway streams status messages to Router
  rpc GatewayStatus(stream gateway.Status) returns (google.protobuf.Empty);

  // Gateway streams uplink messages to Router
  rpc Uplink(stream UplinkMessage) returns (google.protobuf.Empty);

  // Gateway subscribes to downlink messages from Router
  // It is possible to open multiple subscriptions (but not recommended).
  // If you do this, you are responsible for de-duplication of downlink messages.
  rpc Subscribe(SubscribeRequest) returns (stream DownlinkMessage);

  // Gateway requests device activation
  rpc Activate(DeviceActivationRequest) returns (DeviceActivationResponse);
}

// message GatewayStatusRequest is used to request the status of a gateway from
// this Router
message GatewayStatusRequest {
  string gateway_id = 1 [(gogoproto.customname) = "GatewayID"];
}

message GatewayStatusResponse {
  int64           last_seen  = 1;
  gateway.Status  status     = 2 [(gogoproto.nullable) = false];
}

// message StatusRequest is used to request the status of this Router
message StatusRequest {}

// message Status is the response to the StatusRequest
message Status {
  api.SystemStats    system    = 1;
  api.ComponentStats component = 2;

  api.Rates gateway_status   = 11;
  api.Rates uplink           = 12;
  api.Rates downlink         = 13;
  api.Rates activations      = 14;

  // Connections
  uint32  connected_gateways  = 21;
  uint32  connected_brokers   = 22;
}

// The RouterManager service provides configuration and monitoring functionality
service RouterManager {
  // Gateway owner or network operator requests Gateway status from Router Manager
  // Deprecated: Use monitor API (NOC) instead of this
  rpc GatewayStatus(GatewayStatusRequest) returns (GatewayStatusResponse);

  // Network operator requests Router status
  rpc GetStatus(StatusRequest) returns (Status);
}
```

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
