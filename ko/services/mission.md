# Mission Protocol

The mission sub-protocol allows a GCS or developer API to manage *mission* (flight plan), *geofence* and *safe point* information on a drone/component.

The protocol covers:

* Operations to upload, download and clear missions, set/get the current mission item number, and get notification when the current mission item has changed.
* Message type(s) for exchanging mission items.
* MAVLink commands that are common to most autpilots/GCS.

The protocol follows the client/server pattern, where operations (and most commands) are initiated by the GCS/developer API (client) and acknowledged by the autopilot (server).

The protocol supports re-request of messages that have not arrived, allowing missions to be reliably transferred over a lossy link. <!-- not quite guaranteed :-) -->

## Mission Types {#mission_types}

MAVLink 1 supports only "regular" flight-plan missions. MAVLink 2 supports three types of "missions": flight plans, geofence and rally/safe Points (the vehicle must store and act on these separately).

The mission type is specified in a MAVLink 2 message extension field: `mission_type` using one of the [MAV_MISSION_TYPE](../messages/common.md#MAV_MISSION_TYPE) enum values:

* [MAV_MISSION_TYPE_MISSION](../messages/common.md#MAV_MISSION_TYPE_MISSION)
* [MAV_MISSION_TYPE_FENCE](../messages/common.md#MAV_MISSION_TYPE_FENCE)
* [MAV_MISSION_TYPE_RALLY](../messages/common.md#MAV_MISSION_TYPE_RALLY)

> **Note** You set the `mission_type` in messages: [MISSION_COUNT](../messages/common.md#MISSION_COUNT), [MISSION_REQUEST](../messages/common.md#MISSION_REQUEST), [MISSION_ITEM_INT](../messages/common.md#MISSION_ITEM_INT), [MISSION_CLEAR_ALL](../messages/common.md#MISSION_CLEAR_ALL), etc.

The protocol uses the same sequence of operations for all types (albeit with different MAVLink commands).

## MAVLink Commands {#mavlink_commands}

MAVLink commands are defined in the [MAV_CMD](../messages/common.md#MAV_CMD) enum.

> **Note**

* Some commands can be sent outside of a mission context using the [Command Protocol](../services/command.md). 
* Not all commands in `MAV_CMD` are supported by all systems (or all flight modes).

The *mission* commands are broadly divided into three groups.

* NAV commands (`MAV_CMD_NAV_*`) for navigation/movement - e.g. setting waypoints, taking off/landing, etc.
* DO commands (`MAV_CMD_DO_*`) for immediate actions like changing speed or activating a servo.
* CONDITION commands (`MAV_CMD_CONDITION_*`) for changing the execution of the mission based on a condition - e.g. pausing the mission for a time before executing next command.

The commands are transmitted/encoded in [MISSION_ITEM](../messages/common.md#MISSION_ITEM) or [MISSION_ITEM_INT](../messages/common.md#MISSION_ITEM_INT) messages. These messages include fields to identify the desired command (command id) and up to 7 command-specific parameters. The first four parameters can be used for any purpose (depends on the particular [command](../messages/common.md#MAV_CMD)). The last three parameters (x, y, z) are used for positional information in NAV commands, but can be used for any purpose in other commands.

The command-specific fields in the messages are shown below:

| Field Name | Type            | Values                                   | Description                                                                                                      |
| ---------- | --------------- | ---------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| command    | uint16_t        | [MAV_CMD](../messages/common.md#MAV_CMD) | Command id, as defined in [MAV_CMD](../messages/common.md#MAV_CMD).                                              |
| param1     | float           |                                          | Param #1.                                                                                                        |
| param2     | float           |                                          | Param #2.                                                                                                        |
| param3     | float           |                                          | Param #3.                                                                                                        |
| param4     | float           |                                          | Param #4.                                                                                                        |
| x          | float / int32_t |                                          | X coordinate (local frame) or latitude (global frame) for navigation commands (otherwise Param #5).              |
| y          | float / int32_t |                                          | Y coordinate (local frame) or longitude (global frame) for navigation commands (otherwise Param #6).             |
| z          | float           |                                          | Z coordinate (local frame) or altitude (global - relative or absolute, depending on frame) (otherwise Param #7). |

The remaining message fields are used for addressing, defining the mission type, specifying the frame used for x, y, z in NAV messages, etc.:

| Field Name       | Type     | Values             | Description                                                                                                                                                                                          |
| ---------------- | -------- | ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| target_system    | uint8_t  |                    | System ID                                                                                                                                                                                            |
| target_component | uint8_t  |                    | Component ID                                                                                                                                                                                         |
| seq              | uint16_t |                    | Sequence number for                                                                                                                                                                                  |
| frame            | uint8_t  | MAV_FRAME          | The coordinate system of the waypoint.  
ArduPilot and PX4 both only support global frames in MAVLink commands (local frames may be supported if the same command is sent via the command protocol). |
| mission_type     | uint8_t  | MAV_MISSION_TYPE | [Mission type](#mission_types).                                                                                                                                                                      |
| current          | uint8_t  | false:0, true:1    | When downloading, whether the item is the current mission item.                                                                                                                                      |
| autocontinue     | uint8_t  |                    | Autocontinue to next waypoint when the command completes.                                                                                                                                            |

## Operations {#operations}

This section explains the main operations defined by the protocol.

### Upload a Mission to the Vehicle {#uploading_mission}

The diagram below shows the communication sequence to upload a mission to a drone (assuming all operations succeed).

{% mermaid %} sequenceDiagram; participant GCS participant Drone GCS->>Drone: MISSION_COUNT GCS->>GCS: Start timeout Drone->>GCS: MISSION_REQUEST_INT (0) GCS->>GCS: Start timeout GCS-->>Drone: MISSION_ITEM_INT (0) Drone->>GCS: MISSION_REQUEST_INT (1) GCS->>GCS: Start timeout GCS-->>Drone: MISSION_ITEM_INT (1) Drone->>GCS: MISSION_ACK {% endmermaid %}

Note:

* The GCS (client) sets a [timeout](#timeout) after every message and will resend if there is no response from the vehicle.
* The client will re-request mission items that are received out of sequence.
* The sequence above shows the [MAVLink commands](#mavlink_commands) packaged in [MISSION_ITEM_INT](../messages/common.md#MISSION_ITEM_INT) messages. Protocol implementations must also support [MISSION_ITEM](../messages/common.md#MISSION_ITEM) and [MISSION_REQUEST](../messages/common.md#MISSION_REQUEST) in the same way (see [MISSION_ITEM_INT vs MISSION_ITEM below](#command_message_type)).

In more detail, the sequence of operations is:

1. GCS (client) sends [MISSION_COUNT](../messages/common.md#MISSION_COUNT) including the number of mission items to be uploaded (`count`) 
    * A [timeout](#timeout) must be started for the GCS to wait on the response from Drone (`MISSION_REQUEST_INT`) .
2. Drone (server) receives the message, and prepares to upload mission items.
3. Drone responds with [MISSION_REQUEST_INT](../messages/common.md#MISSION_REQUEST_INT) requesting the mission number to upload (`seq`)
4. GCS receives `MISSION_REQUEST_INT` and responds with the requested mission item in a [MISSION_ITEM_INT](../messages/common.md#MISSION_ITEM_INT) message.
5. Drone and GCS repeat the `MISSION_REQUEST_INT`/`MISSION_ITEM_INT` cycle until all items are uploaded.
6. For the last mission item, the drone responds with [MISSION_ACK](../messages/common.md#MISSION_ACK) with the result of the operation result: `type` ([MAV_MISSION_RESULT](../messages/common.md#MAV_MISSION_RESULT)): 
    * On success, `type` must be set to [MAV_MISSION_ACCEPTED](../messages/common.md#MAV_MISSION_ACCEPTED)
    * On failure, `type` must set to [MAV_MISSION_ERROR](../messages/common.md#MAV_MISSION_ERROR) or some other error code. 
7. GCS receives `MISSION_ACK`: 
    * If `MAV_MISSION_ACCEPTED` the operation is complete.
    * If an error, the transaction fails but may be retried. <!-- not clear here -->

### Download a Mission from the Vehicle

The diagram below shows the communication sequence to download a mission from a drone (assuming all operations succeed).

{% mermaid %} sequenceDiagram; participant GCS participant Drone GCS->>Drone: MISSION_REQUEST_LIST GCS->>GCS: Start timeout Drone-->>GCS: MISSION_COUNT GCS->>Drone: MISSION_REQUEST_INT (0) GCS->>GCS: Start timeout Drone-->>GCS: MISSION_ITEM_INT (0) GCS->>Drone: MISSION_REQUEST_INT (1) GCS->>GCS: Start timeout Drone-->>GCS: MISSION_ITEM_INT (1) GCS->>Drone: MISSION_ACK {% endmermaid %}

> **Note** The sequence shows the [MAVLink commands](#mavlink_commands) packaged in [MISSION_ITEM_INT](../messages/common.md#MISSION_ITEM_INT) messages. Protocol implementations must also support [MISSION_ITEM](../messages/common.md#MISSION_ITEM) and [MISSION_REQUEST](../messages/common.md#MISSION_REQUEST) in the same way (see [MISSION_ITEM_INT vs MISSION_ITEM below](#command_message_type)).

The sequence is similar to that for [uploading a mission](#uploading_mission). The main difference is that the client (e.g. GCS) sends [MISSION_REQUEST_LIST](../messages/common.md#MISSION_REQUEST_LIST), which triggers the autopilot to respond with the current count of items. This starts a cycle where the GCS requests mission items, and the drone supplies them.

### Set Current Mission Item {#current_mission_item}

The diagram below shows the communication sequence to set the current mission item.

{% mermaid %} sequenceDiagram; participant GCS participant Drone GCS->>Drone: MISSION_SET_CURRENT Drone-->>GCS: MISSION_CURRENT {% endmermaid %}

Notes:

* There is no specific timeout/resend on this message.
* The acknowledgment of the message is via broadcast of mission/system status, which is not associated with the original message. This approach is used because the message is relevant to all mission-handling clients.

In more detail, the sequence of operations is:

1. GCS (client) sends [MISSION_SET_CURRENT](../messages/common.md#MISSION_SET_CURRENT), specifying the new sequence number (`seq`). 
2. Drone (server) receives message and attempts to update the current mission sequence number. 
    * On success, the Drone should *broadcast* a [MISSION_CURRENT](../messages/common.md#MISSION_CURRENT) message containing the current sequence number (`seq`). 
    * On failure, the Drone should *broadcast* a [STATUSTEXT](../messages/common.md#STATUSTEXT) with a [MAV_SEVERITY](../messages/common.md#MAV_SEVERITY) and a string stating the problem. This may be displayed in the UI of receiving systems.

### Monitor Mission Progress

GCS/developer API can monitor progress by handling the appropriate messages sent by the drone:

* The vehicle (server) must broadcast a [MISSION_ITEM_REACHED](../messages/common.md#MISSION_ITEM_REACHED) message whenever a new mission item is reached. The message contains the `seq` number of the current mission item.
* The vehicle must also broadcast a [MISSION_CURRENT](../messages/common.md#MISSION_CURRENT) message if the [current mission item](#current_mission_item) is changed by a message.

### Clear Missions

The diagram below shows the communication sequence to clear the mission from a drone (timeouts are not displayed, and we assume all operations succeed).

{% mermaid %} sequenceDiagram; participant GCS participant Drone GCS->>Drone: MISSION_CLEAR_ALL GCS->>GCS: Start timeout Drone-->>GCS: MISSION_ACK {% endmermaid %}

In more detail, the sequence of operations is:

1. GCS (client) sends [MISSION_CLEAR_ALL](../messages/common.md#MISSION_CLEAR_ALL) 
    * A [timeout](#timeout) is started for the GCS to wait on `MISSION_ACK` from Drone.
2. Drone (server) receives the message, and clears the mission. 
    * A mission is considered cleared if a subsequent requests for mission count or current mission item indicates that there is no mission uploaded.
3. Drone responds with [MISSION_ACK](../messages/common.md#MISSION_ACK) that includes the result `type` ([MAV_MISSION_RESULT](../messages/common.md#MAV_MISSION_RESULT)): 
    * On success, this type must be set to [MAV_MISSION_ACCEPTED](../messages/common.md#MAV_MISSION_ACCEPTED)
    * On failure, the type must set to [MAV_MISSION_ERROR](../messages/common.md#MAV_MISSION_ERROR) or some other error code. 
4. GCS receives `MISSION_ACK`: 
    * If `MAV_MISSION_ACCEPTED` the GCS clears its own stored information about the mission (that was just removed from the vehicle) and completes.
    * If an error, the transaction fails, and the GCS record of the mission (if any) is retained.
5. If no `MISSION_ACK` is received the operation will eventually timeout and may be retried (see [above](#timeout)).

### Timeouts and Retries {#timeout}

All the client (GCS) commands are sent with a timeout. If a `MISSION_ACK` is not received before the timeout then the client (GCS) may resend the message. <!-- really guaranteed? What about partial-states in vehicle --> If no response is received after a number of retries then the client must cancel the operation and return to an idle state.

The recommended timeout values before resending, and the number of retries are:

* Timeout (default): 1500 ms
* Timeout (mission items): 250 ms.
* Retries (max): 5

### MISSION_ITEM_INT vs MISSION_ITEM {#command_message_type}

The operations/sequence diagrams above show the [message commands](#message_commands) being requested/sent using [MISSION_REQUEST_INT](../messages/common.md#MISSION_REQUEST_INT) and [MISSION_ITEM_INT](../messages/common.md#MISSION_ITEM_INT).

Protocol implementations must also support the same operations/sequences using the corresponding [MISSION_REQUEST](../messages/common.md#MISSION_REQUEST) and [MISSION_ITEM](../messages/common.md#MISSION_ITEM) message types. The only difference is that `MISSION_ITEM_INT` encodes the latitude and longitude as integers rather than floats.

> **Tip** MAVLink *users* should always prefer the `*_INT` variants. These avoid/reduce the precision limitations from using `MISSION_ITEM`.

## Mission File Formats

The *defacto* standard file format for exchanging missions/plans is discussed in: [File Formats > Mission Plain-Text File Format](../file_formats/README.md#mission_plain_text_file).

## C Implementation

The protocol has been implemented in C by PX4 <!-- and ArduPilot Flight Stacks, --> and 

*QGroundControl*.  
This implementation can be used in your own code within the terms of their software licenses.

PX4 Implementation:

* [src/modules/mavlink/mavlink_mission.cpp](https://github.com/PX4/Firmware/blob/master/src/modules/mavlink/mavlink_mission.cpp)

QGroundControl* implementation:

* [src/MissionManager/PlanManager.cc](https://github.com/mavlink/qgroundcontrol/blob/master/src/MissionManager/PlanManager.cc)

ArduPilot

* TBD