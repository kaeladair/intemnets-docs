---
description: Guide to writing files to the drones internal memory.
---

# Writing Files

## **Mission Data Storage**

The Air SDK provides a dedicated mission data directory in the drone's internal memory that persists across reboots. This directory is where you'd typically write and read mission-specific data.

In Python, you can access this directory using the following code snippet within the Flight Supervisor state machine:

```python
mission_data_root = self.mission.env.get_root_data_dir()
```

You can also hardcode using this path:

```python
mission_data_root = '/mnt/user-internal/missions-data/{MISSIONS UID}/{FILE NAME}'
```

Once you have access to the directory, you can create and write to files using the standard Python IO operations. One such example below shows how to print a line of text every time the drone "nods" the camera in the hello sample mission.

## Hello Mission Example

In the hello mission, if we want to print a line to a text file every time the drone moves the camera, we need to go into the guidance mode file and find a piece of code that will execute once for every time the camera moves.

Here is some code that does just that, and operates as a timer that sends a message to other parts of the drone system to count the number of movements.

```python
def _timer_cb(self):
   self.log.info("Hello world")
   self.front_cam_pitch_index = 0
   self.say_count += 1

   msg = HelloGroundModeMessages.Event()
   msg.count = self.say_count
   gdnc_core.msghub_send(self.evt_sender, msg)
```

If we want to create and append to a file, we would add this code below:

```python
def _timer_cb(self):
   self.log.info("Hello world")
   self.front_cam_pitch_index = 0
   self.say_count += 1
   
   # Print Hello for every time the drone camera moves
   with open(PATH, 'a') as file:
      file.write("Hello\n")

   msg = HelloGroundModeMessages.Event()
   msg.count = self.say_count
   gdnc_core.msghub_send(self.evt_sender, msg)
```
