---
title: "Working with Arduino Remotely(for Robotics)"
date: 2025-09-06T10:00:00-05:00
draft: false
tags: ["arduino", "raspberry-pi", "robotics", "devops", "automation"]
categories: ["tutorials"]
summary: "Skip the Arduino IDE. Program and deploy Arduino firmware directly from your Raspberry Pi using Arduino CLI for seamless robotics development workflows."
---

I recently started working with Arduino for a small robotics project and the workflow with Arduino IDE went something like this -

1. Shut down the robot. 
2. Unplug the Arduino
3. Plug the Arduino into my computer. 
4. Chmod the mount point, for access. 
5. Start Arduino IDE
6. Make a small code edit. 
7. Re-attach the Arduino to to the robot. 
8. Boot the robot. 
9. Test the code. 
10. See the code didn't work, go back to step 1. 

This process isn't just slow, it's absolutely infuriating if you're trying to get something working, and every single code tweak requires a hardware hookup. There are two general approaches to fixing the problem here. The first would be to mount the robot at my desk and let it just stay plugged in, but that would require running the full robotics software stack locally, instead of the environment I'm trying to deploy it to. The second is to use the Arduino CLI to deploy updates directly from my remote robot(running on a Raspberry Pi) to the Arduino. That's what we're going to talk about here. 

## Installation

To get started, install Arduino CLI on your Raspberry Pi:

```bash
curl -fsSL https://raw.githubusercontent.com/arduino/arduino-cli/master/install.sh | sh
sudo mv bin/arduino-cli /usr/local/bin/
```

Initialize and install your target platform:

- arduino-cli config init
- arduino-cli core update-index
- arduino-cli core install arduino:avr

Fix permissions for USB access:

- sudo usermod -a -G dialout $USER

## Project Structure

Arduino CLI requires proper folder structure. Your `.ino` file must be in a folder with the same name:

```
robot_control/
└── robot_control.ino
```

**Not:**
```
firmware/
└── robot_control.ino  # Folder/file name mismatch
```

## Library Management

Install libraries, as needed for your use case. I needed the pid library. 

- arduino-cli lib search PID
- arduino-cli lib install "PID"
- arduino-cli lib list

For GitHub-hosted libraries:

- arduino-cli lib install --git-url https://github.com/user/library.git

## Compilation and Upload

### Basic workflow:

These are the steps you're going to run over and over again as part of your deployent workflow. I would recommend baking them into a "deploy.sh" script for re-use. 

- arduino-cli compile --fqbn arduino:avr:uno /path/to/sketch/
- arduino-cli upload -p /dev/ttyUSB0 --fqbn arduino:avr:nano /path/to/sketch/


### Board detection:

If you're not using Arduino Nano, or you're not sure - 

arduino-cli board list

### Common FQBN strings:
- Arduino Uno: `arduino:avr:uno`
- Arduino Nano (new bootloader): `arduino:avr:nano:cpu=atmega328`
- Arduino Nano (old bootloader): `arduino:avr:nano:cpu=atmega328old`


## Troubleshooting Upload Issues

The two upload issues I've run into are permissions and bootloaders. 

1. **Wrong bootloader** - A lot of Nano clones use old bootloaders. To get around that, add the bootloader at the end of the fqbn, as shown. 
   - arduino-cli upload -p /dev/ttyUSB0 --fqbn arduino:avr:nano:cpu=atmega328old sketch/

2. **USB permissions** - If you run into permissions issues, don't forget to chmod your Arduino. Make sure that /ttyUSB0 is actually your Arduino before you do though. 
   - sudo chmod 666 /dev/ttyUSB0
   ```



