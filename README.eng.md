# OpenLog GPS Neo-6m parsing NMEA $GPRMC
<p align="center">
  <img src="fotos/ligacoes.jpg" alt="Minha imagem" width="320" />
</p>

## Objective
Create a simple way to use a GPS and save the location of the places we've been.

## Hardware

Use the datalogger from Sparkfun (https://github.com/sparkfun/OpenLog/tree/master) and a GPS module from u-blox, or Neo-6m (https://www.u-blox.com/en/product/neo-6-series).

The Sparkfun datalogger is a very compact module (it could be less compact and provide easy access to some available I/O) that allows you to record data received on the RX pin onto an SD card. It has different operating modes, but none of them are enabled to parse the received data.


<p align="center">
  <img src="fotos/OpenLog.jpg" alt="Minha imagem" width="320" />
</p>

The u-blox Neo-6m GPS is very popular due to its ease of use. Simply plug it in and use it outdoors; it propagates NMEA frames on the TX pin. Examples of modules:
<p align="center">
  <img src="fotos/gps only with rxtx.png" alt="Minha imagem" width="320" />
</p>
<p align="center">
  <img src="fotos/gps with usb and rxtx.jpg" alt="Minha imagem" width="320" />
</p>

This latest model has two interfaces available, RX/TX and USB . It is possible to use them **simultaneously**, the USB interface connected to a PC with the u-center software, https://www.u-blox.com/en/product/u-center , and the RX/TX pins connected to an MCU.



## Software

By connecting the GPS's TX pin to the datalogger's RX pin and selecting Mode Rotate, it records everything you send to it. However, NMEA frames are not easy for humans to read, nor is recording them simple to use without a time reference.

The Neo-6m sends out frames every second, the content of which depends on its current synchronization state.

After turning on and not syncing:
```
15:25:22  $GPRMC,,V,,,,,,,,,,N*53
15:25:22  $GPVTG,,,,,,,,,N*30
15:25:22  $GPGGA,,,,,,0,00,99.99,,,,,,*48
15:25:22  $GPGSA,A,1,,,,,,,,,,,,,99.99,99.99,99.99*30
15:25:22  $GPGSV,1,1,01,24,,,33*7E
15:25:22  $GPGLL,,,,,,V,N*64
```

We already have enough signal to send the time/date:
```
15:31:57  $GPRMC,153157.00,V,,,,,,,051125,,,N*7B
15:31:57  $GPVTG,,,,,,,,,N*30
15:31:57  $GPGGA,153157.00,,,,,0,00,99.99,,,,,,*62
15:31:57  $GPGSA,A,1,,,,,,,,,,,,,99.99,99.99,99.99*30
15:31:57  $GPGSV,1,1,01,24,,,28*74
15:31:57  $GPGLL,,,,,153157.00,V,N*4E
```

And it's synchronized, also sending coordinates, speed, altitude, course, ...
```
16:16:29  $GPRMC,161629.00,A,3828.22358,N,00849.99768,W,3.805,,051125,,,A*4F
16:16:29  $GPVTG,,T,,M,3.805,N,7.048,K,A*26
16:16:29  $GPGGA,161629.00,3857.22358,N,00859.99768,W,1,04,14.33,100.7,M,48.5,M,,*71
16:16:29  $GPGSA,A,3,06,11,12,24,,,,,,,,,17.05,14.33,9.23*0F
16:16:29  $GPGSV,3,1,09,06,06,047,18,10,07,243,,11,10,085,30,12,55,042,17*77
16:16:29  $GPGSV,3,2,09,24,34,112,30,25,74,318,,28,25,304,19,29,39,176,11*7C
16:16:29  $GPGSV,3,3,09,32,52,285,20*4B
16:16:29  $GPGLL,3857.22358,N,00859.99768,W,161629.00,A,A*7A
```

If we record plots without any kind of filter, we store a lot of unnecessary information, and without knowing where the data is located, it becomes more difficult to use it.

So, I used the software provided by Sparkfun ( https://github.com/sparkfun/OpenLog/tree/master/firmware/OpenLog_Firmware/OpenLog ), added a new operating mode, MODE_GPS, parsed the $GPRCM frame, and saved only the most frequently used data.

The $GPRCM frame has 3 types of formats:

Not synchronized:
```
	$GPRMC,,V,,,,,,,,,,N*53
```

With time/date:
```
	$GPRMC,153157.00,V,,,,,,,051125,,,N*7B
```

And synchronized:
```
$GPRMC,161629.00,A,3855.22358,N,00849.99768,W,3.805,,051125,,,A*5F
		  |		 |     |      |     |       |   |       |
		  |		 |     |      |     |       |   |       - date
		  |		 |     |      |     |       |   - speed       
		  |		 |     |      |     |       - East/West   
		  |		 |     |      |     - latitude
		  |		 |     |      - North/South
		  |		 |     - longitude
		  |		 - sinc. (A)/ not sinc. (V)     
		  - hour     
```
From this last format, one can extract the time, longitude, latitude, date and speed, more than enough for most applications.
The data is saved to the SD card every minute in a text file whose name is the date it was obtained, and only if the GPS is synchronized.
The date has been manipulated to appear as YYMMDD for easier sorting.

The saved files contain lines with time, longitude and latitude, already converted to degrees, and speed in km/h, for example:

```
14:35:00; 38.71690; -9.13983; 0.740;
14:36:00; 38.71664; -9.14000; 0.926;
14:37:00; 38.71674; -9.14010; 0.740;
14:38:00; 38.71685; -9.13996; 1.111;
```

The original Sparkfun program remains as it was, with only this added functionality (MODE_GPS).

When the program starts, it looks for the configuration file, "config.txt", and creates it if it doesn't exist.
The contents of the configuration file are:
```
9600,36,3,4,1,1,0,100,100
baud,escape,esc#,mode,verb,echo,ignoreRX,maxFilesize,maxFilenum
```
The differences from the original file are:
```
The escape character to enter in configuration mode is $ (36).
The mode is MODE_GPS (4).
```

Next, it creates the "dummy.txt" file. This file is only used at the beginning so that the program doesn't get stuck and enter the loop that waits for valid data from the $GPRMC frame or 3 characters $ to enter configuration mode.
From here, you wait for valid data and write it, every minute, to a text file whose name is the date it was acquired.

# Connections

To program the datalogger, a USB/TTL converter, such as an FTDI or similar, is used.

See https://learn.sparkfun.com/tutorials/openlog-hookup-guide/all

<p align="center">
  <img src="https://cdn.sparkfun.com/assets/learn_tutorials/4/9/8/OpenLogFTDI.png" alt="Minha imagem" width="320" />
</p>



To connect the GPS to the datalogger, the connection between the modules must be crossed, that is, the TX pin of the GPS connects to the RX pin of the datalogger. The RX pin of the GPS should remain unconnected (it is not necessary).


<p align="center">
  <img src="fotos/ligacoes.jpg" alt="Minha imagem" width="320" />
</p>



Most modules on the market have a voltage regulator that allows for 5VDC power supply; however, it's always good to check.
It's also important to remember that the Neo-6m is not 5V tolerant.


