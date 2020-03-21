# ROS wheelbase controller for the NEXUS Omni 4-Wheeled Mecanum Robot

![4WD Mecanum Wheel Mobile Arduino Robotics Car](4WD_Mecanum_Wheel_Robotics_Car.jpg  "4WD Mecanum Wheel Mobile Arduino Robotics Car")

## Required hardware
This wheel controller has been developed for the 4WD Mecanum Wheel Mobile Arduino Robotics Car 10011 from [NEXUS ROBOT](https://www.nexusrobot.com/product/4wd-mecanum-wheel-mobile-arduino-robotics-car-10011.html)

For teleoperation of the Nexus wheelbase a (wireless) game pad like the Logitech F710 is recommended. The base controller runs on a PC or a Raspberry Pi 3B with ROS Melodic (-desktop) installed. For the installation of ROS Melodic on a Rasberry Pi, it is recommended to use [Ubuntu MATE 18.04](https://ubuntu-mate.org/).

Logitech F710 | ROS Melodic
- | -
![Logitech F710](Logitech_F710.jpg  "Logitech F710") | ![ROS Melodic](Melodic.jpg  "ROS Melodic") 

## Installing ROS dependencies
This project has been tested with ROS Melodic. The project uses an external C++ library. Clone the required PID_Library in the `nexus_base_ros/lib` directory:

 `git clone https://github.com/MartinStokroos/PID_Controller.git`
 
 The following ROS packages must be installed:
 `sudo apt-get install ros-melodic-rosserial-arduino`
`sudo apt-get install ros-melodic-rosserial`
`sudo apt-get install ros-melodic-joy`

## Building the package
Place the package in the workspace *src* directory and run `catkin_make`from the root of the workspace.

## Generating the ros_lib for Arduino
It is possible to compile the Arduino firmware with the catkin make proces without making use of the Arduino IDE. However, now it is done with the Arduino IDE.
The Arduino IDE should be installed and the `Arduino/libraries` directory must exist in your home folder.
Generate the *ros_lib* with the header files of the custom message types used in this project.

1. cd into `Arduino/libraries`
2. type: `rosrun rosserial_arduino make_libraries.py .`

In the newly made `ros_lib` , edit `ros.h`. Reduce the number of buffers and down size the buffers to save memory space:

```
#elif defined(__AVR_ATmega328P__)

  //typedef NodeHandle_<ArduinoHardware, 25, 25, 280, 280> 
  typedef NodeHandle_<ArduinoHardware, 8, 8, 128, 128> 
```

## Flashing the firmware into the wheel controller board of the 10011 platform
* download and include the digitalWriteFast Arduino library from: [digitalwritefast](https://code.google.com/archive/p/digitalwritefast/downloads)
* clone and install the PinChangeInt library: `https://github.com/GreyGnome/PinChangeInt.git`

Program the *Nexus_Omni4WD_Rosserial.ino* sketch from the firmware folder into the Arduino  Duemilanove-328 based controller board of the 10011 platform.

**NOTE:** The Sonars cannot be used simultaneously with the serial interface and must be disconnected permanently!

## Description
This picture shows the rosgraph output from `rqt_graph`:

![rosgraph](/home/martin/catkin_ws/src/nexus_base_ros/rosgraph.png  "rosgraph")

* Topics
The *nexus_base_controller* node runs on the platform computer and listens to the *cmd_vel* topic and uses the x-, y and angular velocity inputs. The *cmd_vel* is from the geometry_msg Twist  type.
The *nexus_base_controller* node publishes odometry data based on the wheel encoder data, with the topic name *sensor_odom*.

*nexus_base* is a rosserial type node running on the wheelbase controller board. The code for this node can be found in the firmware directory. *nexus_base* listens to the *cmd_motor* topic and publishes the *wheel_vel* topic at a constant rate of 20Hz (default).
The data structure of the *cmd_motor* topic is a 4-dim. array of *Int16*'s. The usable range is from -255 to 255 and it represents the duty-cycle of the PWM signals to the motors. Negative values  reverses the direction of rotation.
The *wheel_vel* topic data format is also a 4-dim. array of *Int16*'s. 
*wheel_vel* is the raw wheel speed; the number of encoder increments/decrements per sample interval (0.05s default).
Three seconds after receiving a speed of '0' the motors are set Idle by stopping the PWM to save power.

*/joy* is the standard message from the joystick node. The *teleop_joy* node does the unit scaling and publishes the *cmd_vel* topic and forms the client for the ROS-services *EmergencyStopEnable* and *ArmingEnable*.

* Services
Node *nexus_base* runs two ROS-service servers. The first service is called *EmergencyStopEnable* and enables the emergency stop and the second service is called *ArmingEnable* and this service is to (re)arm the system.

* Block diagram of the Nexus base controller
The blockdiagram shows the internal structure of the *nexus_base_controller* node.

![Nexus base controller](base_controller_block_diagram.png  "Nexus base controller")

## Launching the example project
The tele-operation demo can be launched after connecting a game pad (tested with *Logitech F710*) and the wheel base USB-interface with the platform computer (small formfactor PC or Raspberry Pi 3B). Check if the game pad is present by typing:

`ls -l /dev/input/js0`

Check if the wheel base USB interface is present: 

`ls -l /dev/ttyUSB0`

Launch:

`rosrun nexus_base_ros nexus_teleop_joy `

The left joystick handle steers the angular velocity when moved from left to right. The right joystick controls the x- and y-speed.
The red button enables the emergency stop. The green button is for rearming the robot after an emergency stop.

## Known issues
When building the package for the first time, the following error may appear:
```
fatal error: nexus_base_ros/Encoders.h: No such file or directory
 #include "nexus_base_ros/Encoders.h"
          ^~~~~~~~~~~~~~~~~~~~~~~~~~~
compilation terminated.
```
The reason for this could be, that there is something wrong in the sequence of instructions in the `CMakeList.txt` file or that the dependencies are not finished before linking because the make proces is threaded. The workaround is:

```
cd build/
make -j4 nexus_base_ros
cd
catkin_make
```
#
Rosservice calls do not work with rosserial. ROS Melodic comes with rosserial version 0.8.0. rosserial 0.7.7 works. Workaround:
Download [rosserial](https://repology.org/project/rosserial/packages) 0.7.7 . Unzip and copy the following modules from the rosserial-0.7.7 package into the catkin src directory and rebuild the project:

```
rosserial
rosserial_client
rosserial_msgs
rosserial_python
rosserial_server

``` 
#
Bug: sometimes the wheels do not repond for a while when rearming from an emergency stop.
 