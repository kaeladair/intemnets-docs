---
description: Ground stage overview.
---

# Ground Stage

## Overview

In the drone mission for the ANAFI AI drone, the `stage.py` file for the ground stage is responsible for handling states and behavior when the drone is on the ground. This is a part of the guidance system and defines two states—Idle and Say—which manage the drone's estimation modes and guidance modes when it's on the ground.

## Imports

{% code fullWidth="false" %}
```python
from msghub_utils import is_msg
```
{% endcode %}

```python
from fsup.genstate import State, guidance_modes
```

```python
import colibrylite.estimation_mode_pb2 as cbry_est
```

```python
from fsup.missions.default.ground.stage import GROUND_STAGE as DEF_GROUND_STAGE
```

```python
import samples.hello.guidance.messages_pb2 as hello_gdnc_mode_msgs
```

```python
from ..uid import UID
```

```python
_STATES_TO_REMOVE = ["idle"]

_GROUND_MODE_NAME = UID + ".ground"
```

```python
@guidance_modes(_GROUND_MODE_NAME)
class Idle(State):
    def enter(self, msg):
        self.mc.dctl.cmd.sender.set_estimation_mode(cbry_est.MOTORS_STOPPED)
        self.set_guidance_mode(
            _GROUND_MODE_NAME, hello_gdnc_mode_msgs.Config(say=False)
        )
```

```python
@guidance_modes(_GROUND_MODE_NAME)
class Say(State):
    def enter(self, msg):
        self.mc.dctl.cmd.sender.set_estimation_mode(cbry_est.MOTORS_STOPPED)
        self.set_guidance_mode(
            _GROUND_MODE_NAME, hello_gdnc_mode_msgs.Config(say=True)
        )

    # State machine will call the state step method with the guidance ground
    # mode "count" event.
    def step(self, msg):
        # It is required to check the kind of message received as multiple
        # messages can trigger the step method.
        if is_msg(msg, hello_gdnc_mode_msgs.Event, "count"):
            self.log.info("ground mode count event: msg=%s", msg)
            self.mission.ext_ui_msgs.evt.sender.count(msg.count)
```

```python
IDLE_STATE = {
    "name": "idle",
    "class": Idle,
}

SAY_STATE = {
    "name": "say",
    "class": Say,
}
```

```python
GROUND_STAGE = {
    "name": "ground",
    "class": DEF_GROUND_STAGE["class"],
    "initial": "say",
    "children": [
        child
        for child in DEF_GROUND_STAGE["children"]
        if not child["name"] in _STATES_TO_REMOVE
    ]
    + [IDLE_STATE, SAY_STATE],
}
```
