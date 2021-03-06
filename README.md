# SENZ002-PM2.5-Laser-Dust-Sensor

###### Translation

> For `English`, please click [`here.`](https://github.com/njustcjj/SENZ002-PM2.5-Laser-Dust-Sensor/blob/master/README.md)

> For `Chinese`, please click [`here.`](https://github.com/njustcjj/SENZ002-PM2.5-Laser-Dust-Sensor/blob/master/README_CN.md)

![](https://github.com/njustcjj/SENZ002-PM2.5-Laser-Dust-Sensor/blob/master/pic/SENZ002.jpg "SENZ002") 

### Introduction

> PM2.5 laser dust sensor is a digital universal particle concentration sensor，it can be used to obtain the number of suspended particulate matter in a unit volume of air within 0.3 to 10 microns, namely the concentration of particulate matter, and output with digital interface, also can output quality data of per particle. 
> The Air Quality sensors can be embedded in a variety of concentrations of environment-related instruments suspended particulate matter in the air,to provide timely and accurate concentration data.

### Specification

* Operating voltage :4.95 ~ 5.05V
- Maximum electric current: 120mA
- Measuring pm diameter: 0.3~1.0、1.0~2.5、2.5~10（um）
- Measuring pm range：0~500 ug/m3
- Standby current: ≤200 uA
- Response time: ≤10 s
- Operating temperature range:: -20 ~ 50C
- Operating humidity range: 0 ~ 99% RH
- Maximum size: 65 × 42 × 23 (mm)
- MTBF: >= 5 years

#### Feature

* Quick response
* Standard serial input word output
* Second-order multi-point calibration curve
* The minimum size of 0.3 micron resolution

#### Power supply quality requirements:

- Voltage ripple: less than 100mV.
- The power supply voltage stability: 4.95 ~ 5.05V.
- Power supply: more than 1W (5V@200mA).
- The upper and lower electric voltage surge is less than 50% of the system power supply voltage.

### Tutorial

#### Pin Definition

|Sensor Pin|Ardunio Pin|Function Description|
|-|:-:|-|
|V-LED|5V|Open drain drive input|
|LED-GND|GND||
|LED|Digital pin|Rating from -0.3V to 5V. Recommend pulse cycle=10±1ms, pulse width=0.32±0.02ms|
|S-GND|GND||
|Vo|Analog pin|Output|
|Vcc|5V||

![](https://github.com/njustcjj/SENZ002-PM2.5-Laser-Dust-Sensor/blob/master/pic/SENZ002_pin.jpg "Pin Definition") 

#### Connecting Diagram

![](https://github.com/njustcjj/SENZ002-PM2.5-Laser-Dust-Sensor/blob/master/pic/SENZ002_connect.png "Connecting Diagram") 

#### Sample Code


	/*
	Arduino Uno (ATMEGA328) Port Allocations:

	A0:     A/D:                     Dust Sensor Analog Signal
	D2:     INT0 I/P:                Dust Sensor Interrupt - Link to D9
	D9:     Timer 1 OC1A PWM O/P:    Dust Sensor Samples - Link to D2
	D10:    Timer 1 OC1B PWM O/P:    Dust Sensor LED Pulses
	*/

	#define VOLTAGE 5.0 // Arduino supply voltage

	// Download the Timer 1 Library (r11) from:
	// http://playground.arduino.cc/Code/Timer1
	// https://code.google.com/p/arduino-timerone/downloads/list

	#include <TimerOne.h>

	/*
	   Smoothing

	   Reads repeatedly from an analog input, calculating a running average.
	   Keeps readings in an array and continually averages them.

	   Created 22 April 2007
	   By David A. Mellis  <dam@mellis.org>
	   modified 9 Apr 2012
	   by Tom Igoe
	   http://www.arduino.cc/en/Tutorial/Smoothing

	   This example code is in the public domain.
	*/

	/* With thanks to Adafruit for the millis code - plagiarised from the Adafruit GPS Library */

	const int numReadings = 100;    // samples are taken at 100Hz so calculate average over 1sec

	int readings[numReadings];      // the readings from the analog input
	int readIndex = 0;              // the index of the current reading
	long int total = 0;             // the running total
	int latest_reading = 0;         // the latest reading
	int average_reading = 0;        // the average reading

	int dustPin = A0;

	// Initialisation routine

	void setup()  
	{    
	  Serial.begin(9600);

	  // Configure PWM Dust Sensor Sample pin (OC1A)
	  pinMode(9, OUTPUT);
	  // Configure PWM Dust Sensor LED pin (OC1B)
	  pinMode(10, OUTPUT);
	  // Configure INT0 to receive Dust Sensor Samples
	  pinMode(2, INPUT_PULLUP);

	  // Put Timer 1 into 16-bit mode to generate the 0.32ms low LED pulses every 10ms
	  // A0 needs to be sampled 0.28ms after the falling edge of the LED pulse - via INT0 driven by OC1A (D9)
	  Timer1.initialize(10000); // Set a timer of length 10000 microseconds (or 10ms - or 100Hz)
	  Timer1.pwm(10, 991); // Set active high PWM of (10 - 0.32) * 1024 = 991
	  Timer1.pwm(9, 999); // Set active high PWM of (10 - 0.28) * 1024 = 995 BUT requires a fiddle factor making it 999

	  // Attach the INT0 interrupt service routine
	  attachInterrupt(0, takeReading, RISING); // Sample A0 on the rising edge of OC1A

	  // Initialise sample buffer
	  for (int thisReading = 0; thisReading < numReadings; thisReading++) {
	     readings[thisReading] = 0;
	   }
	}

	// Dust sample interrupt service routine
	void takeReading() {
	   // subtract the last reading:
	   total = total - readings[readIndex];
	   // read from the sensor:
	   latest_reading = analogRead(dustPin);
	   readings[readIndex] = latest_reading;
	   // add the reading to the total:
	   total = total + latest_reading;
	   // advance to the next position in the array:
	   readIndex = readIndex + 1;

	   // if we're at the end of the array...wrap around to the beginning:
	   if (readIndex >= numReadings) readIndex = 0;

	   // calculate the average:
	   average_reading = total / numReadings; // Seems to work OK with integer maths - but total does need to be long int
	}

	// Main loop

	uint32_t timer = millis();

	void loop()                     // run over and over again
	{
	  // if millis() or timer wraps around, we'll just reset it
	  if (timer > millis())  timer = millis();

	  // approximately every second or so, print out the dust reading
	  if (millis() - timer > 1000) { 
	    timer = millis(); // reset the timer

	    float latest_dust = latest_reading * (VOLTAGE / 1023.0);
	    float average_dust = average_reading * (VOLTAGE/ 1023.0);

	    Serial.print("Latest Dust Reading (V): ");
	    Serial.print(latest_dust);
	    Serial.print("\t\tAverage Dust Reading (V): ");
	    Serial.println(average_dust);
	  }
	}


### 购买[*SENZ002-PM2.5-Laser-Dust-Sensor*](https://www.ebay.com/).







