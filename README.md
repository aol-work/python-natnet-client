# Python NatNet Client

Python client for Optitrack NatNet streams.

## Installation

Install this package via pip:

```bash
pip install git+https://github.com/TimSchneider42/python-natnet-client
```

## Usage

The following example highlights the basic usage of this package:

```python
import time

from natnet_client import DataDescriptions, DataFrame, NatNetClient


def receive_new_frame(data_frame: DataFrame):
    global num_frames
    num_frames += 1


def receive_new_desc(desc: DataDescriptions):
    print("Received data descriptions.")


num_frames = 0
if __name__ == "__main__":
    streaming_client = NatNetClient(server_ip_address="127.0.0.1", local_ip_address="127.0.0.1", use_multicast=False)
    streaming_client.on_data_description_received_event.handlers.append(receive_new_desc)
    streaming_client.on_data_frame_received_event.handlers.append(receive_new_frame)

    with streaming_client:
        streaming_client.request_modeldef()

        for i in range(10):
            time.sleep(1)
            print(f"Received {num_frames} frames in {i + 1}s")
```

In this example, we first instantiate `NatNetClient` with the connection parameters and attach one callback function to
each of its events. The `streaming_client.on_data_description_received_event` event is triggered whenever a new data
description packet arrives, while the `streaming_client.on_data_frame_received_event` event is triggered on each
incoming data frame. For the configuration of the NatNet server, please refer to the official documentation.

We then use the `streaming_client` instance as a context manager, which is equivalent to
calling `streaming_client.connect()` (and `streaming_client.shutdown()` afterwards). After the client has been
connected, we request the model definitions from the server, which causes it to send a data description packet. Note
that data frames do not have to be explicitly requested, but are continuously streamed once a connection has been
established.

Apart from requesting model definitions, the `NatNetClient` class allows sending arbitrary commands to the NatNet server
via the `send_command` and `send_request` functions. For a list of different commands and requests, please refer to the
official documentations.

## Remote Requests / Commands

The library has limited support for [Remote Requests/Commands]
(https://docs.optitrack.com/developer-tools/natnet-sdk/natnet-remote-requests-commands).

Here is one example of receiving the current frame rate. You can register a callback
for raw responses and then send the command.
with:

```python
# ...
from natnet_client import PacketBuffer
# ...
def receive_new_response_raw(databuf: PacketBuffer):
  # We know this is a GetFrameRate command, as we only sent a request for this
  # command, so we can assume we receive exactly one float as to the documentation.
  print(f"Current frame rate: {databuf.read_float()}")

# ... other initialization code...
streaming_client.on_response_received_raw_event.handlers.append(
        receive_new_response_raw)
# ...after initialization...
streaming_client.send_command("GetFrameRate")
# ...
```

Not that you need to handle the send / receive flow yourself. If you send
multiple commands at once, you will not be able to distinguish by the callback
parameters alone which response belongs to which command.

## Notes

As of Motive version 2.3, the marker positions of rigid bodies are only transmitted correctly if "Y-up" is selected in
the streaming pane. If "Z-up" is selected, the frame of the rigid bodies is rotated but the marker positions are not,
resulting in wrong positions of the markers relative to the rigid body.
