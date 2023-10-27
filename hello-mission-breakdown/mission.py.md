---
description: Breakdown of the main mission file.
---

# Mission.py

## **Imports**

**msghub\_utils**: Imports `msg_id`, a utility to manage message IDs.

```python
from msghub_utils import msg_id
```

**fsup.genmission**: Imports `AbstractMission`, a base class for the mission.

```python
from fsup.genmission import AbstractMission
```

**fsup.missions.default.\[stage]**: Imports default stages like takeoff, hovering, landing, and critical stages.

```python
from fsup.missions.default.hovering.stage import (
    HOVERING_STAGE as DEF_HOVERING_STAGE,
)
from fsup.missions.default.landing.stage import (
    LANDING_STAGE as DEF_LANDING_STAGE,
)
from fsup.missions.default.critical.stage import (
    CRITICAL_STAGE as DEF_CRITICAL_STAGE,
)
```

**fsup.missions.default.mission**: Imports default mission transitions.

```python
from fsup.missions.default.mission import TRANSITIONS as DEF_TRANSITIONS
```

**parrot.missions.samples.hello.airsdk.messages\_pb2**: Imports message protocols for communication with the mission UI.

```python
# messages exchanged with mission UI
import parrot.missions.samples.hello.airsdk.messages_pb2 as hello_msgs
```

**samples.hello.guidance.messages\_pb2**: Imports message protocols for communication with the guidance ground mode.

```python
# messages exchanged with the guidance ground mode
import samples.hello.guidance.messages_pb2 as hello_gdnc_mode_msgs
```

**samples.hello.cv\_service.messages\_pb2**: Imports message protocols for communication with the computer vision service.

```python
# messages exchanged with the cv service
import samples.hello.cv_service.messages_pb2 as hello_cv_service_msgs
```

**fsup.message\_center.events**: Imports events for message processing.

```python
# events that are not expressed as protobuf messages
import fsup.message_center.events as events
```

**drone\_controller.drone\_controller\_pb2**: Imports message protocols for drone controller.

```python
# messages exchanged with the drone controller
import drone_controller.drone_controller_pb2 as dctl_msgs
```

**colibrylite.motion\_state\_pb2**: Imports states related to the motion.

```
import colibrylite.motion_state_pb2 as cbry_motion_state
```

**.ground.stage, .flying.stage, .uid**: Imports custom mission stages and a unique identifier.

```python
from .ground.stage import GROUND_STAGE
from .flying.stage import FLYING_STAGE
from .uid import UID
```

## **Mission Class**

```python
class Mission(AbstractMission):
    ...
```

**Class Overview**

The class `Mission` is derived from `AbstractMission`. It has some attributes, mostly related to message channels and observers for various services like UI, guidance, and computer vision.

**Methods**

**init**: Initializes message channels and observer variables.

```python
def __init__(self, env):
    """
    Constructor method to initialize the Mission object.

    Parameters:
        env: The environment in which the mission runs.
    """
    super().__init__(env)
    self.ext_ui_msgs = None  # Channel for UI messages
    self.cv_service_msgs_channel = None  # Channel for Computer Vision messages
    self.cv_service_msgs = None  # Computer Vision messages
    self.gdnc_grd_mode_msgs = None  # Guidance ground mode messages
    self.observer = None  # Observer for forwarding messages
    self.dbg_observer = None  # Debug observer for UI messages
```

**on\_load**: Initializes the message pair for external UI communication.

```python
def on_load(self):
    """
    Method to set up messaging and communication channels.
    """
    # Create a communication channel for UI messages
    self.ext_ui_msgs = self.env.make_airsdk_service_pair(hello_msgs)
```

**on\_unload**: Cleans up the message channels for UI.

```python
def on_unload(self):
    """
    Method to clean up messaging channels when the mission is unloaded.
    """
    # Remove the UI messaging channel
    self.ext_ui_msgs = None
```

**on\_activate**: Sets up message communication channels, attaches them, and starts services.

```python
def on_activate(self):
    """
    Method to handle mission activation.
    Sets up various message channels and attaches observers.
    """
    # Attach mission UI messages to the airsdk channel
    self.ext_ui_msgs.attach(self.env.airsdk_channel, True)

    # Attach Guidance ground mode messages to the guidance channel
    self.gdnc_grd_mode_msgs = self.mc.attach_client_service_pair(
        self.mc.gdnc_channel, hello_gdnc_mode_msgs, True
    )

    # Start a new channel for the Computer Vision service
    self.cv_service_msgs_channel = self.mc.start_client_channel(
        "unix:/tmp/hello-cv-service"
    )

    # Attach Computer Vision service messages to the new channel
    self.cv_service_msgs = self.mc.attach_client_service_pair(
        self.cv_service_msgs_channel, hello_cv_service_msgs, True
    )

    # Setup observers for various events and messages
    self.observer = self.mc.observe(
            {
                events.Channel.CONNECTED: lambda _, c: self._on_connected(c),
                msg_id(
                    hello_cv_service_msgs.Event, "close"
                ): lambda *args: self._send_to_ui_stereo_camera_close_state(
                    True
                ),
                msg_id(
                    hello_cv_service_msgs.Event, "far"
                ): lambda *args: self._send_to_ui_stereo_camera_close_state(
                    False
                ),
                msg_id(
                    hello_cv_service_msgs.Event, "depth_mean"
                ): lambda _, msg: self._send_to_ui_depth_mean(msg.depth_mean),
                msg_id(
                    dctl_msgs.Event, "motion_state_changed"
                ): lambda _, msg: self._send_to_ui_drone_motion_state(
                    msg.motion_state_changed == cbry_motion_state.MOVING
                ),
            }
        )

    # Setup a debug observer specifically for UI messages
    self.dbg_observer = self.ext_ui_msgs.cmd.observe(
        {events.Service.MESSAGE: self._on_ui_msg_cmd}
    )

    # Command to start Computer Vision service processing
    self.cv_service_msgs.cmd.sender.processing_start()
```

**\_on\_connected**: Logs information about connected channels.

```python
// Some code
```

1. **\_send\_to\_ui\_stereo\_camera\_close\_state**: Sends state information to the UI about stereo camera close state.
2. **\_send\_to\_ui\_drone\_motion\_state**: Sends drone's motion state to the UI.
3. **\_send\_to\_ui\_depth\_mean**: Sends depth mean to the UI.
4. **\_on\_ui\_msg\_cmd**: Logs UI command messages.
5. **on\_deactivate**: Stops services, detaches message pairs, and cleans up message channels.
6. **states**: Returns an array of stages for the mission.
7. **transitions**: Returns an array of transitions for the mission state machine.
