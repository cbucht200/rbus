# RBUS Wire Protocol

**Version:** 2.0  
**Status:** Technical Specification  
**Date:** January 2026  
**Authors:** RDK Central (rbus) & Community Contributors

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Architecture Overview](#2-architecture-overview)
3. [Transport Layer](#3-transport-layer)
4. [RTMessage Container Format](#4-rtmessage-container-format)
5. [Control Messages](#5-control-messages)
6. [RBus Application Messages](#6-rbus-application-messages)
7. [Data Type Encoding](#7-data-type-encoding)
8. [Status Codes](#8-status-codes)
9. [Security Considerations](#9-security-considerations)
10. [Appendix](#10-appendix)

---

## 1. Introduction

### 1.1 Purpose

This document specifies the wire protocol for RTMessage (RealTime Message), the transport layer of the RDK Bus (RBus) messaging system. RTMessage provides a lightweight, efficient inter-process communication (IPC) mechanism for software components running on RDK (Reference Design Kit) devices.

### 1.2 Scope

This specification defines:
- The binary encoding format for RTMessage containers
- Control message formats for router communication
- Application message formats for RBus operations
- Data type serialization rules
- Error handling and status codes

### 1.3 Terminology

- **RTMessage**: Transport-layer message container that encapsulates all communications
- **RtRouter/rtrouted**: Message broker daemon that routes messages between components
- **Component**: An RBus client or provider process
- **Topic**: A routing identifier used to direct messages
- **Payload**: The data carried within an RTMessage

### 1.4 Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

All multi-byte integer values are encoded in **big-endian** (network byte order) unless otherwise specified.

---

## 2. Architecture Overview

### 2.1 System Components

```
┌──────────────┐         ┌──────────────┐
│   Provider   │         │   Consumer   │
│  Component   │         │  Component   │
└──────┬───────┘         └──────┬───────┘
       │                        │
       │    Unix Domain         │
       │    Socket              │
       │                        │
       └────────┬───────────────┘
                │
        ┌───────▼────────┐
        │   rtrouted     │
        │  (Router       │
        │   Daemon)      │
        └────────────────┘
```

### 2.2 Communication Flow

1. Components connect to rtrouted via Unix Domain Socket at `/tmp/rtrouted`
2. Components subscribe to topics of interest
3. Messages are sent to topics and routed by rtrouted
4. RBus operations (get/set/invoke/event) are implemented on top of RTMessage

---

## 3. Transport Layer

### 3.1 Connection Establishment

**Socket Type:** Unix Domain Socket (SOCK_STREAM)  
**Socket Path:** `/tmp/rtrouted`  
**Protocol:** Stream-based, connection-oriented

### 3.2 Connection Lifecycle

1. Client opens socket connection to `/tmp/rtrouted`
2. Client sends control messages to subscribe to topics
3. Bidirectional message exchange occurs
4. Client closes socket on shutdown

---

## 4. RTMessage Container Format

### 4.1 Message Structure

An RTMessage is the fundamental unit of communication. All messages (control and application) are wrapped in this container.

#### 4.1.1 Binary Layout

```
┌────────────────────────────────────────────────────┐
│  Header Marker (0xAAAA)               │ 2 bytes   │
├────────────────────────────────────────────────────┤
│  Version                               │ 2 bytes   │
├────────────────────────────────────────────────────┤
│  Header Length                         │ 2 bytes   │
├────────────────────────────────────────────────────┤
│  Sequence Number                       │ 4 bytes   │
├────────────────────────────────────────────────────┤
│  Flags                                 │ 4 bytes   │
├────────────────────────────────────────────────────┤
│  Control Data                          │ 4 bytes   │
├────────────────────────────────────────────────────┤
│  Payload Length                        │ 4 bytes   │
├────────────────────────────────────────────────────┤
│  Topic Length                          │ 4 bytes   │
├────────────────────────────────────────────────────┤
│  Topic (variable)                      │ N bytes   │
├────────────────────────────────────────────────────┤
│  Reply Topic Length                    │ 4 bytes   │
├────────────────────────────────────────────────────┤
│  Reply Topic (variable)                │ M bytes   │
├────────────────────────────────────────────────────┤
│  [Roundtrip Fields] (optional)         │ 20 bytes  │
├────────────────────────────────────────────────────┤
│  Header Marker (0xAAAA)               │ 2 bytes   │
├────────────────────────────────────────────────────┤
│  Payload (variable)                    │ P bytes   │
└────────────────────────────────────────────────────┘
```

#### 4.1.2 Field Definitions

| Field | Type | Size | Description |
|-------|------|------|-------------|
| Header Marker (Start) | uint16 | 2 | Fixed value 0xAAAA marking header start |
| Version | uint16 | 2 | Protocol version (MUST be 2) |
| Header Length | uint16 | 2 | Total header length including topics |
| Sequence Number | uint32 | 4 | Monotonically increasing message identifier |
| Flags | uint32 | 4 | Message flags (see Section 4.1.3) |
| Control Data | uint32 | 4 | Reserved for future use (currently 0) |
| Payload Length | uint32 | 4 | Length of payload in bytes |
| Topic Length | uint32 | 4 | Length of topic string in bytes |
| Topic | UTF-8 bytes | variable | Routing topic (raw UTF-8 bytes, no null terminator, no encoding wrapper) |
| Reply Topic Length | uint32 | 4 | Length of reply topic string |
| Reply Topic | string | variable | Topic for replies (UTF-8, no null terminator) |
| Roundtrip Fields | 5×uint32 | 20 | Optional timing fields (MSG_ROUNDTRIP_TIME) |
| Header Marker (End) | uint16 | 2 | Fixed value 0xAAAA marking header end |
| Payload | bytes | variable | Message payload (JSON or MessagePack) |

**Note:** The Header Length calculation (when MSG_ROUNDTRIP_TIME not defined):
```
Header Length = 32 + Topic.length + Reply Topic.length
```

Where 32 bytes includes:
- 4 bytes for markers (2×2)
- 28 bytes for fixed fields (2+2+4+4+4+4+4+4)

#### 4.1.3 Flags Field

The Flags field is a bitfield with the following defined bits:

| Flag | Bit Mask | Value | Description |
|------|----------|-------|-------------|
| Request | 0x01 | 0x00000001 | Message is a request |
| Response | 0x02 | 0x00000002 | Message is a response |
| Undeliverable | 0x04 | 0x00000004 | Message could not be delivered |
| Tainted | 0x08 | 0x00000008 | Message integrity compromised |
| RawBinary | 0x10 | 0x00000010 | Payload is binary (MessagePack) |
| Encrypted | 0x20 | 0x00000020 | Payload is encrypted |

**Flag Combinations:**
- **Request + RawBinary**: RBus application message request (MessagePack-encoded payload)
- **Response + RawBinary**: RBus application message response (MessagePack-encoded payload)
- **Request (only)**: Control message (JSON-encoded payload)

### 4.2 Validation Rules

An RTMessage is considered valid if and only if:

1. Both header markers equal 0xAAAA
2. Version equals 2
3. Header Length = 32 + Topic Length + Reply Topic Length (without roundtrip) OR 52 + Topic Length + Reply Topic Length (with roundtrip)
4. Topic Length ≤ 256
5. Reply Topic Length ≤ 256
6. Payload Length matches actual payload size

Implementations MUST validate these constraints and reject invalid messages.

---

## 5. Control Messages

Control messages are sent between components and rtrouted to manage subscriptions. They use JSON encoding in the payload.

### 5.1 Subscribe Message

#### 5.1.1 RTMessage Envelope

| Field | Value |
|-------|-------|
| Flags | 0x01 (Request, without RawBinary flag) |
| Topic | `_RTROUTED.INBOX.SUBSCRIBE` or similar router control topic |
| Payload Encoding | JSON (RFC 8259) |

#### 5.1.2 JSON Payload Format

**Subscribe Request:**
```json
{
  "add": 1,
  "topic": "<topic-to-subscribe>",
  "route_id": 1
}
```

**Unsubscribe Request:**
```json
{
  "add": 0,
  "topic": "<topic-to-unsubscribe>",
  "route_id": 1
}
```

#### 5.1.3 Field Definitions

| Field | Type | Description |
|-------|------|-------------|
| add | integer | 1 = subscribe, 0 = unsubscribe |
| topic | string | Topic pattern to subscribe/unsubscribe |
| route_id | integer | Route identifier (typically 1) |

**Note:** The `route_id` field is currently always set to 1 and is reserved for future routing enhancements.

### 5.2 Subscription Lifecycle

1. Component connects to router
2. Component sends subscribe control message for each topic
3. Router acknowledges subscription
4. Router forwards matching messages to component
5. Component sends unsubscribe before disconnecting

---

## 6. RBus Application Messages

RBus application messages implement the data model operations. These messages use MessagePack encoding (msgpack.org specification) for their payloads and are wrapped in RTMessage containers with the RawBinary flag set.

### 6.1 Common Structure

All RBus messages consist of:
1. **Message Body**: Operation-specific MessagePack-encoded data
2. **Metadata**: MessagePack-encoded metadata appended to the body

**RTMessage Envelope:**
- Flags: 0x11 (Request + RawBinary) for requests
- Flags: 0x12 (Response + RawBinary) for responses
- Topic: Target element name (e.g., `Device.DeviceInfo.Manufacturer`)
- Reply Topic: Sender's inbox topic (e.g., `rbus.component.INBOX.12345`)

### 6.2 RBus Metadata

All RBus message types (except Event) include a standard metadata structure.

#### 6.2.1 Encoding Format

Metadata is MessagePack-encoded with the following structure:

```
┌─────────────────────────────────────┐
│  method_name (string)               │
├─────────────────────────────────────┤
│  ot_parent (string, empty)          │
├─────────────────────────────────────┤
│  ot_state (string, empty)           │
├─────────────────────────────────────┤
│  offset (i32, fixed)                │
└─────────────────────────────────────┘
```

#### 6.2.2 Field Definitions

| Field | Type | Description |
|-------|------|-------------|
| method_name | MessagePack string | Method identifier (see table below) |
| ot_parent | MessagePack string | OpenTelemetry parent context (reserved, empty) |
| ot_state | MessagePack string | OpenTelemetry state (reserved, empty) |
| offset | MessagePack int32 (fixed) | Byte offset from message start to metadata (MUST use MessagePack fixint32 format 0xd2) |

**IMPORTANT:** The offset field MUST be encoded as a fixed 32-bit signed integer using MessagePack type 0xd2 (int32).

#### 6.2.3 Method Names

| Method Name | Operation |
|-------------|-----------|
| `METHOD_RPC` | Remote method invocation |
| `METHOD_GETPARAMETERVALUES` | Get property value |
| `METHOD_SETPARAMETERVALUES` | Set property value |
| `METHOD_SUBSCRIBE` | Subscribe to event |
| `METHOD_GETPARAMETERNAMES` | Discover/enumerate parameter names |
| `METHOD_ADDTBLROW` | Add table row |
| `METHOD_DELETETBLROW` | Delete table row |
| `METHOD_COMMIT` | Commit transaction (reserved) |
| `METHOD_GETPARAMETERATTRIBUTES` | Get parameter attributes (reserved) |
| `METHOD_SETPARAMETERATTRIBUTES` | Set parameter attributes (reserved) |
| `METHOD_RESPONSE` | Generic response |

### 6.3 Get Operation

Retrieve the value of a property.

#### 6.3.1 Get Request

**Message Structure (MessagePack-encoded):**
```
┌─────────────────────────────────────┐
│  component (MessagePack string)     │
├─────────────────────────────────────┤
│  param_size (MessagePack int) = 1   │
├─────────────────────────────────────┤
│  parameter (MessagePack string)     │
├─────────────────────────────────────┤
│  [RBus Metadata]                    │
│    method_name = "METHOD_           │
│                   GETPARAMETERVALUES"│
└─────────────────────────────────────┘
```

**Field Descriptions:**

| Field | Type | Description |
|-------|------|-------------|
| component | MessagePack string | Requesting component name |
| param_size | MessagePack integer | Number of parameters (always 1) |
| parameter | MessagePack string | Property name to retrieve |
| [metadata] | MessagePack-encoded RBusMetadata | Standard metadata with GET method |

#### 6.3.2 Get Response

**Success Case (status = 0 or 100) - MessagePack-encoded:**
```
┌─────────────────────────────────────┐
│  status (MessagePack int)           │
├─────────────────────────────────────┤
│  has_data (MessagePack int) = 1     │
├─────────────────────────────────────┤
│  value (MessagePack RBusProperty)   │
├─────────────────────────────────────┤
│  [RBus Metadata]                    │
│    method_name = "METHOD_RESPONSE"  │
└─────────────────────────────────────┘
```

**Error Case (status ≠ 0 and ≠ 100) - MessagePack-encoded:**
```
┌─────────────────────────────────────┐
│  status (MessagePack int)           │
├─────────────────────────────────────┤
│  has_data (MessagePack int) = 0     │
├─────────────────────────────────────┤
│  [RBus Metadata]                    │
│    method_name = "METHOD_RESPONSE"  │
└─────────────────────────────────────┘
```

**Status Codes:**
- `0` or `100`: Success
- Other values: Error (see Section 8)

### 6.4 Set Operation

Update the value of a property.

#### 6.4.1 Set Request

**MessagePack-encoded structure:**
```
┌─────────────────────────────────────┐
│  session_id (MessagePack int) = 0   │
├─────────────────────────────────────┤
│  component (MessagePack string)     │
├─────────────────────────────────────┤
│  need_rollback (MessagePack int) = 0│
├─────────────────────────────────────┤
│  param_size (MessagePack int) = 1   │
├─────────────────────────────────────┤
│  parameter (MessagePack RBusProperty)│
├─────────────────────────────────────┤
│  commit (MessagePack string)="TRUE"/"FALSE"│
├─────────────────────────────────────┤
│  [RBus Metadata]                    │
│    method_name = "METHOD_           │
│                   SETPARAMETERVALUES"│
└─────────────────────────────────────┘
```

**Field Descriptions:**

| Field | Type | Description |
|-------|------|-------------|
| session_id | MessagePack integer | Session identifier (0 for non-session) |
| component | MessagePack string | Requesting component name |
| need_rollback | MessagePack integer | Rollback capability flag (reserved, 0) |
| param_size | MessagePack integer | Number of parameters (always 1) |
| parameter | MessagePack-encoded RBusProperty | Property name and new value |
| commit | MessagePack string | "TRUE" to commit, "FALSE" to defer |
| [metadata] | MessagePack-encoded RBusMetadata | Standard metadata with SET method |

#### 6.4.2 Set Response

**Success Case (MessagePack-encoded):**
```
┌─────────────────────────────────────┐
│  status (MessagePack int) = 0 or 100│
├─────────────────────────────────────┤
│  [RBus Metadata]                    │
│    method_name = "METHOD_RESPONSE"  │
└─────────────────────────────────────┘
```

**Error Case (MessagePack-encoded):**
```
┌─────────────────────────────────────┐
│  status (MessagePack int) ≠ 0       │
├─────────────────────────────────────┤
│  error_message (MessagePack string) │
├─────────────────────────────────────┤
│  [RBus Metadata]                    │
│    method_name = "METHOD_RESPONSE"  │
└─────────────────────────────────────┘
```

### 6.5 Invoke Operation

Execute a remote method on a provider.

#### 6.5.1 Invoke Request

**MessagePack-encoded structure:**
```
┌─────────────────────────────────────┐
│  session_id (MessagePack int) = 0   │
├─────────────────────────────────────┤
│  method_name (MessagePack string)   │
├─────────────────────────────────────┤
│  has_params (MessagePack int)       │
├─────────────────────────────────────┤
│  params (MessagePack RBusObject) [optional]│
├─────────────────────────────────────┤
│  [RBus Metadata]                    │
│    method_name = "METHOD_RPC"       │
└─────────────────────────────────────┘
```

**Field Descriptions:**

| Field | Type | Description |
|-------|------|-------------|
| session_id | MessagePack integer | Session identifier (reserved, 0) |
| method_name | MessagePack string | Name of method to invoke |
| has_params | MessagePack integer | 1 if params present, 0 otherwise |
| params | MessagePack-encoded RBusObject | Input parameters (if has_params = 1) |
| [metadata] | MessagePack-encoded RBusMetadata | Standard metadata with RPC method |

#### 6.5.2 Invoke Response

**Success Case (MessagePack-encoded):**
```
┌─────────────────────────────────────┐
│  status (MessagePack int) = 0 or 100│
├─────────────────────────────────────┤
│  result (MessagePack RBusObject)    │
├─────────────────────────────────────┤
│  [RBus Metadata]                    │
│    method_name = "METHOD_RESPONSE"  │
└─────────────────────────────────────┘
```

**Error Case:**
```
┌─────────────────────────────────────┐
│  status (int) ≠ 0                   │
├─────────────────────────────────────┤
│  [RBus Metadata]                    │
│    method_name = "METHOD_RESPONSE"  │
└─────────────────────────────────────┘
```

### 6.6 Subscribe Operation

Subscribe to property change events.

#### 6.6.1 Subscribe Request

**MessagePack-encoded structure:**
```
┌─────────────────────────────────────┐
│  event_name (MessagePack string)    │
├─────────────────────────────────────┤
│  reply_topic (MessagePack string)   │
├─────────────────────────────────────┤
│  has_payload (MessagePack int) = 1  │
├─────────────────────────────────────┤
│  payload (MessagePack binary)       │
│    ├─ component_id (MessagePack int) = 0│
│    ├─ interval (MessagePack int) = 0│
│    ├─ duration (MessagePack int) = 0│
│    └─ has_filter (MessagePack int) = 0│
├─────────────────────────────────────┤
│  publishOnSubscribe (MessagePack int) = 0│
├─────────────────────────────────────┤
│  rawDataSubscription (MessagePack int) = 0│
├─────────────────────────────────────┤
│  [RBus Metadata]                    │
│    method_name = "METHOD_SUBSCRIBE" │
└─────────────────────────────────────┘
```

**Field Descriptions:**

| Field | Type | Description |
|-------|------|-------------|
| event_name | MessagePack string | Event/property name to monitor |
| reply_topic | MessagePack string | Topic where events should be sent |
| has_payload | MessagePack integer | Always 1 (payload present) |
| payload | MessagePack binary | MessagePack-encoded subscription parameters |
| publishOnSubscribe | MessagePack integer | Send immediate event on subscribe (0=no, 1=yes) |
| rawDataSubscription | MessagePack integer | Raw data mode (0=normal, 1=raw) |
| [metadata] | MessagePack-encoded RBusMetadata | Standard metadata with SUBSCRIBE method |

**Payload Fields (MessagePack-encoded within binary payload):**

| Field | Type | Description |
|-------|------|-------------|
| component_id | MessagePack integer | Component identifier (reserved, 0) |
| interval | MessagePack integer | Polling interval in ms (0=change-based) |
| duration | MessagePack integer | Subscription duration (0=indefinite) |
| has_filter | MessagePack integer | Filter present flag (0=no filter) |

#### 6.6.2 Subscribe Response

**MessagePack-encoded structure:**
```
┌─────────────────────────────────────┐
│  status (MessagePack int)           │
├─────────────────────────────────────┤
│  [RBus Metadata]                    │
│    method_name = "METHOD_RESPONSE"  │
└─────────────────────────────────────┘
```

**Field Descriptions:**

| Field | Type | Description |
|-------|------|-------------|
| status | MessagePack integer | Result status (0=success) |
| [metadata] | MessagePack-encoded RBusMetadata | Standard metadata with RESPONSE method |

### 6.7 Discovery Operation

The discovery operation enumerates elements in the data model tree.

#### 6.7.1 Get Parameter Names Request

**MessagePack-encoded structure:**
```
┌─────────────────────────────────────┐
│  object_name (MessagePack string)   │
├─────────────────────────────────────┤
│  depth (MessagePack int)            │
├─────────────────────────────────────┤
│  get_row_names_only (MessagePack int)│
└─────────────────────────────────────┘
```

**Field Descriptions:**

| Field | Type | Description |
|-------|------|-------------|
| object_name | MessagePack string | Root object for discovery (e.g., "Device.WiFi.") |
| depth | MessagePack integer | Recursion depth (0=single level, -1=unlimited) |
| get_row_names_only | MessagePack integer | 1=table rows only, 0=all elements |

**Note:** When `get_row_names_only=1`, the operation returns only table row instance numbers and aliases.

#### 6.7.2 Get Parameter Names Response

**MessagePack-encoded structure:**
```
┌─────────────────────────────────────┐
│  status (MessagePack int)           │
├─────────────────────────────────────┤
│  count (MessagePack int)            │
├─────────────────────────────────────┤
│  ┌─ Element Info (repeated count times) ─┐
│  │  name (MessagePack string)       │
│  │  type (MessagePack int)          │
│  │  access (MessagePack int)        │
│  └──────────────────────────────────┘
└─────────────────────────────────────┘
```

**Field Descriptions:**

| Field | Type | Description |
|-------|------|-------------|
| status | MessagePack integer | Result status (0=success) |
| count | MessagePack integer | Number of elements found |
| name | MessagePack string | Full path of element (e.g., "Device.WiFi.SSID") |
| type | MessagePack integer | Element type (see rbusElementType_t) |
| access | MessagePack integer | Access permissions (read/write flags) |

**Element Types (rbusElementType_t):**

| Value | Name | Description |
|-------|------|-------------|
| 0 | RBUS_ELEMENT_TYPE_PROPERTY | Leaf property value |
| 1 | RBUS_ELEMENT_TYPE_TABLE | Multi-instance table |
| 2 | RBUS_ELEMENT_TYPE_EVENT | Event source |
| 3 | RBUS_ELEMENT_TYPE_METHOD | Invocable method |

**Access Flags:**

| Bit | Name | Description |
|-----|------|-------------|
| 0x01 | Read | Property can be read |
| 0x02 | Write | Property can be written |

**Alternative Response Format (when get_row_names_only=1):**
```
┌─────────────────────────────────────┐
│  status (MessagePack int)           │
├─────────────────────────────────────┤
│  count (MessagePack int)            │
├─────────────────────────────────────┤
│  ┌─ Row Info (repeated count times) ─┐
│  │  instance_number (MessagePack int)│
│  │  alias (MessagePack string)      │
│  └──────────────────────────────────┘
└─────────────────────────────────────┘
```

### 6.8 Table Operations

Table operations allow dynamic creation and deletion of table rows in the data model. Tables represent multi-instance objects (e.g., "Device.WiFi.AccessPoint.{i}").

#### 6.8.1 Add Table Row

Add a new row to a table.

##### 6.8.1.1 Add Row Request

**MessagePack-encoded structure:**
```
┌─────────────────────────────────────┐
│  session_id (MessagePack int) = 0   │
├─────────────────────────────────────┤
│  table_name (MessagePack string)    │
├─────────────────────────────────────┤
│  alias_name (MessagePack string)    │
├─────────────────────────────────────┤
│  [RBus Metadata]                    │
│    method_name = "METHOD_ADDTBLROW" │
└─────────────────────────────────────┘
```

**Field Descriptions:**

| Field | Type | Description |
|-------|------|-------------|
| session_id | MessagePack integer | Session identifier (reserved, 0) |
| table_name | MessagePack string | Name of table (e.g., "Device.WiFi.AccessPoint.") |
| alias_name | MessagePack string | Optional alias for new row (empty string if not used) |
| [metadata] | MessagePack-encoded RBusMetadata | Standard metadata with ADDTBLROW method |

**Note:** The `table_name` MUST end with a period (e.g., "Device.WiFi.AccessPoint.").

##### 6.8.1.2 Add Row Response

**MessagePack-encoded structure:**
```
┌─────────────────────────────────────┐
│  status (MessagePack int)           │
├─────────────────────────────────────┤
│  instance_number (MessagePack int)  │
├─────────────────────────────────────┤
│  [RBus Metadata]                    │
│    method_name = "METHOD_RESPONSE"  │
└─────────────────────────────────────┘
```

**Field Descriptions:**

| Field | Type | Description |
|-------|------|-------------|
| status | MessagePack integer | Result status (0=success) |
| instance_number | MessagePack integer | Assigned instance number for new row |
| [metadata] | MessagePack-encoded RBusMetadata | Standard metadata with RESPONSE method |

**Instance Numbering:**
- Providers assign unique instance numbers to each row
- Instance numbers SHOULD be monotonically increasing
- Rows can be accessed by instance number: "Device.WiFi.AccessPoint.1"
- Rows can also be accessed by alias if provided: "Device.WiFi.AccessPoint.[home_network]"

#### 6.8.2 Remove Table Row

Remove an existing row from a table.

##### 6.8.2.1 Remove Row Request

**MessagePack-encoded structure:**
```
┌─────────────────────────────────────┐
│  session_id (MessagePack int) = 0   │
├─────────────────────────────────────┤
│  row_name (MessagePack string)      │
├─────────────────────────────────────┤
│  [RBus Metadata]                    │
│    method_name = "METHOD_           │
│                   DELETETBLROW"     │
└─────────────────────────────────────┘
```

**Field Descriptions:**

| Field | Type | Description |
|-------|------|-------------|
| session_id | MessagePack integer | Session identifier (reserved, 0) |
| row_name | MessagePack string | Fully qualified row name |
| [metadata] | MessagePack-encoded RBusMetadata | Standard metadata with DELETETBLROW method |

**Row Naming Formats:**
- By instance number: "Device.WiFi.AccessPoint.1"
- By alias: "Device.WiFi.AccessPoint.[home_network]"

##### 6.8.2.2 Remove Row Response

**MessagePack-encoded structure:**
```
┌─────────────────────────────────────┐
│  status (MessagePack int)           │
├─────────────────────────────────────┤
│  [RBus Metadata]                    │
│    method_name = "METHOD_RESPONSE"  │
└─────────────────────────────────────┘
```

**Field Descriptions:**

| Field | Type | Description |
|-------|------|-------------|
| status | MessagePack integer | Result status (0=success) |
| [metadata] | MessagePack-encoded RBusMetadata | Standard metadata with RESPONSE method |

### 6.9 Event Message

Events are published to subscribers when values change.

**IMPORTANT:** Event messages use a different metadata structure than other RBus messages.

#### 6.9.1 Event Message Structure

**MessagePack-encoded structure:**
```
┌─────────────────────────────────────┐
│  event_name (MessagePack string)    │
├─────────────────────────────────────┤
│  event_type (MessagePack int) = 3   │
├─────────────────────────────────────┤
│  has_data (MessagePack int)         │
├─────────────────────────────────────┤
│  data (MessagePack RBusObject) [optional]│
├─────────────────────────────────────┤
│  has_filter (MessagePack int) = 0   │
├─────────────────────────────────────┤
│  interval (MessagePack int) = 0     │
├─────────────────────────────────────┤
│  duration (MessagePack int) = 0     │
├─────────────────────────────────────┤
│  component_id (MessagePack int)     │
├─────────────────────────────────────┤
│  [RBus Event Metadata]              │
└─────────────────────────────────────┘
```

**Field Descriptions:**

| Field | Type | Description |
|-------|------|-------------|
| event_name | MessagePack string | Name of the event being published |
| event_type | MessagePack integer | Event type (3 = general event) |
| has_data | MessagePack integer | 1 if event data present, 0 otherwise |
| data | MessagePack-encoded RBusObject | Event data payload (if has_data = 1) |
| has_filter | MessagePack integer | Filter applied flag (reserved, 0) |
| interval | MessagePack integer | Event interval (reserved, 0) |
| duration | MessagePack integer | Event duration (reserved, 0) |
| component_id | MessagePack integer | Publishing component ID |
| [event_metadata] | MessagePack-encoded RBusEventMetadata | Special event metadata |

#### 6.9.2 Event Metadata

**CRITICAL DIFFERENCE:** Events use a different metadata format than other operations.

**MessagePack-encoded structure:**
```
┌─────────────────────────────────────┐
│  event_name (MessagePack string)    │
├─────────────────────────────────────┤
│  object_name (MessagePack string)   │
├─────────────────────────────────────┤
│  is_rbus_2 (MessagePack int) = 1    │
├─────────────────────────────────────┤
│  offset (MessagePack int32, fixed)  │
└─────────────────────────────────────┘
```

| Field | Type | Description |
|-------|------|-------------|
| event_name | MessagePack string | Event name (matches event_name in body) |
| object_name | MessagePack string | Publishing component name |
| is_rbus_2 | MessagePack integer | RBus version flag (always 1) |
| offset | MessagePack int32 (fixed 0xd2) | Byte offset to metadata (MUST use fixed 32-bit encoding) |

### 6.10 Reserved Operations

The following operations are defined in the protocol but are either stubs or reserved for future use.

#### 6.10.1 Commit Transaction

**Method:** `METHOD_COMMIT`  
**Status:** Stub implementation - always returns success

The commit operation is reserved for future transaction support. Currently, it has no effect and immediately returns `RBUS_ERROR_SUCCESS` (0).

**Request:** Empty payload  
**Response:** Status code 0 (success)

#### 6.10.2 Get Parameter Attributes

**Method:** `METHOD_GETPARAMETERATTRIBUTES`  
**Status:** Stub implementation - always returns success

The get attributes operation is reserved for retrieving parameter metadata (e.g., access control, validation rules). Currently, it has no effect and immediately returns `RBUS_ERROR_SUCCESS` (0) with no attributes.

**Request:** Parameter name(s) (format undefined)  
**Response:** Status code 0 (success), empty attribute list

**Note:** For components using the full RBus API, attribute information is available through the element registration system.

#### 6.10.3 Set Parameter Attributes

**Method:** `METHOD_SETPARAMETERATTRIBUTES`  
**Status:** Reserved - not implemented

The set attributes operation is reserved for modifying parameter metadata. This operation is defined but not currently used in RBus implementations.

---

## 7. Data Type Encoding

All RBus data types use MessagePack encoding with specific conventions.

### 7.1 String Encoding

Strings are encoded as MessagePack strings (fixstr, str8, str16, or str32).

**Encoding:**
1. Write MessagePack string header
2. Write UTF-8 bytes
3. **No null terminator** in MessagePack

**Wire Format:**
```
[MessagePack Str Header][UTF-8 Bytes]
```

### 7.2 RBusValue

RBus values use a two-part encoding: type ID followed by value.

#### 7.2.1 Type Identifiers

| RBus Type | Type ID | MessagePack Encoding | Notes |
|-----------|---------|---------------------|-------|
| None | 0x512 | binary(1 byte: 0x00) | Empty placeholder |
| Boolean | 0x500 | binary(1 byte) | 0x00=false, 0x01=true |
| Char | 0x501 | binary(1 byte) | Single character |
| Int8 | 0x503 | binary(1 byte) | Signed byte |
| UInt8 | 0x504 | binary(1 byte) | Unsigned byte |
| Int16 | 0x505 | int | MessagePack integer |
| UInt16 | 0x506 | int | MessagePack integer |
| Int32 | 0x507 | int | MessagePack integer |
| UInt32 | 0x508 | int | MessagePack integer |
| Int64 | 0x509 | int64 (fixed) | MessagePack fixed i64 |
| UInt64 | 0x50A | int64 (fixed) | MessagePack fixed i64 |
| Single | 0x50B | float64 | Promoted to f64 |
| Double | 0x50C | float64 | MessagePack f64 |
| String | 0x50E | binary + null | UTF-8 + 0x00 terminator |
| Bytes | 0x50F | binary | Raw bytes |

#### 7.2.2 Encoding Procedure

**For all types:**
1. Encode type ID as MessagePack integer
2. Encode value according to type rules

**Example: Boolean True**
```
[MessagePack Int: 0x500][MessagePack Bin8: 0xc4 0x01 0x01]
                                          │   │   │
                                          │   │   └─ value: 0x01 (true)
                                          │   └───── length: 1
                                          └───────── bin8 marker
```

**Example: Int32 Value 42**
```
[MessagePack Int: 0x507][MessagePack Int: 42]
```

**Example: String "hello"**
```
[MessagePack Int: 0x50E][MessagePack Bin8: 0xc4 0x06 'h' 'e' 'l' 'l' 'o' 0x00]
                                          │   │    └─────┬─────┘ │
                                          │   │          │       └─ null terminator
                                          │   └────────── length: 6
                                          └────────────── bin8 marker
```

#### 7.2.3 Special Encoding Rules

**Binary Encoding Types:**
The following types encode their value as MessagePack binary (not integer):
- None, Boolean, Char, Int8, UInt8, String, Bytes

**Integer Encoding:**
Int16 through UInt32 use MessagePack variable-length integer encoding.

**Fixed-Width Encoding:**
Int64 and UInt64 MUST use MessagePack fixed 64-bit encoding (0xd3).

**Float Encoding:**
Single values are promoted to double-precision for encoding.

**String Termination:**
String type (0x50E) includes a null terminator byte after the UTF-8 data.

### 7.3 RBusProperty

A property is a name-value pair.

**MessagePack Encoding:**
```
┌─────────────────────────────────────┐
│  name (MessagePack string)          │
├─────────────────────────────────────┤
│  value (MessagePack-encoded RBusValue)│
└─────────────────────────────────────┘
```

**Example:**
```
Property: name="temperature", value=Int32(25)

MessagePack wire bytes:
[MessagePack Str: "temperature"][Type ID: 0x507][MessagePack Int: 25]
```

### 7.4 RBusObject

An object is a named collection of properties.

**MessagePack Encoding:**
```
┌─────────────────────────────────────┐
│  name (MessagePack string)          │
├─────────────────────────────────────┤
│  object_type (MessagePack int) = 0  │
├─────────────────────────────────────┤
│  property_count (MessagePack int)   │
├─────────────────────────────────────┤
│  properties (MessagePack RBusProperty[])│
├─────────────────────────────────────┤
│  children_count (MessagePack int) = 0│
└─────────────────────────────────────┘
```

**Field Descriptions:**

| Field | Type | Description |
|-------|------|-------------|
| name | MessagePack string | Object name identifier |
| object_type | MessagePack integer | Object type (0=single instance) |
| property_count | MessagePack integer | Number of properties that follow |
| properties | MessagePack array | MessagePack-encoded RBusProperty elements |
| children_count | MessagePack integer | Number of child objects (always 0) |

**Example:**
```
Object: name="ThermostatData"
  properties:
    - name="temperature", value=Int32(25)
    - name="humidity", value=Int32(60)

MessagePack wire bytes:
[MessagePack Str: "ThermostatData"]
[MessagePack Int: 0]                    // object_type
[MessagePack Int: 2]                    // property_count
  [MessagePack Str: "temperature"][Type: 0x507][MessagePack Int: 25]
  [MessagePack Str: "humidity"][Type: 0x507][MessagePack Int: 60]
[MessagePack Int: 0]                    // children_count
```

---

## 8. Status Codes

### 8.1 Success Codes

| Code | Symbol | Meaning |
|------|--------|---------|
| 0 | RBUS_ERROR_SUCCESS | Operation completed successfully |
| 100 | (Alternative Success) | Operation completed successfully |

**Note:** Both 0 and 100 are treated as success. Check for `status == 0 || status == 100`.

### 8.2 Error Codes

Error codes align with the RBus API error enumeration:

| Code | Symbol | Description |
|------|--------|-------------|
| 1 | RBUS_ERROR_BUS_ERROR | General bus communication error |
| 2 | RBUS_ERROR_INVALID_INPUT | Invalid input parameters |
| 3 | RBUS_ERROR_NOT_INITIALIZED | RBus not initialized |
| 4 | RBUS_ERROR_OUT_OF_RESOURCES | System resources exhausted |
| 5 | RBUS_ERROR_DESTINATION_NOT_FOUND | Target element not found |
| 6 | RBUS_ERROR_DESTINATION_NOT_REACHABLE | Target not accessible |
| 7 | RBUS_ERROR_DESTINATION_RESPONSE_FAILURE | Target failed to respond |
| 8 | RBUS_ERROR_INVALID_RESPONSE_FROM_DESTINATION | Malformed response |
| 9 | RBUS_ERROR_INVALID_OPERATION | Operation not supported |
| 10 | RBUS_ERROR_INVALID_EVENT | Invalid event registration |
| 11 | RBUS_ERROR_INVALID_HANDLE | Invalid RBus handle |
| 12 | RBUS_ERROR_SESSION_ALREADY_EXIST | Session ID conflict |
| 13 | RBUS_ERROR_COMPONENT_NAME_DUPLICATE | Component name in use |
| 14 | RBUS_ERROR_ELEMENT_NAME_DUPLICATE | Element already registered |
| 15 | RBUS_ERROR_ELEMENT_NAME_MISSING | No element name provided |
| 16 | RBUS_ERROR_COMPONENT_DOES_NOT_EXIST | Component not registered |
| 17 | RBUS_ERROR_ELEMENT_DOES_NOT_EXIST | Element not registered |
| 18 | RBUS_ERROR_ACCESS_NOT_ALLOWED | Permission denied |
| 19 | RBUS_ERROR_INVALID_CONTEXT | Invalid callback context |
| 20 | RBUS_ERROR_TIMEOUT | Operation timed out |
| 21 | RBUS_ERROR_ASYNC_RESPONSE | Asynchronous response pending |
| 22 | RBUS_ERROR_INVALID_METHOD | Method not found |
| 23 | RBUS_ERROR_NOSUBSCRIBERS | No active subscribers |
| 24 | RBUS_ERROR_SUBSCRIPTION_ALREADY_EXIST | Duplicate subscription |
| 25 | RBUS_ERROR_INVALID_NAMESPACE | Namespace validation failed |
| 26 | RBUS_ERROR_DIRECT_CON_NOT_EXIST | Direct connection unavailable |
| 27 | RBUS_ERROR_NOT_WRITABLE | Element is read-only |
| 28 | RBUS_ERROR_NOT_READABLE | Element is write-only |
| 29 | RBUS_ERROR_INVALID_PARAMETER_TYPE | Type mismatch |
| 30 | RBUS_ERROR_INVALID_PARAMETER_VALUE | Value out of range |

---

## 9. Security Considerations

### 9.1 Access Control

- Socket permissions on `/tmp/rtrouted` control access
- No built-in authentication mechanism
- Relies on Unix file permissions for security

### 9.2 Data Validation

Implementations MUST validate:
- Header markers and version
- Field lengths against maximum limits
- MessagePack structure integrity
- UTF-8 string encoding validity

### 9.3 Resource Limits

To prevent denial-of-service:
- Topic length ≤ 256 bytes
- Reply topic length ≤ 256 bytes
- Payload length should be bounded (implementation-specific)
- Message rate limiting recommended

### 9.4 Encryption

The Encrypted flag (0x20) is defined but:
- Encryption format is implementation-specific
- No standard encryption scheme defined
- End-to-end encryption must be implemented separately

---

## 10. Appendix

### 10.1 MessagePack Quick Reference

| Type | First Byte | Description |
|------|-----------|-------------|
| fixint | 0x00-0x7f | Positive integer 0-127 |
| fixstr | 0xa0-0xbf | String length 0-31 |
| nil | 0xc0 | Null value |
| false | 0xc2 | Boolean false |
| true | 0xc3 | Boolean true |
| bin8 | 0xc4 | Binary, 1-byte length |
| bin16 | 0xc5 | Binary, 2-byte length |
| bin32 | 0xc6 | Binary, 4-byte length |
| float64 | 0xcb | 64-bit IEEE 754 float |
| uint8 | 0xcc | 8-bit unsigned |
| uint16 | 0xcd | 16-bit unsigned |
| uint32 | 0xce | 32-bit unsigned |
| int8 | 0xd0 | 8-bit signed |
| int16 | 0xd1 | 16-bit signed |
| int32 | 0xd2 | 32-bit signed |
| int64 | 0xd3 | 64-bit signed |
| str8 | 0xd9 | String, 1-byte length |
| str16 | 0xda | String, 2-byte length |
| str32 | 0xdb | String, 4-byte length |

### 10.2 Example Message Flows

#### 10.2.1 Get Property Flow

```
1. Consumer → Router (RTMessage)
   Topic: "Device.Temperature"
   Reply Topic: "rbus.consumer.INBOX.001"
   Flags: 0x11 (Request + RawBinary)
   Payload: [Get Request]

2. Router → Provider (RTMessage)
   Topic: "Device.Temperature"
   Reply Topic: "rbus.consumer.INBOX.001"
   Flags: 0x11 (Request + RawBinary)
   Payload: [Get Request]

3. Provider → Router (RTMessage)
   Topic: "rbus.consumer.INBOX.001"
   Reply Topic: ""
   Flags: 0x12 (Response + RawBinary)
   Payload: [Get Response with value]

4. Router → Consumer (RTMessage)
   Topic: "rbus.consumer.INBOX.001"
   Reply Topic: ""
   Flags: 0x12 (Response + RawBinary)
   Payload: [Get Response with value]
```

#### 10.2.2 Event Publication Flow

```
1. Consumer → Router (RTMessage - Subscribe Control)
   Topic: "_RTROUTED.INBOX.SUBSCRIBE"
   Payload: {"add": 1, "topic": "Device.Temperature", ...}

2. Consumer → Router (RTMessage - Subscribe Request)
   Topic: "Device.Temperature"
   Reply Topic: "rbus.consumer.INBOX.001"
   Flags: 0x11 (Request + RawBinary)
   Payload: [Subscribe Request]

3. Router → Provider (RTMessage)
   (Subscribe routed to provider)

4. Provider → Router (RTMessage - Subscribe Response)
   (Acknowledgment)

5. Provider → Router (RTMessage - Event)
   Topic: "rbus.consumer.INBOX.001"
   Flags: 0x12 (Response + RawBinary)
   Payload: [Event Message with data]

6. Router → Consumer (RTMessage - Event)
   Topic: "rbus.consumer.INBOX.001"
   Flags: 0x12 (Response + RawBinary)
   Payload: [Event Message with data]
```

#### 10.2.3 Add Table Row Flow

```
1. Consumer → Router (RTMessage)
   Topic: "Device.WiFi.AccessPoint."
   Reply Topic: "rbus.consumer.INBOX.002"
   Flags: 0x11 (Request + RawBinary)
   Payload: [Add Row Request with alias="home_network"]

2. Router → Provider (RTMessage)
   Topic: "Device.WiFi.AccessPoint."
   Reply Topic: "rbus.consumer.INBOX.002"
   Flags: 0x11 (Request + RawBinary)
   Payload: [Add Row Request with alias="home_network"]

3. Provider creates row with instance number 1

4. Provider → Router (RTMessage)
   Topic: "rbus.consumer.INBOX.002"
   Reply Topic: ""
   Flags: 0x12 (Response + RawBinary)
   Payload: [Add Row Response with status=0, instance_number=1]

5. Router → Consumer (RTMessage)
   Topic: "rbus.consumer.INBOX.002"
   Reply Topic: ""
   Flags: 0x12 (Response + RawBinary)
   Payload: [Add Row Response with status=0, instance_number=1]

6. Consumer can now access row as:
   - "Device.WiFi.AccessPoint.1" (by instance)
   - "Device.WiFi.AccessPoint.[home_network]" (by alias)
```

#### 10.2.4 Remove Table Row Flow

```
1. Consumer → Router (RTMessage)
   Topic: "Device.WiFi.AccessPoint.1"
   Reply Topic: "rbus.consumer.INBOX.003"
   Flags: 0x11 (Request + RawBinary)
   Payload: [Remove Row Request]

2. Router → Provider (RTMessage)
   Topic: "Device.WiFi.AccessPoint.1"
   Reply Topic: "rbus.consumer.INBOX.003"
   Flags: 0x11 (Request + RawBinary)
   Payload: [Remove Row Request]

3. Provider deletes row instance 1

4. Provider → Router (RTMessage)
   Topic: "rbus.consumer.INBOX.003"
   Reply Topic: ""
   Flags: 0x12 (Response + RawBinary)
   Payload: [Remove Row Response with status=0]

5. Router → Consumer (RTMessage)
   Topic: "rbus.consumer.INBOX.003"
   Reply Topic: ""
   Flags: 0x12 (Response + RawBinary)
   Payload: [Remove Row Response with status=0]
```

#### 10.2.5 Discovery Flow

```
1. Consumer → Router (RTMessage)
   Topic: "Device.WiFi."
   Reply Topic: "rbus.consumer.INBOX.004"
   Flags: 0x11 (Request + RawBinary)
   Payload: [Get Parameter Names Request]
     - object_name: "Device.WiFi."
     - depth: 1 (one level deep)
     - get_row_names_only: 0

2. Router → Provider (RTMessage)
   Topic: "Device.WiFi."
   Reply Topic: "rbus.consumer.INBOX.004"
   Flags: 0x11 (Request + RawBinary)
   Payload: [Get Parameter Names Request]

3. Provider enumerates child elements

4. Provider → Router (RTMessage)
   Topic: "rbus.consumer.INBOX.004"
   Reply Topic: ""
   Flags: 0x12 (Response + RawBinary)
   Payload: [Get Parameter Names Response]
     - status: 0
     - count: 3
     - Element 1: {name: "Device.WiFi.SSID", type: 0, access: 3}
     - Element 2: {name: "Device.WiFi.Enable", type: 0, access: 3}
     - Element 3: {name: "Device.WiFi.AccessPoint.", type: 1, access: 0}

5. Router → Consumer (RTMessage)
   Topic: "rbus.consumer.INBOX.004"
   Reply Topic: ""
   Flags: 0x12 (Response + RawBinary)
   Payload: [Get Parameter Names Response]
```

### 10.3 Compliance Checklist

Implementations MUST:
- [ ] Validate both 0xAAAA header markers
- [ ] Use version 2 in all messages
- [ ] Encode all integers in big-endian
- [ ] Set RawBinary flag for MessagePack payloads
- [ ] Encode metadata offset as fixed 32-bit integer
- [ ] Include null terminator in String type (0x50E) values
- [ ] Validate topic lengths ≤ 256 bytes
- [ ] Support all defined RBusValue types
- [ ] Handle both status codes 0 and 100 as success

Implementations SHOULD:
- [ ] Implement connection retry logic
- [ ] Rate-limit message sending
- [ ] Validate MessagePack structure
- [ ] Log malformed messages for debugging
- [ ] Implement graceful shutdown
- [ ] Clean up subscriptions on disconnect

### 10.4 References

- **RBus GitHub Repository**: https://github.com/rdkcentral/rbus
- **MessagePack Specification**: https://github.com/msgpack/msgpack/blob/master/spec.md
- **JSON Specification**: RFC 8259
- **Unix Domain Sockets**: POSIX.1-2001

### 10.5 Version History

| Version | Date | Changes |
|---------|------|---------|
| 2.0 | 2026-01 | Initial specification based on rbus implementation |

---

**END OF SPECIFICATION**
