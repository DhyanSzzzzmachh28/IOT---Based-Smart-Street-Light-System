# IOT---Based-Smart-Street-Light-System
Using ESP32 we designed an automated street light
Hackathon Project Report: IoT-based Smart Street Light System
Track: Internet of Things (IoT) & Hardware
Team ID: — T053
Team Members: —4
● Member Name : Nikhil R Gunagi
                                        Rahul L B
                                        Dhyan S
                                        Ananya N
Repository Link: 
 https://github.com/DhyanSzzzzmachh28/IOT---Based-Smart-Street-Light-System
Demo Link: 
https://jumpshare.com/share/cxJWxBkjp6gyNkMfBlH2
________________________________________
1. Project Abstract
The IoT-based Smart Street Light System automates street lighting based on real-time ambient brightness using an ESP32 microcontroller and an LDR sensor.
 When the environment becomes dark, the system turns the street light ON; during daylight, it turns it OFF automatically. 
The ESP32’s Wi-Fi capability also enables IoT-based monitoring, significantly reducing energy wastage and manual operation.
________________________________________
2. Problem Statement
Inefficient street lighting leads to significant energy wastage and increased operational costs. 
This project presents an IoT-based Smart Street Light System designed to automate lighting control based on ambient environmental conditions. 
Using an ESP32 microcontroller interfaced with an LDR (Light Dependent Resistor), the system continuously monitors sunlight intensity. 
When natural light falls below a specific threshold, the ESP32 automatically activates the LED street light; conversely, it switches the light off during the day. 
The system also leverages IoT connectivity to transmit status data for remote monitoring. 
This solution effectively reduces power consumption and eliminates the need for manual operation.
________________________________________
3. Solution Overview
The system uses an ESP32 connected to an LDR sensor to measure light intensity. 
Based on the readings, it switches the LED street light ON or OFF through GPIO23.
 The ESP32 continuously monitors the brightness via its ADC pin (GPIO34). 
If light falls below a set threshold, the LED turns ON; if brightness rises, it turns OFF.
Below is the Arduino IDE code used to implement the smart street light logic:
________________________________________
________________________________________
4. Hardware & Technical Stack
4.1 Components List
•	ESP32 microcontroller
•	LDR sensor
•	1k-ohm resistor
•	220-ohm resistor
•	LED (street light simulation)
•	Micro USB cable
4.2 Communication Protocols
•	Wi-Fi (optional IoT monitoring)
•	ADC (LDR reading)
•	GPIO output (LED control)
________________________________________
5. Circuit / Architecture Diagram
Connections:
•	LDR → GPIO34
•	LDR second leg → 3.3V
•	LDR + 1kΩ resistor → GND (voltage divider)
•	LED anode → GPIO23 via 220Ω resistor
•	LED cathode → GND

________________________________________
6. Prototype & Implementation
•	Assembled on a breadboard using ESP32, LDR, and LED.
•	Programmed using Arduino IDE.
•	ESP32 reads light levels and toggles LED automatically.

________________________________________
7. Challenges & Solutions
Challenge	Solution
LDR values fluctuating	Added threshold and smoothing
LED not switching reliably	Adjusted resistor values
Wi-Fi dropouts (optional IoT)	Added reconnection logic
Light sensitivity too high/low	Tuned ADC threshold

________________________________________
8. Scalability & Commercial Viability
•	Cost per unit is low, making mass deployment feasible.
•	In real street lights, LED can be replaced with a relay driver to control AC lamps.
•	Can integrate solar charging for smart energy saving.
•	IoT features make it suitable for smart city infrastructure.
 
 
                                               **************************************
