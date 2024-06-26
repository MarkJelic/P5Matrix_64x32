# PxMatrix - LED matrix panel driver for ESP8266, ESP32 and ATMEL

<center><img src=./images/front_anim2.gif></center>

## Overview

This driver controls Chinese RGB LED Matrix modules without any additional components whatsoever. These panels are commonly used in large-scale LED displays and come in different layouts and resolutions:

Pitch (mm)| Size | Resolution | Rows-scan pattern
----|----|----|----
P6   | 192mmx96mm  | 32x16 | 1/2, 1/4 or 1/8
P10  | 320mmx160mm | 32x16 | 1/2, 1/4 or 1/8
P4   | 128mmx128mm | 32x32 | 1/8 or 1/16
P5   | 160mmx160mm | 32x32 | 1/8 or 1/16
P6   | 192mmx192mm | 32x32 | 1/8 or 1/16
P7.62| 244mmx244mm | 32x32 | 1/8 or 1/16
P2.5 | 160mmx80mm  | 64x32 | 1/16
P4   | 256mmx128mm | 64x32 | 1/16
P5   | 320mmx160mm | 64x32 | 1/16
P2.5 | 160mmx160mm | 64x64 | 1/32
P3   | 192mmx192mm | 64x64 | 1/32

Multiple panels may be chained together to build larger displays. The driver is Adafruit GFX compatible and is optimized for low pin count. It does not require DMA capabilities and works with ESP8266, ESP32 and Atmel-based Arduinos (only up to 64x32). If your microcontroller has anought I/O pins and dedicated DMA hardware you may want to consider more optimized libraries such as the [ESP32 I2S library](https://github.com/mrfaptastic/ESP32-HUB75-MatrixPanel-I2S-DMA) by mrfaptastic.

This readme gives an overview over the library - for a more detailed guide check out @witnessmenow's detailed [tutorial](http://www.instructables.com/id/RGB-LED-Matrix-With-an-ESP8266/).


## Display structure

The display basically consists of 6 large shift register. On the input connector you will find the inputs to the shift register (two for each color - Rx,Gx,Bx), a 2 to 5 bit latch address input (A,B,C,D,E), a latch enable input (LAT/STB), a clock input (CLK) and the output enable input (OE).

There are a few basic row scanning layouts: 1/4, 1/8, 1/16 and 1/32 row scan. You can enable the correct pattern for your display with display.begin(n) where n={4,8,16,32} defines the pattern.

For example, the 32x16 displays work like this (other varieties operate accordingly): Each of the shift register is 64(1/4 row scan) / 32(1/8 row scan) bits long. R1 and R2 will together therefore cover 128(1/4 row scan) / 64(1/8 row scan) bits or 4(1/4 row scan) / 2(1/8 row scan) lines respectively. The rows are, however, not next to each other but have a spacing of 4(1/4 row scan) / 8(1/8 row scan). In case of 1/4 the row scan pattern may also alternate between rows or even reverse bit order.

Such LED matrix are usually used as a sub-module for larger displays and therefore feature an output connector for daisy chaining. On the output connector you will find the identical signals to the input connector where A,B,C,LAT,CLK are simply routed through and (R,G,B) pins are the outputs of the shift registers on the module.

## Configure the library for your panel
The number of required address lines (A,B,C ...) in the constructor depends on the row-scan pattern and the multiplex pattern. See cabling-section for that. There are also three parameters that define how the panel works. 
### Row scanning
The first one is the basic row scanning layout explained above. You can specify this in the `display.begin(x)` call where x={4,8,16,32} is the scanning layout. 
### Scan pattern
Secondly, you may have to specify a different scanning pattern to the default LINE scanning. This can be achieved by calling `display.setScanPattern(x)` where x={LINE, ZIGZAG,ZZAGG, ZAGGIZ, WZAGZIG, VZAG, ZAGZIG, WZAGZIG2}. This defines where parts of your picture end up on the matrix. Change this if your picture is broken up into pieces at the wrong spots.
### Mux pattern
Finally, most panels use BINARY mapping of inputs A,B,C,D,E to the physical rows. But your panel may handle row multiplexing differntly. Some panels with 1/2 or 1/4 row scanning will map A,B,C,D(,E) STRAIGHT through to rows 0-4. Others may use a Shift-Register connected to A,B,C for row selection (SHIFTREG_ABC), like ones with Chip RT5957. For those, alternative cabling which frees up some IOs is possible with SHIFTREG_SPI_SE. See cabling section. However, this mode is barely tested and will not work with `setFastUpdate(true)`!

### Selection
Select the right pattern by calling `display.setMuxPattern(x)` where x={BINARY, STRAIGHT, SHIFTREG_ABC, SHIFTREG_SPI_SE}.
So for some very strange displays you may have execute:

```
display.begin(4);
display.setScanPattern(ZAGGIZ);
display.setMuxPattern(STRAIGHT);
```

## Set-up and cabling

When driving a long chain of LED modules in a row, parallel color data lines make a lot of sense since it reduces the data rate. But since we are only driving a few modules here, we really don't need that. We can therefore use jumper wires between input connector (PI) and output connector (PO) to chain all shift registers together and create one big shift register. This has two advantages: it reduces the number of required GPIO pins on the microcontroller and we can use the hardware SPI interface to drive it.

* If your panel input connector has "R1" in the top left corner:

  ![P10_conn_A](/images/32x16_con_A.jpg)

  Connect PI and PO as follows:

  PI | PO
  ---|---
  R2 | R1
  G1 | R2
  G2 | G1
  B1 | G2
  B2 | B1

  Connect panel input (PI) to the ESP8266 / ESP32 as follows:

  PI  | ESP8266 GPIO (NodeMCU) | ESP32 GPIO | ATMEGA GPIO |NOTE
  ----|----|----|----|----
  A   |  05 - (D1) | 19 | 2
  B   |  04 - (D2) | 23 | 3
  C   |  15 - (D8) | 18 | 4 | only for 1/8, 1/16, 1/32 scan BINARY mux pattern or 1/4 STRAIGHT mux pattern
  D   |  12 - (D6) | 5 | 5 | only for 1/16, 1/32 scan BINARY mux pattern or 1/4 STRAIGHT mux pattern
  E   |  00 - (D3) | 15 | 6 | only for 1/32 scan
  STB/LAT |  16 - (D0) | 22 | 7
  P_OE|  02 - (D4) | 16  | 8
  CLK |  14 - (D5) | 14 | 13(168/328), 52(2560)
  R1  |  13 - (D7) | 13 | 11(168/328), 51(2560) 

* If your panel input connector has "R0" in the top left corner:

  ![P10_conn_B](/images/32x16_con_B.jpg)

  Connect PI and PO as follows:

  PI | PO
  ---|---
  R1 | R0
  G0 | R1
  G1 | G0
  B0 | G1
  B1 | B0

  Connect panel input (PI) to the ESP8266 / ESP32 as follows:

  PI  | ESP8266 GPIO (NodeMCU) | ESP32 GPIO | ATMEGA GPIO | NOTE
  ----|----|----|----|----
  A   |  05 - (D1) | 19 | 2 | 
  B   |  04 - (D2) | 23 | 3 | optional on SHIFTREG_ABC (always LOW)
  C   |  15 - (D8) | 18 | 4 | only for 1/8, 1/16, 1/32 scan BINARY mux pattern or 1/4 STRAIGHT mux pattern or SHIFTREG_ABC mux pattern
  D   |  12 - (D6) | 5 | 5 | only for 1/16, 1/32 scan BINARY mux pattern or 1/4 STRAIGHT mux pattern
  E   |  00 - (D3) | 15 | 6 | only for 1/32 scan
  STB/LAT |  16 - (D0) | 22 | 7
  P_OE|  02 - (D4) | 16 | 8
  CLK |  14 - (D5) | 14 | 13(168/328), 52(2560) 
  R0  |  13 - (D7) | 13 | 11(168/328), 51(2560) 

You should end up with something like this (VCC/supply not connected here yet):

![Cabling](/images/P10_cables.jpg)

If you want it more professional, some users have created custom PCBs to get rid of all those lose cables.

  * [Mike](https://github.com/mike-rankin/ESP8266_RGB_Matrix_Cable_Version)
  * [Peppe](http://www.instructables.com/id/tabuled)
  * [Brian Lough (@witnessmenow)](https://www.tindie.com/products/brianlough/esp32-matrix-shield-mini-32/)

## ABCDE cabling 
The ABCDE cabling depends on the mux pattern used by your panel. For the majority of panels ABCDE is connected as shown in the previous section, however, for some variants e.g. using RT5957 chips, it may be different.


### BINARY (default)
A 1/4 scan display with scan pattern BINARY will require only A and B. 1/8 requires A-C, 1/16 A-D and 1/32 A-E to be connected as above. A good hint is usally the connector labeling on your matrix.

### STRAIGHT 
A STRAIGHT 1/2 display will require A and B to be connected as above. A STRAIGHT 1/4 display will require A-D to be connected as above.

### SHIFTREG_ABC - for RT5957 panels
SHIFTREG_ABC (with RT5957 or other Shift-Register for row selection) will require connecting A,B,C with B always beeing LOW (you may connect GND instead and reuse the IO for other stuff). So this needs extra pins on your microcontroller.

### SHIFTREG_SPI_SE - for RT5957 panels (experimental)
SHIFTREG_SPI_SE is a functionally identical variant of SHIFTREG_ABC where we use the SPI signal carrying the display data also for the row selection logic. This frees up three exta I/O pins required by SHIFTREG_ABC. To make this work you will have to connect the outputs of your last panel to the ABC inputs of the first panel - in particular A (IN) needs to be connected to Clock (OUT), C (IN) to Data/Blue_2 (OUT), B (IN) to GND. 

### SHIFTREG_ABC_BIN_DE
SHIFTREG_ABC_BIN_DE is for really weird panels using a combination of Shift-Register (e.g. SM5266PH) and also binary decoding for addressing the rows. A,B,C are connected to a 8 rows wide Shift-register Clock, Data, /Enable. Additionally, D-E control which block of 8 rows is used through binary decoding. This is for 1/16 and 1/32 panels only.

## Colors
The number of color levels can be selected in the header file. The default (8 color levels per primary RGB color) works well with hardly any flickering. Note that the number of color levels determines the achievable display refresh rate. Hence, the more color levels are selected, the more flickering is to be expected. If you run into problems with flickering it is a good idea to increase the CPU frequency to 160MHz. This way the processor has more headroom to compute the display updates and refresh the display in time.

## Chaining
 Chaining any number of displays together horizontally is rather straightforward. Simply use the supplied flat band connector between the panels and then treat the entire chain as one display. For example, three 32x16 displays would result in one 96x16 display where we use the input connector (PI) on the first and the output connector (PO) on the last panel as explained above. Don't forget to specify the correct resolution in the constructor, i.e. PxMATRIX display(96,16,...), and how many panels you chained together, i.e. display.setPanelsWidth(3). Initializing this chaining set-up with 1/4 scan panels would look like this:

```
#include <PxMatrix.h>
PxMATRIX display(96,16,...);
...
[]
...
void setup(){
display.begin(4);
display.setPanelsWidth(3);
...
```

![Chaining](/images/chain.png)


## Troubleshooting

  * Some panels have a slow multiplexer and only a partial image is displayed.
  To remedy this you can add some delay to the multiplexing using, for example, `display.setMuxDelay(1,1,1,1,1)` which would add a 1us delay to each of the A-E channels.<br><img src=./images/half_mario.jpg width="400">

  * If you have any problems with ghosting or randomly lit-up pixels, please double-check the ground connection between ESP and your panel and make sure that your power supply can deliver >2A. Also make sure that your cabling between power suppply and power connector (center of the panel) is sufficient to carry the current.

  * If you still problems with ghosting, shifted images or randomly lit-up pixels make sure that unused (multiplex) inputs are grounded. For example, some 1/16 scan panels expose an (sometimes unlabeled) E input that needs grounding where only ABCD inputs are connected to the ESP.

  * Check you cabling with a multimeter (diode-test). You can measure the connection between the input/ouput panel connector and the NodeMCU/ESP8266 via the exposed SMD pads/legs.

  * Your display may have a different scanning pattern. Make sure that you have selected the correct scanning pattern in the `display.begin()` call.

  * Run the "pattern_test.ino" and check if the scanning pattern is appearing ok. For a 8 row-step display with LINE scanning pattern it should look like this (red then yellow then white line progressing): <br><img src=./images/8step.gif width="400">

## Thanks to
  * xsrf for many useful additions and working the trenches in the issue tracker
  * Kinsey Moore for improving the code
  * Jean Carlos for testing the chaining
  * Dave Davenport for FM62126A support
  * Rasmusfk and Maciej Matczak for double buffering support
  * Brian Lough (@witnessmenow) for hardware donations and a continuous stream of cool projects with PxMatrix
  * Peppe Monnezza for hardware donations

If you would like to buy me a beer: [![paypal](https://www.paypalobjects.com/en_US/i/btn/btn_donateCC_LG.gif)](https://www.paypal.com/cgi-bin/webscr?cmd=_s-xclick&hosted_button_id=TXLWSPUD9X2KL)


## Examples

Animated weather clock from [here](https://2dom.github.io/PixelTime/)  (ESP8266)

![Example](/images/PixelTime_small.jpg)

Animated picture frame from [here](https://2dom.github.io/PixelTimes/) (ESP8266 - Pixels from [eboy](http://hello.eboy.com/eboy/category/everything/explore/animations/))

![Example](/images/front_anim2.gif)

Decoding 10 FPS MJPEG on 64x64 RGB LED matrix (ESP32)

<nobr>![Example](/images/starwars.gif)

[Particle physics by witnessmenow](https://twitter.com/witnessmenow/status/1163039308254760960) on 64x64 RGB LED matrix (ESP32)

![Particle](/images/particle.JPG)

https://www.adafruit.com/product/2277
