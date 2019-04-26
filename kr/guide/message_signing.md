# Message Signing (Authentication)

> **Note** This topic covers message signing in [MAVLink 2](../guide/mavlink_2.md). MAVLink 1 does not support message signing.

One of the key features of *MAVLink 2* is support for signing of messages. To enable signing in your application you will need to add some additional code. In particular you will need to add:

* Code to handle the SETUP_SIGNING message
* Code to setup and teardown signing on a link
* Code to save and load the secret key and timestamp in persistent storage
* A callback to allow for accepting of certain kinds of unsigned messages

## Handling SETUP_SIGNING

The [SETUP_SIGNING](../messages/common.md#SETUP_SIGNING) message is the mechanism for a GCS to setup a signing key on a *MAVLink 2* device. It takes a 32 byte secret key and an initial timestamp. The method of generating the 32 byte secret key is up to the GCS implementation, although it is suggested that all GCS implementations should support the use of a sha256 hash of a user provided passphrase.

## Handling Timestamps

The timestamp is a 64 bit number, and is in units of 10 microseconds since 1st January 2015. For systems where the time since 1/1/1970 is available (the unix epoch) you can use an offset in seconds of 1420070400.

Storage and handling of the timestamp is critical to the security of the signing system. The rules are:

* The current timestamp should be stored regularly in persistent storage (suggested at least once a minute)
* The timestamp used on startup should be the maximum of the timestamp implied by the system clock and the stored timestamp
* If the system does not have a RTC mechanism then it should update its timestamp when GPS lock is achieved. The maximum of the timestamp from the GPS and the stored timestamp should be used
* The timestamp should be incremented by one on each message send. This is done for you by the generated headers.
* When a correctly signed message is decoded the timestamp should be replaced by the timestamp of the incoming message if that timestamp is greater than the current timestamp. This is done for you by the generated headers
* The timestamp on incoming signed messages should be checked against the previous timestamp for the incoming `(linkID,srcSystem,SrcComponent)` tuple and the message rejected if it is smaller. This is done for you by generated headers.
* If there is no previous message with the given `(linkID,srcSystem,SrcComponent)` then the timestamp should be accepted if it not more than 6 million (one minute) behind the current timestamp

## Enabling Signing on a Channel

To enable signing on a channel you need to fill in two pointers in the status structure for the channel. The two pointed are:

```
mavlink_signing_t *signing;
mavlink_signing_streams_t *signing_streams;
```

The signing pointer controls signing for this stream. It is per-stream, and contains the secret key, the timestamp and a set of flags, plus an optional callback function for accepting unsigned packets. Typical setup would be:

```
memcpy(signing.secret_key, key.secret_key, 32);
signing.link_id = (uint8_t)chan;
signing.timestamp = key.timestamp;
signing.flags = MAVLINK_SIGNING_FLAG_SIGN_OUTGOING;
signing.accept_unsigned_callback = accept_unsigned_callback;
mavlink_status_t *status = mavlink_get_channel_status(chan);
status.signing = &signing;
```

The `signing_streams pointer` is a structure used to record the previous timestamp for a `(linkId,srcSystem,SrcComponent)` tuple. This must point to a structure that is common to all channels in order to prevent inter-channel replay attacks. Typical setup is:

```
mavlink_status_t *status = mavlink_get_channel_status(chan);
status.signing_streams = &signing_streams;
```

The maximum number of signing streams supported is given by the `MAVLINK_MAX_SIGNING_STREAMS` macro. This defaults to 16, but it may be worth raising this for GCS implementations. If the C implementation runs out of signing streams then new streams will be rejected.

## Using the accept_unsigned_callback

In the signing structure there is an optional `accept_unsigned_callback` function pointer. The C prototype for this function is:

```
bool accept_unsigned_callback(const mavlink_status_t *status, uint32_t msgId);
```

If set in the signing structure then this function will be called on any unsigned packet (including all *MAVLink 1* packets) or any packet where the signature is incorrect. The function offers a way for the implementation to allow unsigned packets to be accepted.

The rules for what unsigned packets should be accepted is implementation specific, but it is suggested the following rules be considered:

* have a mechanism for marking a particular communication channel as being secure (such as a USB connection) to allow for signing setup.
* always accept `RADIO_STATUS` packets for feedback from 3DR radios (which don't do signing)

For example:

```c
static const uint32_t unsigned_messages[] = {
	MAVLINK_MSG_ID_RADIO_STATUS
};

static bool accept_unsigned_callback(const mavlink_status_t *status, uint32_t message_id)
{
	// Make the assumption that channel 0 is USB and should always be accessible
	if (status == mavlink_get_channel_status(MAVLINK_COMM_0)) {
		return true;
	}

	for (unsigned i = 0; i < sizeof(unsigned_messages) / sizeof(unsigned_messages[0]); i++) {
		if (unsigned_messages[i] == message_id) {
			return true;
		}
	}

	return false;
}
```

## Handling Link IDs

The purpose of the `link_id` field in the *MAVLink 2* signing structure is to prevent cross-channel replay attacks. Without the `link_id` an attacker could record a packet (such as a disarm request) on one channel, then play it back on a different channel.

The intention with the link IDs is that each channel of communication between an autopilot and a GCS uses a different link ID. There is no requirement that the same link ID be used in both directions however.

For C implementations the obvious mechanism is to use the MAVLink channel number as the link ID. That works well for an autopilot, but runs into an issue for a GCS implementation. The issue is that a user may launch multiple GCS instances talking to the same autopilot via different communication links (such as two radios, or USB and a radio). These multiple GCS instances will not be aware of each other, and so may choose the same link ID. If that happens then a large number of correctly signed packets will be rejected by the autopilot as they will have timestamps that are older than the timestamp received for the same stream tuple on the other communication link.

The solution adopted for MAVProxy is shown below:

```
if (msg.get_signed() and
	self.mav.signing.link_id == 0 and
	msg.get_link_id() != 0 and
	self.target_system == msg.get_srcSystem() and
	self.target_component == msg.get_srcComponent()):
	# change to link_id from incoming packet
	self.mav.signing.link_id = msg.get_link_id()
```

what that says is that if the current link ID in use by MAVProxy is zero, and it receives a correctly signed packet with a non-zero link ID then it switches link ID to the one from the incoming packet.

The has the effect of making the GCS slave its link ID to the link ID of the autopilot.