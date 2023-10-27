---
description: Breakdown of the main mission file.
---

# Mission.py

## **Overview**

In a drone mission for the ANAFI AI drone, the `mission.py` file serves as the control center. Imagine it as the drone's brain that tells it what to do and how to do it during a mission. The mission has several stages like takeoff, hovering, and landing, and this file orchestrates all of them. Here's a breakdown:

1. **Initialize Mission**: When the mission starts, this file sets up everything, connecting different parts of the system, such as the user interface, camera, and other control features.
2. **Communication Through Messages**: Throughout the mission, 'messages' are exchanged to relay commands and receive updates. It's how the drone and control interface talk to each other.
3. **Different Stages**: The drone goes through various stages (takeoff, hovering, flying, landing) controlled by this file.
4. **Event Handling**: It listens for certain 'events' (like a change in altitude) and responds accordingly, like adjusting the drone's height.
5. **Real-Time Updates**: Whether it's info from the drone's camera or its motion status, real-time updates are sent and received to and from the drone.
6. **Debugging**: If anything goes wrong, this file has a way to observe what happened for troubleshooting.
7. **Cleanup**: Once the mission is complete, the file tidies everything up, ensuring all connections are closed and the system is ready for the next mission.

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
def _on_connected(self, channel):
    """
    Method to handle channel connection events.
    Logs information about which channel was successfully connected.
    
    :param channel: The channel that got connected
    """
    # Logging information when a channel gets connected
    if channel == self.env.airsdk_channel:
        self.log.info("connected to airsdk channel")
    elif channel == self.cv_service_msgs_channel:
        self.log.info("connected to cv service channel")
```

**\_send\_to\_ui\_stereo\_camera\_close\_state**: Sends state information to the UI about stereo camera close state.

```python
def _send_to_ui_stereo_camera_close_state(self, state):
    """
    Method to send the close state of the stereo camera to the UI.
    
    :param state: The close state of the stereo camera
    """
    self.ext_ui_msgs.evt.sender.stereo_close(state)
```

**\_send\_to\_ui\_drone\_motion\_state**: Sends drone's motion state to the UI.

```python
def _send_to_ui_drone_motion_state(self, state):
    """
    Method to send the motion state of the drone to the UI.
    
    :param state: The motion state of the drone
    """
    self.ext_ui_msgs.evt.sender.drone_moving(state)
```

**\_send\_to\_ui\_depth\_mean**: Sends depth mean to the UI.

```python
def _send_to_ui_depth_mean(self, depth_mean):
    """
    Method to send the mean depth calculated by the CV service to the UI.
    
    :param depth_mean: The mean depth value
    """
    self.ext_ui_msgs.evt.sender.depth_mean(depth_mean)
```

**\_on\_ui\_msg\_cmd**: Logs UI command messages.

```python
def _on_ui_msg_cmd(self, event, message):
    """
    Debug method to handle incoming UI command messages.
    Logs incoming UI command messages for debugging purposes.
    
    :param event: The event that triggered this method
    :param message: The UI command message
    """
    self.log.debug("%s: UI command message %s", UID, message)
```

**on\_deactivate**: Stops services, detaches message pairs, and cleans up message channels.

```python
def on_deactivate(self):
    """
    Method to handle mission deactivation.
    Cleans up various message channels and observers.
    """
    # Stop Computer Vision service processing
    self.cv_service_msgs.cmd.sender.processing_stop()
    
    # Unobserve previously attached observers
    self.observer.unobserve()
    self.observer = None
    self.dbg_observer.unobserve()
    self.dbg_observer = None

    # Detach message channels
    self.mc.detach_client_service_pair(self.cv_service_msgs)
    self.cv_service_msgs = None
    self.mc.detach_client_service_pair(self.gdnc_grd_mode_msgs)
    self.gdnc_grd_mode_msgs = None
    
    # Stop Computer Vision service channel
    self.mc.stop_channel(self.cv_service_msgs_channel)
    self.cv_service_msgs_channel = None

    # Detach UI messages
    self.ext_ui_msgs.detach()
```

**states**: Returns an array of stages for the mission.

```python
def states(self):
    """
    Method to define the different stages (or states) in the mission.
    
    :return: List of stages that are part of this mission
    """
    return [
        GROUND_STAGE,
        DEF_TAKEOFF_STAGE,
        DEF_HOVERING_STAGE,
        FLYING_STAGE,
        DEF_LANDING_STAGE,
        DEF_CRITICAL_STAGE,
    ]
```

**transitions**: Returns an array of transitions for the mission state machine.

```python
def transitions(self):
    """
    Method to define the state-machine transitions for this mission.
    
    :return: List of transitions in the mission state-machine
    """
    # Custom transitions for this mission
      TRANSITIONS = [
        # "say/hold" messages from the mission UI alternate between "say"
        # and "idle" states in the ground stage.
        [
            msg_id(hello_msgs.Command, "say"),
            "ground.idle",
            "ground.say",
        ],
        [
            msg_id(hello_msgs.Command, "hold"),
            "ground.say",
            "ground.idle",
        ],
    ]

    # Combine custom transitions with default transitions
    return TRANSITIONS + DEF_TRANSITIONS
```
