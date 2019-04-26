# Camera Protocol

The camera protocol is used to configure camera payloads and request their status. 
It supports photo capture, and video capture and streaming.
It also includes messages to query and configure the onboard camera storage.

> **Tip** The [Dronecode Camera Manager](https://camera-manager.dronecode.org/en/) provides an implementation of this protocol.

## Camera Connection

Camera components are expected to follow the [Heartbeat/Connection Protocol](../services/heartbeat.md) and sent a constant flow of heartbeats (nominally at 1Hz).
Each camera must use a different pre-defined camera component ID: [MAV_COMP_ID_CAMERA](../messages/common.md#MAV_COMP_ID_CAMERA) to [MAV_COMP_ID_CAMERA6](../messages/common.md#MAV_COMP_ID_CAMERA6).

The first time a heartbeat is detected from a new camera, a GCS (or other receiving system) should start the [Camera Identification](#camera_identification) process.

> **Note** If a receiving system stops receiving heartbeats from the camera it is assumed to be *disconnected*, and should be removed from the list of available cameras. 
  If heartbeats are again detected, the *camera identification* process below must be restarted from the beginning.


## Basic Camera Operations

The [CAMERA_INFORMATION.flags](../messages/common.md#CAMERA_INFORMATION) provides information about camera capabilities.
It contains a bitmap of [CAMERA_CAP_FLAGS](../messages/common.md#CAMERA_CAP_FLAGS) values that tell the GCS if the camera supports still image capture, video capture, or video streaming, and if it needs to be in a certain mode for capture, etc.

### Camera Identification {#camera_identification}

The camera identification operation determines what cameras are available/exist (this is carried out before all other operations).

The first time a heartbeat is received from a camera component the GCS will send the camera a [MAV_CMD_REQUEST_CAMERA_INFORMATION](../messages/common.md#MAV_CMD_REQUEST_CAMERA_INFORMATION) message. 
The camera component will then respond with the a [COMMAND_ACK](../messages/common.md#COMMAND_ACK) message containing a result.
On success (result is [MAV_RESULT_ACCEPTED](../messages/common.md#MAV_RESULT_ACCEPTED)) the camera component must then send a [CAMERA_INFORMATION](../messages/common.md#CAMERA_INFORMATION) message.

{% mermaid %}
sequenceDiagram;
    participant GCS
    participant Camera
    Camera->>GCS: HEARTBEAT [cmp id: MAV_COMP_ID_CAMERA] (first) 
    GCS->>Camera: MAV_CMD_REQUEST_CAMERA_INFORMATION
    GCS->>GCS: Start timeout
    Camera->>GCS: COMMAND_ACK
    Note over Camera,GCS: If MAV_RESULT_ACCEPTED send info.
    Camera->>GCS: CAMERA_INFORMATION
{% endmermaid %}

The operation follows the normal [Command Protocol](../services/command.md) rules for command/acknowledgment (if no `COMMAND_ACK` response is received for `MAV_CMD_REQUEST_CAMERA_INFORMATION` the command will be re-sent a number of times before failing).
If `CAMERA_INFORMATION` is not received after receiving an ACK with `MAV_RESULT_ACCEPTED`, the protocol assumes the message was lost, and the cycle of sending `MAV_CMD_REQUEST_CAMERA_INFORMATION` is repeated. 
If `CAMERA_INFORMATION` is still not received after three cycle repeats, the GCS may assume that the camera is not supported.

The `CAMERA_INFORMATION` response contains the bare minimum information about the camera and what it can or cannot do.
This is sufficient for basic image and/or video capture.

If a camera provides finer control over its settings `CAMERA_INFORMATION.cam_definition_uri` will include a URI to a [Camera Definition File](../services/camera_def.md).
If this URI exists, the GCS will request it (using a standard HTTP GET request), parse it and prepare the UI for the user to control the camera settings. 
The definition file can be *hosted* anywhere.

> **Note** If the camera component provides an HTTP interface, the definition file can be hosted on the camera itself. 
  Otherwise, it can be hosted by any regular, reachable server. 

The `CAMERA_INFORMATION.cam_definition_version` field should provide a version for the definition file, allowing the GCS to cache it. 
Once downloaded, it would only be requested again if the version number changes.

If a vehicle has more than one camera, each camera will have a different component ID and send its own heartbeat. 
The GCS should create multiple instances of a camera controller based on the component ID of each camera. 
All commands are sent to a specific camera by addressing the command to a specific component ID.


### Camera Modes

Some cameras must be in a certain mode for still and/or video capture.

The GCS can determine if it needs to make sure the camera is in the proper mode prior to sending a start capture (image or video) command by checking whether the [CAMERA_CAP_FLAGS_HAS_MODES](../messages/common.md#CAMERA_CAP_FLAGS_HAS_MODES) bit is set true in [CAMERA_INFORMATION.flags](../messages/common.md#CAMERA_INFORMATION).

In addition, some cameras can capture images in any mode but with different resolutions. 
For example, a 20 megapixel camera would take a full resolution image when set to `CAMERA_MODE_IMAGE` but only at the current video resolution if it is set to `CAMERA_MODE_VIDEO`. 

To get the current mode, the GCS would send a [MAV_CMD_REQUEST_CAMERA_SETTINGS](../messages/common.md#MAV_CMD_REQUEST_CAMERA_SETTINGS) command.
The camera component will then respond with the a [COMMAND_ACK](../messages/common.md#COMMAND_ACK) message containing a result.
On success (`COMMAND_ACK.result` is [MAV_RESULT_ACCEPTED](../messages/common.md#MAV_RESULT_ACCEPTED)) the camera must then send a [CAMERA_SETTINGS](../messages/common.md#CAMERA_SETTINGS) message.
The current mode is the `CAMERA_SETTINGS.mode_id` field.

The sequence is shown below:

{% mermaid %}
sequenceDiagram;
    participant GCS
    participant Camera
    GCS->>Camera: MAV_CMD_REQUEST_CAMERA_SETTINGS
    GCS->>GCS: Start timeout
    Camera->>GCS: COMMAND_ACK
    Note over Camera,GCS: If MAV_RESULT_ACCEPTED send info.
    Camera->>GCS: CAMERA_SETTINGS
{% endmermaid %}

> **Note** Command acknowledgment and message resending is handled in the same way as for [camera identification](#camera_identification)
  (if a successful ACK is received the camera will expect the `CAMERA_SETTINGS` message, and repeat the cycle - up to 3 times - until it is received).

To set the camera to a specific mode, the GCS would send the [MAV_CMD_SET_CAMERA_MODE](../messages/common.md#MAV_CMD_SET_CAMERA_MODE) command with the appropriate mode.

The sequence is shown below:

{% mermaid %}
sequenceDiagram;
    participant GCS
    participant Camera
    GCS->>Camera: MAV_CMD_SET_CAMERA_MODE
    GCS->>GCS: Start timeout
    Camera->>GCS: COMMAND_ACK
    Note over Camera,GCS: If MAV_RESULT_ACCEPTED, mode was changed.
{% endmermaid %}

> **Note** The operation follows the normal [Command Protocol](../services/command.md) rules for command/acknowledgment.


### Storage Status

Before capturing images and/or videos, a GCS should query the storage status to determine if the camera has enough free space for these operations (and provide the user with feedback as to the current storage status). 
The GCS will send the [MAV_CMD_REQUEST_STORAGE_INFORMATION](../messages/common.md#MAV_CMD_REQUEST_STORAGE_INFORMATION) command and it expects a [STORAGE_INFORMATION](../messages/common.md#STORAGE_INFORMATION) response. 
For formatting (or erasing depending on your implementation), the GCS will send a [MAV_CMD_STORAGE_FORMAT](../messages/common.md#MAV_CMD_STORAGE_FORMAT) command.


### Camera Capture Status

In addition to querying about storage status, the GCS will also request the current *Camera Capture Status* in order to provide the user with proper UI indicators.
The GCS will send a [MAV_CMD_REQUEST_CAMERA_CAPTURE_STATUS](../messages/common.md#MAV_CMD_REQUEST_CAMERA_CAPTURE_STATUS) command and it expects a [CAMERA_CAPTURE_STATUS](../messages/common.md#CAMERA_CAPTURE_STATUS) response.


### Still Image Capture

A camera supports *still image capture* if the [CAMERA_CAP_FLAGS_CAPTURE_IMAGE](../messages/common.md#CAMERA_CAP_FLAGS_CAPTURE_IMAGE) bit is set in [CAMERA_INFORMATION.flags](../messages/common.md#CAMERA_INFORMATION).

A GCS/MAVLink app uses the [MAV_CMD_IMAGE_START_CAPTURE](../messages/common.md#MAV_CMD_IMAGE_START_CAPTURE) command to request that the camera capture a specified number of images (or forever), and the duration between them.
The camera immediately returns the normal command acknowledgment ([MAV_RESULT](../messages/common.md#MAV_RESULT_ACCEPTED)).

Each time an image is captured, the camera *broadcasts* a [CAMERA_IMAGE_CAPTURED](../messages/common.md#CAMERA_IMAGE_CAPTURED) message.
This message not only tells the GCS the image was captured, it is also intended for geo-tagging.

The [MAV_CMD_IMAGE_STOP_CAPTURE](../messages/common.md#MAV_CMD_IMAGE_STOP_CAPTURE) command can optionally be sent to stop an image capture sequence (this is needed if image capture has been set to continue forever).

The still image capture message sequence *for missions* (as described above) is shown below :

{% mermaid %}
sequenceDiagram;
    participant GCS
    participant Camera
    GCS->>Camera: MAV_CMD_IMAGE_START_CAPTURE (interval, count/forever)
    GCS->>GCS: Start timeout
    Camera->>GCS: MAV_RESULT_ACCEPTED
    Note over Camera,GCS: Camera start capture of "count" images at "interval"
    Camera->>GCS: CAMERA_IMAGE_CAPTURED  (broadcast)
    Camera->>GCS: ...
    Camera->>GCS: CAMERA_IMAGE_CAPTURED (broadcast)
    Note over Camera,GCS: (Optional) Stop capture
    GCS->>Camera: MAV_CMD_IMAGE_STOP_CAPTURE
    GCS->>GCS: Start timeout
    Camera->>GCS: MAV_RESULT_ACCEPTED
{% endmermaid %}

The message sequence for *interactive user-initiated image capture* through a GUI is slightly different.
In this case the GCS should:
- Confirm that the camera is *ready* to take images before allowing the user to request image capture.
  - It does this by by sending [MAV_CMD_REQUEST_CAMERA_CAPTURE_STATUS](../messages/common.md#MAV_CMD_REQUEST_CAMERA_CAPTURE_STATUS).
  - The camera should return a `MAV_RESULT` and then [CAMERA_CAPTURE_STATUS](../messages/common.md#CAMERA_CAPTURE_STATUS).
  - The GCS should check that the status is "Idle" before enabling camera capture in the GUI.
- Send [MAV_CMD_IMAGE_START_CAPTURE](../messages/common.md#MAV_CMD_IMAGE_START_CAPTURE) specifying a single image (only).

The sequence is as shown below:

{% mermaid %}
sequenceDiagram;
    participant GCS
    participant Camera
    GCS->>Camera: MAV_CMD_REQUEST_CAMERA_CAPTURE_STATUS
    GCS->>GCS: Start timeout
    Camera->>GCS: MAV_RESULT_ACCEPTED
    GCS->>GCS: Start timeout
    Camera->>GCS: CAMERA_CAPTURE_STATUS (status)
    Note over Camera,GCS: Repeat until status is IDLE
    GCS->>Camera: MAV_CMD_IMAGE_START_CAPTURE (interval, count/forever)
    GCS->>GCS: Start timeout
    Camera->>GCS: MAV_RESULT_ACCEPTED
    Note over Camera,GCS: Camera start capture of 1 image
    GCS->>GCS: Start timeout
    Camera->>GCS: CAMERA_IMAGE_CAPTURED  (broadcast)
{% endmermaid %}


### Video Capture

A camera supports video capture if the [CAMERA_CAP_FLAGS_CAPTURE_VIDEO](../messages/common.md#CAMERA_CAP_FLAGS_CAPTURE_VIDEO) bit is set in [CAMERA_INFORMATION.flags](../messages/common.md#CAMERA_INFORMATION).

To start recording videos, the GCS uses the [MAV_CMD_VIDEO_START_CAPTURE](../messages/common.md#MAV_CMD_VIDEO_START_CAPTURE) command. 
If requested, the [CAMERA_CAPTURE_STATUS](#camera_capture_status) message is sent to the GCS at a set interval.

To stop recording, the GCS uses the [MAV_CMD_VIDEO_STOP_CAPTURE](../messages/common.md#MAV_CMD_VIDEO_STOP_CAPTURE) command.


### Video Streaming

A camera is capable of streaming video if it sets the [CAMERA_CAP_FLAGS_HAS_VIDEO_STREAM](../messages/common.md#CAMERA_CAP_FLAGS_HAS_VIDEO_STREAM) bit set in [CAMERA_INFORMATION.flags](../messages/common.md#CAMERA_INFORMATION). 


When the GCS receives the [CAMERA_INFORMATION](../messages/common.md#CAMERA_INFORMATION) message and it detects the [CAMERA_CAP_FLAGS_HAS_VIDEO_STREAM](../messages/common.md#CAMERA_CAP_FLAGS_HAS_VIDEO_STREAM) flag, it will then send the [MAV_CMD_REQUEST_VIDEO_STREAM_INFORMATION](../messages/common.md#MAV_CMD_REQUEST_VIDEO_STREAM_INFORMATION) message to the camera requesting the video streaming configuration. 
In response, the camera returns a [VIDEO_STREAM_INFORMATION](../messages/common.md#VIDEO_STREAM_INFORMATION) message for each stream it supports.

> **Note** If your camera only provides video streaming and nothing else (no camera features), the [CAMERA_CAP_FLAGS_HAS_VIDEO_STREAM](../messages/common.md#CAMERA_CAP_FLAGS_HAS_VIDEO_STREAM) flag is the only flag you need to set. 
  The GCS will then provide video streaming support and skip camera control.

## Message/Enum Summary

Message | Description
-- | --
<span id="mav_cmd_request_camera_information"></span>[MAV_CMD_REQUEST_CAMERA_INFORMATION](../messages/common.md#MAV_CMD_REQUEST_CAMERA_INFORMATION) | Send command to request [CAMERA_INFORMATION](#camera_information).
<span id="camera_information"></span>[CAMERA_INFORMATION](../messages/common.md#CAMERA_INFORMATION) | Basic information about camera including supported features and URI link to extended information (`cam_definition_uri` field).
<span id="mav_cmd_request_camera_settings"></span>[MAV_CMD_REQUEST_CAMERA_SETTINGS](../messages/common.md#MAV_CMD_REQUEST_CAMERA_SETTINGS) | Send command to request [CAMERA_SETTINGS](#camera_settings).
<span id="camera_settings"></span>[CAMERA_SETTINGS](../messages/common.md#CAMERA_SETTINGS) | Timestamp and camera mode information.
<span id="mav_cmd_set_camera_mode"></span>[MAV_CMD_SET_CAMERA_MODE](../messages/common.md#MAV_CMD_SET_CAMERA_MODE) | Send command to set [CAMERA_MODE](#camera_mode).
<span id="mav_cmd_request_video_stream_information"></span>[MAV_CMD_REQUEST_VIDEO_STREAM_INFORMATION](../messages/common.md#MAV_CMD_REQUEST_VIDEO_STREAM_INFORMATION) | Send command to request [VIDEO_STREAM_INFORMATION](#video_stream_information). This is sent once for each camera when a camera is detected and it has set the [CAMERA_CAP_FLAGS_HAS_VIDEO_STREAM](../messages/common.md#CAMERA_CAP_FLAGS_HAS_VIDEO_STREAM) flag within the [CAMERA_INFORMATION](../messages/common.md#CAMERA_INFORMATION) message `flags` field.
<span id="video_stream_information"></span>[VIDEO_STREAM_INFORMATION](../messages/common.md#VIDEO_STREAM_INFORMATION) | Information defining a video stream configuration. If a camera has more than one video stream, it would send one of this for each video stream, with their specific configuration. Each stream must have its own, unique `stream_id`.
<span id="mav_cmd_request_video_stream_status"></span>[MAV_CMD_REQUEST_VIDEO_STREAM_STATUS](../messages/common.md#MAV_CMD_REQUEST_VIDEO_STREAM_STATUS) | Send command to request [VIDEO_STREAM_STATUS](#video_stream_status). This is sent whenever there is a mode change (when [MAV_CMD_SET_CAMERA_MODE](../messages/common.md#MAV_CMD_SET_CAMERA_MODE) is sent.) It allows the camera to update the stream configuration when a camera mode change occurs.
<span id="video_stream_status"></span>[VIDEO_STREAM_STATUS](../messages/common.md#VIDEO_STREAM_STATUS) | Information updating a video stream configuration. <!-- TBD? -->
<span id="mav_cmd_request_storage_information"></span>[MAV_CMD_REQUEST_STORAGE_INFORMATION](../messages/common.md#MAV_CMD_REQUEST_STORAGE_INFORMATION) | Send command to request [STORAGE_INFORMATION](#storage_information).
<span id="storage_information"></span>[STORAGE_INFORMATION](../messages/common.md#STORAGE_INFORMATION) | Storage information (e.g. number and type of storage devices, total/used/available capacity, read/write speeds).
<span id="mav_cmd_storage_format"></span>[MAV_CMD_STORAGE_FORMAT](../messages/common.md#MAV_CMD_STORAGE_FORMAT) | Send command to format the specified storage device.
<span id="mav_cmd_request_camera_capture_status"></span>[MAV_CMD_REQUEST_CAMERA_CAPTURE_STATUS](../messages/common.md#MAV_CMD_REQUEST_CAMERA_CAPTURE_STATUS) | Send command to request [CAMERA_CAPTURE_STATUS](#camera_capture_status).
<span id="camera_capture_status"></span>[CAMERA_CAPTURE_STATUS](../messages/common.md#CAMERA_CAPTURE_STATUS) | Camera capture status, including current capture type (if any), capture interval, available capacity.
<span id="mav_cmd_image_start_capture"></span>[MAV_CMD_IMAGE_START_CAPTURE](../messages/common.md#MAV_CMD_IMAGE_START_CAPTURE) | Send command to start image capture, specifying the duration between captures and total number of images to capture.
<span id="mav_cmd_image_stop_capture"></span>[MAV_CMD_IMAGE_STOP_CAPTURE](../messages/common.md#MAV_CMD_IMAGE_STOP_CAPTURE) | Send command to stop image capture.
<span id="camera_image_captured"></span>[CAMERA_IMAGE_CAPTURED](../messages/common.md#CAMERA_IMAGE_CAPTURED) | Information about image captured (returned to GPS every time an image is captured). 
<span id="mav_cmd_video_start_capture"></span>[MAV_CMD_VIDEO_START_CAPTURE](../messages/common.md#MAV_CMD_VIDEO_START_CAPTURE) | Send command to start video capture, specifying the frequency that [CAMERA_CAPTURE_STATUS](#camera_capture_status) messages should be sent while recording.
<span id="mav_cmd_video_stop_capture"></span>[MAV_CMD_VIDEO_STOP_CAPTURE](../messages/common.md#MAV_CMD_VIDEO_STOP_CAPTURE) | Send command to stop video capture.
<span id="camera_image_captured"></span>[CAMERA_IMAGE_CAPTURED](../messages/common.md#CAMERA_IMAGE_CAPTURED) | Information about image captured (returned to GPS every time an image is captured). 
<span id="mav_cmd_image_start_streaming"></span>[MAV_CMD_VIDEO_START_STREAMING](../messages/common.md#MAV_CMD_VIDEO_START_STREAMING) | Send command to start video streaming for the given Stream ID (`stream_id`.) This is mostly for streaming protocols that _push_ a stream. If your camera uses a connection based streaming configuration (RTSP, TCP, etc.), you may ignore it if you don't need it but note that you still must ACK the command, like all `MAV_CMD_XXX` commands. When using a connection based streaming configuration, the GCS will connect the stream from its side. When a camera offers more than one stream and the user switches from one stream to another, the GCS will send a [MAV_CMD_VIDEO_STOP_STREAMING](../messages/common.md#MAV_CMD_VIDEO_STOP_STREAMING) command targeting the current Stream ID followed by a [MAV_CMD_VIDEO_START_STREAMING](../messages/common.md#MAV_CMD_VIDEO_START_STREAMING) targeting the newly selected Stream ID.
<span id="mav_cmd_image_stop_streaming"></span>[MAV_CMD_VIDEO_STOP_STREAMING](../messages/common.md#MAV_CMD_VIDEO_STOP_STREAMING) | Send command to stop video streaming for the given Stream ID (`stream_id`.) This is mostly for streaming protocols that _push_ a stream. If your camera uses a connection based streaming configuration (RTSP, TCP, etc.), you may ignore it if you don't need it but note that you still must ACK the command, like all `MAV_CMD_XXX` commands. When using a connection based streaming configuration, the GCS will disconnect the stream from its side. When a camera offers more than one stream and the user switches from one stream to another, the GCS will send a [MAV_CMD_VIDEO_STOP_STREAMING](../messages/common.md#MAV_CMD_VIDEO_STOP_STREAMING) command targeting the current Stream ID followed by a [MAV_CMD_VIDEO_START_STREAMING](../messages/common.md#MAV_CMD_VIDEO_START_STREAMING) targeting the newly selected Stream ID.


Enum | Description
-- | --
<span id="camera_cap_flags"></span>[CAMERA_CAP_FLAGS](../messages/common.md#CAMERA_CAP_FLAGS) | Camera capability flags (Bitmap). For example: ability to capture images in video mode, support for survey mode etc. Received in [CAMERA_INFORMATION](#camera_information).
<span id="camera_mode"></span>[CAMERA_MODE](../messages/common.md#CAMERA_MODE) | Camera mode (image, video, survey etc.). Received in [CAMERA_SETTINGS](#camera_settings).
<span id="video_stream_type"></span>[VIDEO_STREAM_TYPE](../messages/common.md#VIDEO_STREAM_TYPE) | Type of stream - e.g. RTSP, MPEG. Received in [VIDEO_STREAM_INFORMATION ](#video_stream_information).
<span id="video_stream_status_flags"></span>[VIDEO_STREAM_STATUS_FLAGS](../messages/common.md#VIDEO_STREAM_TYPE) | Bitmap of stream status flags - e.g. zoom, thermal imaging, etc. Received in [VIDEO_STREAM_INFORMATION ](#video_stream_information).

