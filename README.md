 * ESP-NTP-DCF77-Encoder.
 *
 * ESP NTP Clock Encoder Injecting / Transmitting DCF77 (Bye pin or external AM Transmitter).
 * 
 * 
 * Time part working with ezTime = Auto TZ, DLS and much more goodies.
 * Using ezTimes "setEvent(advance,now()+1);" as Second ticker for the DCF keying.
 * Constant looping / keying / sending next minute.
 * Using switch for keying DCF and calculating the DCF arrey:
 *   case 3: Waiting for the first 59 second after boot and keying out 100 ms until second 58.
 *   case 2: Long pulse Modulation: 200 ms 
 *   case 1: Short pulse Modulation: 100 ms 
 *   case 0: Reading time and calculate the DCF array the next minute ( = in all 59 Seconds)
 * First minute after boot waiting for second 59 before start keying time out DCF time.
 * Using ezTime variables for calculating DLS on / off and Summer / Winter Time Announcement.
 * Also, for Day of week in DCF format.
 * 
 * Arduino OTA (Minimum delay in main loop and other places for OTA to working).
 * 
 * Not implanted: Sec 19, Leap Second announcement (inserting a mark in Sec 59 and sending minute sync in (not existing)Sec 60. 
 * 
 * ezTime rolls over (32 bit signed integer overflows) in 7th of February 2036 and 19th of January 2038, 
 * so some thing like these can being good to putting in:
 * #define MIN_TIME  1582165202     // Date and time (GMT): Thursday, February 20, 2020 2:20:02 AM  
 * #define MAX_TIME  2082762061     // Date and time (GMT): Tuesday, January 1, 2036 1:01:01 AM
 *  if (processedTime<MIN_TIME || processedTime>MAX_TIME) {
 *  Serial.println("You being out of time !!!");
 *  }
 * 
 * NTP update inside ezTime (If to much drift it can being problem with the timing of the DCF keying,
 * also if problem with NTP update its taking to long or not working (NTP Server or network problems).  
 * If NTP not getting time sync then boot its retrying until its getting time before entering main loop,
 * (Not looking nice but better then starting in the cold winter (-6.8° under mean) of 1970).
 * 
 * 
 * Sample Serial out put of around one sec 59:

55 Second, 200 ms = 1

56 Second, 100 ms = 0

57 Second, 100 ms = 0

58 Second, 200 ms = 1

59 Second, No pulse = New Minute. Reading time and calculate the DCF array the next minute.

EZ Date and Time: Mittwoch, 26-Feb-2020 17:22:59 CET

Unix Timestamp: 1582737779

Unix Time + TZ +120 Timestamp: 1582741499

0-00000000000000-000101-00100100-1110100-011001-110-01000-000001001 

0 Second, 100 ms = 0

1 Second, 100 ms = 0

2 Second, 100 ms = 0

3 Second, 100 ms = 0

 * For live and historical DCF reference: https://www.dcf77logs.de/live
 * 
 * Custom config:
 * 
 * Timezone Austria; //A Name for Your Time Zone 
 * #define WIFI_SSID "Your SSDI"
 * #define WIFI_PASS "Your WiFi Pass"
 * #define OTA_NAME "Device OTA Name" // ESP01S DCF77 Test
 * #define OTA_PASS "Device OTA Pass"
 * #define LedPin 2 // Using the LED pin as DCF output. Define other pin if DCF using other output pin
 * Austria.setPosix("CET-1CEST,M3.5.0,M10.5.0/3"); // Posix for Your Time Zone
 * Austria.setDefault(); // Setting Your time Zone as default
 * setInterval(1800); // NTP update interval
 * setServer("ntp.se"); // Your NTP Serverpool
 * 
 *
 * Code its partly based on the Swiss code at:

https://www.changpuak.ch/electronics/Arduino-DCF77.php
 
////////////////////////////////////////////////////////////// 
ARDUINO/Genuino (UNO) Test/Demo Sketch for Low Frequency
ASK Synthesizer
Software Version 2.4, 
24.04.2017, Alexander C. Frank
//////////////////////////////////////////////////////////////

 * Witch contain traces of German and Italian :)
 * 
 * All mods and adds are Copyright 2020 Mattias Westerberg and are free for private use only.
 * And WITHOUT ANY WARRANTY and on your own risk !!!
 * 
 * 
 * Alexander have made a very nice looking DCF77 Transmitter that using a SiT8208AC for 
 * transmitting and keying DCF77.
 *  
 * In My AVR-DCF-Transmitter i using a 16 MHz ATmega328P for transmitting the keyed DCF77.
 *  
 * Andreas Spiess have a working DCF77 Transmitter for ESP32 at: 
 * https://github.com/SensorsIot/DCF77-Transmitter-for-ESP32.
 * (With German, Spanish and Italian trace in the Swiss code :))
 * 
 * 
 * Example updating one old weather station with one ESP-01S:
 * If the power in the weather station (normally around 3 volt) its in working range connect GND and VSS, 
 * if not putt one Step-Down / Buck Converter or LDO between.
 * Find the data out pin from the DCF module or IC. Cut the cable or PCB trace. 
 * Connect the GPIO2 to the data pin on the MCU side. 
 * If having a module you can demounting it and using it for other things / experiments.
 * 
 * Most clocks needs 2 sets of correct data for sinking the clock and its normally waiting for the
 * minute sync (second 59) before starting decoding data, so after 2 - 3 minutes you have  a 
 * well synced DCF77 clock :))).
 * 
 * If having one ATmega328P with 16 MHz crystal laying around you can using it as a transmitter 
 * and don’t need doing hardware mods. 
 * Just connect GPO2 from your ESP8266 to ATMegas D2 and GND to GND, and your antenna to D3. 
 * Flash my AVR-DCF-Transmitter on the ATMega and put the antenna near / around your clock.
 * 
 * One real ESP-01S have 1M flash and all pullups and downs and need only VCC and GND for running your code.
 * GPIO2 have pullup (Internal LED and resistor), but must being high on boot.
 * Its normally the right polarity and current for connecting to a weather stations MCU with out problems.
 * If polarity its wrong only shifting the the commands HIGH and LOW for the 100 and 200 ms pulses. 
 * Its also have large pads for the flash and make it easy to upgrade the flash if NEEEEEEDED !!
 * 
 * The original (blue 512K flash) and updated (black 1M flash) don’t have all resistors and need 
 * some to being attached for working and have smaller pads for the flash if need upgrading the it.
 * 
 * WARNING !! 
 * Manny sellers selling the updated ESP-01 version (black 1M flach with out "S") as ESP-01S 
 * (look on the PCB and counting components). If having problem flashing or getting it running
 * you need some resistors attached.
 * 
 * If you need one GPIO that can being LOW at boot (GPIO 0 and 2 cant = Not booting) 
 * then use GPIO3 (the normal RXD0) and define UART0 to use alternate pins GPIO13 and GPIO15
 * or disable it in your code.
 * 
 *
