# SENZ002 PM2.5激光粉尘传感器

###### 翻译

> `英文`请参考 [`这里`](https://github.com/njustcjj/SENZ002-PM2.5-Laser-Dust-Sensor/blob/master/README.md).

> `中文`请参考 [`这里`](https://github.com/njustcjj/SENZ002-PM2.5-Laser-Dust-Sensor/blob/master/README_CN.md).

![](https://github.com/njustcjj/SENZ002-PM2.5-Laser-Dust-Sensor/blob/master/pic/SENZ002.jpg "SENZ002") 
 

### 产品介绍

> PM2.5激光粉尘传感器是一款数字式通用颗粒物浓度传感器，可以用于获得单位体积内空气中0.3～10微米悬浮颗粒物个数，即颗粒物浓度，并以数字接口形式输出，同时也可输出每种粒子的质量数据。

> 
> 用途：嵌入各种与空气中悬浮颗粒物浓度相关的仪器仪表或环境改善设备，为其提供及时准确的浓度数据。

### 产品参数

* 工作电压 : 4.95 ~ 5.05V
* 最大工作电流 : 120mA
* 测量范围  : 0.3~1.0、1.0~2.5、2.5~10微米（um）
- 量程  :0~500 ug/m<sup>3</sup>
+ 待机电流  : ≤200 微安（uA）
+ 响应时间  : ≤10 s
- 工作温度范围 : -20 ~ 50摄氏度
- 工作湿度范围 : 0~99% RH
- 最大尺寸  : 65×42×23（mm）
- 平均无故障时间 : >=5年

#### 主要特性：

- 响应迅速
- 标准串口输字输出
- 二阶曲线多点校准
- 最小分辨粒径0.3微米

#### 供电电源质量要求：

1. 纹波小于100mV;
2. 电源电压稳定度：4.95～5.05V;
3. 电源功率：大于1W (电流大于200mA);
4. 上下电电压冲击小于系统电源电压的50%。

### 使用教程

#### 引脚定义

|Sensor Pin|Ardunio Pin|Function Description|
|-|:-:|-|
|V-LED|5V|Open drain drive input|
|LED-GND|GND||
|LED|Digital pin|Rating from -0.3V to 5V. Recommend pulse cycle=10±1ms, pulse width=0.32±0.02ms|
|S-GND|GND||
|Vo|Analog pin|Output|
|Vcc|5V||

![](https://github.com/njustcjj/SENZ002-PM2.5-Laser-Dust-Sensor/blob/master/pic/SENZ002_pin.jpg "引脚定义") 


#### 连线图

![](https://github.com/njustcjj/SENZ002-PM2.5-Laser-Dust-Sensor/blob/master/pic/SENZ002_connect.png "连线图") 


### 示例代码

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


### 购买[*SENZ002 PM2.5激光粉尘传感器*](https://www.ebay.com/).







