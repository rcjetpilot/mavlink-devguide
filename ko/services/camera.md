# Camera Protocol

The camera protocol is used to configure camera payloads and request their status. It supports photo and video cameras and includes messages to query and configure the onboard camera storage.

> **Tip** The [Dronecode Camera Manager](https://camera-manager.dronecode.org/en/) provides an implementation of this protocol.

## Camera Identification

The first step is to determine if a camera exists. Camera components are supposed to send heartbeats just like any other component. There are pre-defined component IDs for cameras - see [MAV_COMP_ID_CAMERA](../messages/common.md#MAV_COMP_ID_CAMERA). If a camera component exists, once a heartbeat is received a [MAV_CMD_REQUEST_CAMERA_INFORMATION](../messages/common.md#MAV_CMD_REQUEST_CAMERA_INFORMATION) message is sent from the GCS. The camera component will then reply with a [CAMERA_INFORMATION](../messages/common.md#CAMERA_INFORMATION) message.

This response contains the bare minimum information about the camera and what it can or cannot do. By itself, it is sufficient for default image and/or video capture. However, if a camera provides finer control over its settings, this message will also include an URI to a [Camera Definition File](../services/camera_def.md). If this URI exists, the GCS will request it (using a standard HTTP GET request), parse it and prepare the UI for the user to control the camera settings. The definition file can be *hosted* anywhere. If the camera component provides an HTTP interface, the definition file can be hosted on the camera itself. Otherwise, it can be hosted by any regular, reachable server. The `CAMERA_INFORMATION` message should provide a version for the definition file (`cam_definition_version`), allowing the GCS to cache it. Once downloaded, it would only be requested again if the version number changes.

> **Note** If no response is sent for a `MAV_CMD_REQUEST_CAMERA_INFORMATION` message, it is assumed camera support is not available and no support for it will be provided by the GCS.

If a vehicle has more than one camera, each camera will have a different component ID and send their own heartbeats. The GCS will create multiple instances of a camera controller based on the component ID of each camera. All commands are sent to a specific camera by addressing the command to a specific component ID.

## Basic Camera Operations

The [CAMERA_INFORMATION](../messages/common.md#CAMERA_INFORMATION) message contains a `flags` field indicating the camera capabilities. The flag is a bit field based on the [CAMERA_CAP_FLAGS](../messages/common.md#CAMERA_CAP_FLAGS) enum. It will tell the GCS if the camera is able to capture still images and/or video, if it needs to be in a certain mode to capture, etc.

### Camera Modes

Some cameras must be in a certain mode for still and/or video capture. The [CAMERA_INFORMATION](../messages/common.md#CAMERA_INFORMATION) message `flags` field uses the [CAMERA_CAP_FLAGS_HAS_MODES](../messages/common.md#CAMERA_CAP_FLAGS_HAS_MODES) bit true to inform the GCS that it needs to make sure the camera is in the proper mode prior to sending a start capture (image or video) command. In addition, some cameras can capture images in any mode but with different resolutions. For example, a 20 megapixel camera would take a full resolution image when set to `CAMERA_MODE_IMAGE` but only at the current video resolution if it is set to `CAMERA_MODE_VIDEO`.

To get the current mode, the GCS would send a [MAV_CMD_REQUEST_CAMERA_SETTINGS](../messages/common.md#MAV_CMD_REQUEST_CAMERA_SETTINGS) command. The current mode is sent back in the `mode_id` field of the [CAMERA_SETTINGS](../messages/common.md#CAMERA_SETTINGS) message.

To set the camera to a specific mode, the GCS would send in turn the [MAV_CMD_SET_CAMERA_MODE](../messages/common.md#MAV_CMD_SET_CAMERA_MODE) command with the appropriate mode.

### Storage Status

Before capturing images and/or videos, the GCS will query the storage status to determine if the camera has enough free space for these operations (and provide the user with feedback as to the current storage status). The GCS will send the [MAV_CMD_REQUEST_STORAGE_INFORMATION](../messages/common.md#MAV_CMD_REQUEST_STORAGE_INFORMATION) command and it expects a [STORAGE_INFORMATION](../messages/common.md#STORAGE_INFORMATION) response. For formatting (or erasing depending on your implementation), the GCS will send a [MAV_CMD_STORAGE_FORMAT](../messages/common.md#MAV_CMD_STORAGE_FORMAT) command.

### Camera Capture Status

In addition to querying about storage status, the GCS will also request the current *Camera Capture Status* in order to provide the user with proper UI indicators. The GCS will send a [MAV_CMD_REQUEST_CAMERA_CAPTURE_STATUS](../messages/common.md#MAV_CMD_REQUEST_CAMERA_CAPTURE_STATUS) command and it expects a [CAMERA_CAPTURE_STATUS](../messages/common.md#CAMERA_CAPTURE_STATUS) response.

### Still Image Capture

If the `flags` field of the [CAMERA_INFORMATION](../messages/common.md#CAMERA_INFORMATION) message has the [CAMERA_CAP_FLAGS_CAPTURE_IMAGE](../messages/common.md#CAMERA_CAP_FLAGS_CAPTURE_IMAGE) bit set, it will indicate the GCS can send image capture commands to the camera.

To capture an image, the GCS uses the [MAV_CMD_IMAGE_START_CAPTURE](../messages/common.md#MAV_CMD_IMAGE_START_CAPTURE) command. Each time an image is captured, a [CAMERA_IMAGE_CAPTURED](../messages/common.md#CAMERA_IMAGE_CAPTURED) message is sent back to the GCS.

The `CAMERA_IMAGE_CAPTURED` message not only tells the GCS the image was captured, it is also intended for geo-tagging.

The capture command can be used to request one single image capture or a time lapse. If the command is set to take more than one single image, the GCS might use the [MAV_CMD_IMAGE_STOP_CAPTURE](../messages/common.md#MAV_CMD_IMAGE_STOP_CAPTURE) command to stop it.

### Video Capture

Just like for image capture, if the `flags` field of the [CAMERA_INFORMATION](../messages/common.md#CAMERA_INFORMATION) message has the [CAMERA_CAP_FLAGS_CAPTURE_VIDEO](../messages/common.md#CAMERA_CAP_FLAGS_CAPTURE_VIDEO) bit set, it will indicate the GCS can send the video capture command to the camera.

To start recording videos, the GCS uses the [MAV_CMD_VIDEO_START_CAPTURE](../messages/common.md#MAV_CMD_VIDEO_START_CAPTURE) command. If requested, the [CAMERA_CAPTURE_STATUS](#camera_capture_status) message is sent to the GCS at a set interval.

To stop recording, the GCS uses the [MAV_CMD_VIDEO_STOP_CAPTURE](../messages/common.md#MAV_CMD_VIDEO_STOP_CAPTURE) command.

## Message Summary

| Message                                                                                                                            | Description                                                                                                                                                   |
| ---------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| <span id="mav_cmd_request_camera_information"></span>[MAV_CMD_REQUEST_CAMERA_INFORMATION](../messages/common.md#MAV_CMD_REQUEST_CAMERA_INFORMATION)        | Send command to request [CAMERA_INFORMATION](#camera_information).                                                                                            |
| <span id="camera_information"></span>[CAMERA_INFORMATION](../messages/common.md#CAMERA_INFORMATION)                                            | Basic information about camera including URI link to extended information (`cam_definition_uri` field).                                                       |
| <span id="camera_cap_flags"></span>[CAMERA_CAP_FLAGS](../messages/common.md#CAMERA_CAP_FLAGS)                                              | Camera capability flags (Bitmap). For example: ability to capture images in video mode, support for survey mode etc.                                          |
| <span id="mav_cmd_request_camera_settings"></span>[MAV_CMD_REQUEST_CAMERA_SETTINGS](../messages/common.md#MAV_CMD_REQUEST_CAMERA_SETTINGS)              | Send command to request [CAMERA_SETTINGS](#camera_settings).                                                                                                  |
| <span id="camera_settings"></span>[CAMERA_SETTINGS](../messages/common.md#CAMERA_SETTINGS)                                                  | Timestamp and camera mode information.                                                                                                                        |
| <span id="mav_cmd_set_camera_mode"></span>[MAV_CMD_SET_CAMERA_MODE](../messages/common.md#MAV_CMD_SET_CAMERA_MODE)                              | Send command to set [CAMERA_MODE](#camera_mode).                                                                                                              |
| <span id="camera_mode"></span>[CAMERA_MODE](../messages/common.md#CAMERA_MODE)                                                          | Camera mode (image, video, survey etc.)                                                                                                                       |
| <span id="mav_cmd_request_storage_information"></span>[MAV_CMD_REQUEST_STORAGE_INFORMATION](../messages/common.md#MAV_CMD_REQUEST_STORAGE_INFORMATION)      | Send command to request [STORAGE_INFORMATION](#storage_information).                                                                                          |
| <span id="storage_information"></span>[STORAGE_INFORMATION](../messages/common.md#STORAGE_INFORMATION)                                          | Storage information (e.g. number and type of storage devices, total/used/available capacity, read/write speeds).                                              |
| <span id="mav_cmd_storage_format"></span>[MAV_CMD_STORAGE_FORMAT](../messages/common.md#MAV_CMD_STORAGE_FORMAT)                                  | Send command to format the specified storage device.                                                                                                          |
| <span id="mav_cmd_request_camera_capture_status"></span>[MAV_CMD_REQUEST_CAMERA_CAPTURE_STATUS](../messages/common.md#MAV_CMD_REQUEST_CAMERA_CAPTURE_STATUS) | Send command to request [CAMERA_CAPTURE_STATUS](#camera_capture_status).                                                                                    |
| <span id="camera_capture_status"></span>[CAMERA_CAPTURE_STATUS](../messages/common.md#CAMERA_CAPTURE_STATUS)                                   | Camera capture status, including current capture type (if any), capture interval, available capacity.                                                         |
| <span id="mav_cmd_image_start_capture"></span>[MAV_CMD_IMAGE_START_CAPTURE](../messages/common.md#MAV_CMD_IMAGE_START_CAPTURE)                     | Send command to start image capture, specifying the duration between captures and total number of images to capture.                                          |
| <span id="mav_cmd_image_stop_capture"></span>[MAV_CMD_IMAGE_STOP_CAPTURE](../messages/common.md#MAV_CMD_IMAGE_STOP_CAPTURE)                       | Send command to stop image capture.                                                                                                                           |
| <span id="camera_image_captured"></span>[CAMERA_IMAGE_CAPTURED](../messages/common.md#CAMERA_IMAGE_CAPTURED)                                   | Information about image captured (returned to GPS every time an image is captured).                                                                           |
| <span id="mav_cmd_video_start_capture"></span>[MAV_CMD_VIDEO_START_CAPTURE](../messages/common.md#MAV_CMD_VIDEO_START_CAPTURE)                     | Send command to start video capture, specifying the frequency that [CAMERA_CAPTURE_STATUS](#camera_capture_status) messages should be sent while recording. |
| <span id="mav_cmd_video_stop_capture"></span>[MAV_CMD_VIDEO_STOP_CAPTURE](../messages/common.md#MAV_CMD_VIDEO_STOP_CAPTURE)                       | Send command to stop video capture.                                                                                                                           |
| <span id="camera_image_captured"></span>[CAMERA_IMAGE_CAPTURED](../messages/common.md#CAMERA_IMAGE_CAPTURED)                                   | Information about image captured (returned to GPS every time an image is captured).                                                                           |