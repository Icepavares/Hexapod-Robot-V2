# ROS2 Architecture

- Due to the fact that I had the walking and gait logic from what I wrote in the previous robot, Hexapod V1, which powered by Arduino Mega. The whole core system between ROS2 and Arduino are hugely different.
- So, the architecture of the code needed to be changed.

## Why I Changed to ROS2

In the previous robot, most of the code was written in a single system running on Arduino Mega.  
For this project, I moved to ROS2 running on Raspberry Pi 5.

ROS2 works differently compared to Arduino projects.  
Instead of putting everything into one main program, ROS2 separates the system into multiple nodes and packages.

Because of that, I needed to redesign the software structure from the beginning.

---

# Current Software Structure

The software is separated into different parts so each node has its own responsibility.

## Motion Manager Node

This node acts like the main controller of the robot.

Responsibilities:
- Receive movement commands
- Handle walking and stopping states
- Control motion transitions
- Send movement data to the gait controller

For example:
- If the joystick is pushed forward → start walking
- If the joystick is released → return to standing position safely
- Also, this node will authorize the command that send from the controller(web-app controller) by refer to the robot state. If the robot is in Sitting Position, this node must know that in order to walk the robot must stand first. So, it prevent the state that could possibly harm the robot.

---

## Gait Controller Node

This node handles the walking pattern of the hexapod.

Responsibilities:
- Generate gait sequence
- Calculate leg movement timing
- Control walking cycle
- Convert movement commands into target leg positions

The gait logic is separated from hardware control to make the system easier to upgrade later.

This node is really important, In located in Gait package, that I can add more gait node to handle different task.

Futuremore, I separated the reusable math function from the node as a python file so I can use the function in another node also such as Bezier curve, swing and propel logic.

---

## Inverse Kinematics

This part calculates servo angles from target foot positions.

Responsibilities:
- Calculate joint angles for each leg
- Handle leg position math
- Convert coordinates into servo commands

This helps keep the walking system independent from the servo hardware.

---

## Servo Driver Node

This node handles communication between ROS2 and the bus servos.

Responsibilities:
- Receive target servo angles
- Send commands to servo controller
- Handle serial communication
- Keep low-level hardware control separated from higher-level logic

---

# Planned ROS2 Package Structure

```text
motion_manager
gait_controller
inverse_kinematics
servo_driver
robot_interfaces
robot_bringup
