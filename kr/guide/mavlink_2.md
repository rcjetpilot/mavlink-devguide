# MAVLink 2

*MAVLink 2* is a backward-compatible update to the MAVLink protocol that has been designed to bring more flexibility and security to MAVLink communication.

The key new features of *MAVLink 2* are:
* More than 256 message IDs (24 bit message ID - over 16 million packets)
* [Packet signing](../guide/message_signing.md) (authentication)
* [Extending existing MAVLink messages](#message_extensions)
* [Variable length arrays](#variable_arrays) (*simplified implementation*).

*MAVLink 2* bindings have been developed for C, C++11 and Python (see [Supported Languages](../README.md#supported_languages)).

> **Tip** We recommend you start by reading the *MAVLink 2* [design document](https://docs.google.com/document/d/1XtbD0ORNkhZ8eKrsbSIZNLyg9sFRXMXbsR2mp37KbIg/edit?usp=sharing)


## Upgrading an Existing C Installation

The existing *MAVLink 1* pre-built library [mavlink/c_library_v1](https://github.com/mavlink/c_library_v1) can be upgraded by simply dropping in the *MAVLink 2* library from Github: [mavlink/c_library_v2](https://github.com/mavlink/c_library_v2)

*MAVLink 2* usage is then covered in the following section.


## Using the C Implementation

For most users usage of the pre-generated C headers is recommended: https://github.com/mavlink/c_library_v2

These headers offer the same range of APIs as was offered by MAVLink1. 

The major changes from an API perspective are:

* you don't need to provide a message CRC table any more, or message length table. 
  These have been folded into a single packed table, replacing the old table which was indexed by `msgId`. 
  That was necessary to cope with the much larger 24 bit namespace of message IDs.


### Version Handshaking/Negotiation

[MAVLink Versions](../guide/mavlink_version.md) explains the [handshaking](../guide/mavlink_version.md#version_handshaking) used to determine the supported MAVLink version of either end of the channel, and how to [negotiate the version to use](#negotiating_versions).


### Sending and Receiving MAVLink 1 Packets


The *MAVLink 2* library will send packets in *MAVLink 2* framing by default. 
To force sending *MAVLink 1* packets on a particular channel you change the flags field of the status object. 

For example, the following code causes subsequent packets on the given channel to be sent as *MAVLink 1*:

```C
mavlink_status_t* chan_state = mavlink_get_channel_status(MAVLINK_COMM_0);
chan_state->flags |= MAVLINK_STATUS_FLAG_OUT_MAVLINK1;
```


Incoming *MAVLink 1* packets will be automatically handled as *MAVLink 1*. If you need to determine if a particular message was received as *MAVLink 1* or *MAVLink 2* then you can use the magic field of the message, like this:

```c
if (msg->magic = MAVLINK_STX_MAVLINK1) {
   printf("This is a MAVLink 1 message\n");
}
```

In most cases this should not be necessary as the XML message definition files for *MAVLink 1* and *MAVLink 2* are the same, so you can treat incoming *MAVLink 1* messages the same as *MAVLink 2* messages.

> **Note** *MAVLink 1* is restricted to messageIDs less than 256, so any messages with a higher messageID won't be received as *MAVLink 1*.


It is advisable to switch to *MAVLink 2* when the communication partner sends *MAVLink 2* (see [Version Handshaking](../guide/mavlink_version.md#version_handshaking)). The minimal solution is to watch incoming packets using code similar to this:

```C
if (mavlink_parse_char(MAVLINK_COMM_0, buf[i], &msg, &status)) {

	// check if we received version 2 and request a switch.
	if (!(mavlink_get_channel_status(MAVLINK_COMM_0)->flags & MAVLINK_STATUS_FLAG_IN_MAVLINK1)) {
		// this will only switch to proto version 2
		chan_state->flags &= ~(MAVLINK_STATUS_FLAG_OUT_MAVLINK1);
	}
}
```



## Message Extensions {#message_extensions}

The XML for messages can contain extensions that are optional in the protocol. 
This allows for extra fields to be added to a message.

The rules for extended messages are:

* Extended fields are not sent in *MAVLink 1* messages. 
* If received by an implementation that doesn't have the extended fields then the fields will not be seen.
* If sent by an implementation that doesn't have the extended fields then the recipient will see zero values for the extended fields.

For example the fields after the `<extensions>` line below are extended fields:

```xml
    <message id="100" name="OPTICAL_FLOW">
      <description>Optical flow from a flow sensor (e.g. optical mouse sensor)</description>
      <field type="uint64_t" name="time_usec" units="us">Timestamp (UNIX)</field>
      <field type="uint8_t" name="sensor_id">Sensor ID</field>
      <field type="int16_t" name="flow_x" units="dpixels">Flow in pixels * 10 in x-sensor direction (dezi-pixels)</field>
      <field type="int16_t" name="flow_y" units="dpixels">Flow in pixels * 10 in y-sensor direction (dezi-pixels)</field>
      <field type="float" name="flow_comp_m_x" units="m">Flow in meters in x-sensor direction, angular-speed compensated</field>
      <field type="float" name="flow_comp_m_y" units="m">Flow in meters in y-sensor direction, angular-speed compensated</field>
      <field type="uint8_t" name="quality">Optical flow quality / confidence. 0: bad, 255: maximum quality</field>
      <field type="float" name="ground_distance" units="m">Ground distance in meters. Positive value: distance known. Negative value: Unknown distance</field>
      <extensions/>
      <field type="float" name="flow_rate_x" units="rad/s">Flow rate in radians/second about X axis</field>
      <field type="float" name="flow_rate_y" units="rad/s">Flow rate in radians/second about Y axis</field>
    </message>
```


## Message/Packet Signing

Authentication is covered in the topic: [Message signing](../guide/message_signing.md) (authentication)


## Variable Length Arrays {#variable_arrays}

*MAVLink 2* truncates any zero (empty) elements at the end of the *largest* array in a message. Other arrays in the message will not be truncated.

> **Note** This is a "light" implementation of variable length arrays. For messages with 0 or 1 arrays (the most common case) this is the same as a full implementation.