---
layout: course
title: HMI-2 Arduino Fundamentals and the Serial Port
date: 2017-06-02
---

** Lecture Video **

[![HMI-2 Arduino Fundamentals and the Serial Port](../assets/images/HMI-Portada-YouTube.jpg)](http://www.youtube.com/watch?v=)

1. Background Knowledge and Prerequisites
1. The Arduino Board
1. The Serial Port
1. Sending Commands using the Serial Monitor
1. Advanced Topic Sending a stream of commands and values to the Arduino

## Background Knowledge and Prerequisites
I'm assuming you are already familiar at least with the basics of microcontrollers and electronics. If not I strongly suggest you to take the time and follow this tutorials **(and if you are one of my students, then it's a must)**. Feel free to skip information you are already familiar with:

1. [Introduction to Microcontrollers](https://ti.tuwien.ac.at/ecs/teaching/courses/mclu/theory-material/Microcontroller.pdf) Chapter 2. Microcontroller Components. Pages 11 to 72.
1. [Installing Arduino on a Windows Machine](https://www.arduino.cc/en/Guide/Windows)
1. [What's an Arduino](https://www.arduino.cc/en/Guide/Introduction)
1. [Bare Minimum code needed](https://www.arduino.cc/en/Tutorial/BareMinimum)
1. [Blink a LED](https://www.arduino.cc/en/Tutorial/Blink)
1. [Using a PushButton](https://www.arduino.cc/en/Tutorial/Button)
1. [Digital Read Serial](https://www.arduino.cc/en/Tutorial/DigitalReadSerial)
1. [Read Analog Voltage](https://www.arduino.cc/en/Tutorial/ReadAnalogVoltage)

## The Arduino Board

So what's and Arduino Board

> Arduino is an open-source electronics platform based on easy-to-use hardware and software. Arduino boards are able to read inputs - light on a sensor, a finger on a button, or a Twitter message - and turn it into an output - activating a motor, turning on an LED, publishing something online. You can tell your board what to do by sending a set of instructions to the microcontroller on the board. To do so you use the Arduino programming language (based on Wiring), and the Arduino Software (IDE), based on Processing.
>
>[Arduino Intro](https://www.arduino.cc/en/Guide/Introduction)

In other words, an Arduino board is a **microcontroller** with a bunch of other **useful components** attached to it, these components allow you to easily prototype and test new ideas with few lines of code an the bare minimum extra hardware.

I dare to say that Arduino may be very famous for students but it is a tabu in the industry. Experienced engineers undervalue the power of an Arduino saying it's merely a toy. In my personal opinion an Arduino is a perfectly capable developing board for prototyping and experiments and you can even use it in real final products with some tweaks, although coding in the Arduino Programming Language is not scalable nor maintainable you can always use pure C or any other language supported by AVR.

We use an **Arduino** for this course because we don't care about complicated hardware, performance or code scalaility, we are concerned only with the basics of the Serial communication and how to design an easy to use intuitive GUI.

### Basic Inputs and Outputs

Microcontrollers are used because they allow a piece of hardware to interact with it's environment "easily".

Imagine you want to build a "robot" (just imagine it...), you want this robot to be aware of it's environment, what I mean with this is that you want this robot to "see", "hear", "smell" and "feel" everything around it.
You may want also to make this robot interact with the environment, you may want it to "walk" or at least "move" and avoid obstacles, or maybe you want to speak an order and make the robot obey, for example saying "make me a sandwich" will make the robot go to the fridge, fetch the ingredients and start making you a damn sandwich!!

Well that wasn't very realistic right now, but a microcontroller can help you retrieving signals from the environment and cotrol actuators in order to make your little wall-e behave decently.

The environment signals the micronctroller retrieve are the inputs and logically the signals sent to the actuators are the outputs.

I'll use an [Arduino UNO](https://www.arduino.cc/en/Main/ArduinoBoardUno) which uses an **ATmega328P** as an example for Inputs and Outputs.

In the next image there's a typical Arduino UNO board, this board
![Arduino UNO IO](../assets/images/hmi-2-Arduino-IO.jpg)

Here is how the pins of the microcontroller (Atmega168) are mapped to the board terminals. This image was extracted from [here](https://www.arduino.cc/en/Hacking/PinMapping168). If you want to see the full schematic visit: [link](https://www.arduino.cc/en/uploads/Main/Arduino_Uno_Rev3-schematic.pdf).
![Atmega168 Pin Map](../assets/images/hmi-2-Atmega168PinMap2.png)


### C coding for Arduino (just an Intro)

### How to upload the code to the Arduino Board

### Example: Using multiple input/outputs



## The Serial Port

### What's a Serial Port

### The signal

### Serial Communication in the Industry


## Sending Commands to Arduino via Serial using the Serial Monitor

### Example: Turn On/Off via Serial

### Example: Controlling multiple input/output's via Serial



## Advanced Topic Sending a stream of commands and values to the Arduino

### Example: Controlling multiple input/output's via Serial using a stream of commands