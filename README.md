# micropython-epaper
A driver to enable the Pyboard to access a 2.7 inch e-paper display from
[Embedded Artists](www.embeddedartists.com/products/displays/lcd_27_epaper.php)
This can be bought from [Adafruit](www.adafruit.com) and in Europe from
[Cool Components](www.coolcomponents.co.uk).

### Introduction

E-paper displays have high contrast and the ability to retain an image with the power
disconnected. They also have very low power consumption when in use. The Embedded
Artists model suopports monochrome only, with no grey scale: pixels are either on or off.
Further the display refresh takes time. The minimum update time defined by the manufacturer's
data is 3.6 seconds. With the current driver it takes 5.6 S. This is after some
efforts at optimisation. A time closer to 3.6 S could be achieved by writing key
methods in Assembler but I have no immediate plans to do this.

The EA display includes an LM75 temperature sensor and a 1MB flash memory chip. The
driver provides access to the current temperature. The driver does not use the flash
chip: the current image is buffered in RAM. I intend to provide an option to mount
the flash device to enable it to be used to store data such as images and fonts. This
is under development and attempts to mount it will crash the Pyboard (the ``use_flash``
Display constructor argument).

An issue with the EA module is that the Flash memory and the display module use the
SPI bus in different, incompatible ways. The driver provides a means of arbitration between
these devices discussed below.

# The driver

The driver enables the display of simple graphics and/or text in any font. The graphics
capability may readily be extended by the user. This document includes instructions for
converting Windows system fonts to a form usable by the driver (using free - as in beer - software).
Such fonts may be edited prior to conversion.

It also support the display of XBM format graphics files. Currently only files corresponding
to the exact size of the display (264 bits *176 lines) are supported.

# Connecting the display

The display is supplied with a 14 way ribbon cable. The easiest way to connect
it to the Pyboard is to cut this cable and wire it as follows. I fitted the Pyboard
with socket headers and wired the display cable to a 14 way pin header, enabling it
to be plugged in to either side of the Pyboard (the two sides, labelled X and Y on
the board, are symmetrical).

| display | signal     |  X  |  Y  | Python name   |
|:-------:|:----------:|:---:|:---:|:-------------:|
|  1      | GND        | GND | GND |               |
|  2      | 3V3        | 3V3 | 3V3 |               |
|  3      | SCK        | Y6  | X6  | (SPI bus)     |
|  4      | MOSI       | Y8  | X8  |               |
|  5      | MISO       | Y7  | X7  |               |
|  6      | SSEL       | Y5  | X5  | Pin_EPD_CS    |
|  7      | Busy       | X11 | Y11 | Pin_BUSY      |
|  8      | Border Ctl | X12 | Y12 | Pin_BORDER    |
|  9      | SCL        | X9  | Y9  | (I2C bus)     |
| 10      | SDA        | X10 | Y10 |               |
| 11      | CS Flash   | Y1  | X1  | Pin_FLASH_CS  |
| 12      | Reset      | Y2  | X2  | Pin_RESET     |
| 13      | Pwr        | Y3  | X3  | Pin_PANEL_ON  |
| 14      | Discharge  | Y4  | X4  | Pin_DISCHARGE |

E-paper 14 way 0.1inch pitch connector as viewed looking down on the pins with the keying
cutout to the left

|  1 |  2 | Red stripe on cable is pin 1
|  3 |  4 |
|  5 |  6 |
|  7 |  8 |
|  9 | 10 |
| 11 | 12 |
| 13 | 14 |

# Getting started

Assuming the device is connected on the 'Y' side simply cut and paste this at the REPL.

```python
import epaper
a = epaper.Display('Y')
a.rectangle(20, 20, 150, 150, 3)
a.show()

```python
To clear the screen and print a message (assuming we are using an SD card):

```
a.clear_screen()
with a.font('/sd/inconsolata'):
 a.puts("Large font\ntext here")
```

### Modules

To employ the driver it is only neccessary to import the epaper module and to instantiate the
``Display`` class. The driver comprises the following modules:
 1. epaper.py The user interface to the display and flash memory
 2. epd.py Low level driver for the EPD (electrophoretic display)
 3. flash.py Low level driver for the Flash memory (under development)

### Utilities

CfontToBinary.py Converts a "C" code file generated by GLC Font Creator into a binary file
usable by the driver.

### epaper.py

This is the user interface to the display, the flash memory and the temperature sensor. Display
data is buffered. The procedure for displaying text or graphics is to use the various methods
described below to write text or graphics to the buffer and then to call ``show()`` to
display the result.

The coodinate system has its origin at the top left corner of the display, with integer
X values from 0 to 263 and Y from 0 to 175 (inclusive).


## Display class

# Methods

``Display()`` The constructor has the following arguments:
 1. testing If True causes timing information to be printed. Default False.
 2. use_flash Mounts the flash drive as /fc for general use. Default False.
 3. side This must be 'X' or 'Y' depending on the side of the Pyboard in use. Default 'Y'.

The use_flash argument must currently be set False.

``clear_screen()`` Clears the screen by blanking the screen buffer and calling ``show()``
``show()`` Displays the contents of the screen buffer.
``line()`` Draw a line. Arguments X0, Y0, X1, Y1, Width, Black. Defaults: width = 1 pixel,
Black = True
``rect()`` Draw a rectangle. Arguments X0, Y0, X1, Y1, Width, Black. Defaults: width = 1 pixel,
Black = True
``fillrect()`` Draw a filled rectangle. Arguments X0, Y0, X1, Y1, Black. Defaults: Black = True
``circle()`` Draw a circle. Arguments x0, y0, r, width, black Defaults: width = 1 pixel,
Black = True. x0, y0 are the coodinates of the centre, r is the radius.
``fillcircle()`` Draw a filled circle. Arguments x0, y0, r, black. Defaults: Black = True
``load_xbm()`` Load a full screen image formatted as an XBM file. Argument sourcefile. This is
the path to the XBM file.
``locate()`` This sets the pixel location of the text cursor. Arguments x, y.
``puts()`` Write a text string to the buffer. Argument s, the string to display. This must
be called from a ``with`` block that defines the font. For example
```python```
with a.font('/sd/timesroman45x46'):
 a.puts("Large font\ntext here")
```

# Properties

``temperature`` Returns the current temperature in degrees Celcius.

## Font class

This is a Python context manager whose purpose is to define a context for the Display ``puts()``
method described above. It has no user accessible properties or methods. A font is instantiated
for the duration of outputting one or more strings. It mus be provided with the path to
a valid binary font file. See the code sample above.

In the interests of conserving scarce RAM, fonts are stored in binary files. Individual
characters are buffered in RAM as required. This contrasts with the conventional approach of
buffering the entire font in RAM, which is faster. The EPD is not a fast device and RAM is
in short supply, hence the design decision. This is transparent to the user.

### epd.py

This provides the low level interface to the EPD display module. It provides two methods
accesed by the epaper module:

``showdata()`` Displays the current text buffer
``clear_data()`` Clears the buffer without displaying it.

It also provides the ``temperature`` property.

### flash.py

This provides an interface to the flash memory. It is intended to support the block protocol
enabling the flash device to be mounted on the Pyboard filesystem. At the present time for
reasons which are unclear it doesn't work reliably and should not be used.

This module is under development and contains much unused test code. The block protocol is
implemented in a particularly dumb fashion in order to simplify the code for test purposes.
Methods other than those listed below will become protected in due course.

## File copy

A ``cp(source, dest)`` function is provided as a generic file copy routine. Arguments are
full pathnames to source and destination files.

## FlashClass

# Methods providing the block protocol (avoid at present)

For the protocol definition see
[the pyb documentation](http://docs.micropython.org/en/latest/library/pyb.html)

``readblocks()``
``writeblocks()``
``sync()``
``count()``

# Low level methods

The following low level methods work.
``available()`` Returns True if the device is detected and is supported.
``info()`` Returns manufacturer and device ID as integers.
``read()`` Arguments buf, address, count. Defaults: count = None Reads data from a byte address
to a pre-allocated bytearray. The buffer will be filled unless a byte count less than the buffer
size is provided. It will also stop if the end of a flash sector (4096 bytes) is reached. the
caller should prevent this from occurring.
``write()`` Arguments buf, address. Writes data to a byte address from a bytearray or bytes object.
The addresses to be written must have been erased (0xff). Addresses can cross block boundaries.
``sector_erase()`` Argument: address. Erase the flash sector (4096 bytes) containing the byte
address provided.

# SPI bus arbitration

This is provided for information for those wishing to modify the code. The Embedded Artists device
has a flash memory chip (Winbond MX25V8005 8Mbit chip) which shares the SPI bus with the display
device (Pervasive Displays EM027BS013). These use the bus in different incompatible ways,
consequently to use the display with the flash device mounted requires arbitration. This is
done in the Display class ``show()`` method. Firstly it disables the Flash memory's use of the bus
with ``self.flash.end()``. The EPD class is a Python context manager and its appearance in a
``with`` statement performs hardware initialisation including setting up the bus.

On completion of the ``with`` block the display hardware is shut down in an orderly fashion and
the bus de-initilased. The flash device is then re-initilased and re-mounted.

### Fonts

As explained above, the driver requires fonts to be provided in a specific binary file format. This
section describes how to produce these. Alas this needs a Windows program available
[here](http://www.mikroe.com/glcd-font-creator/) but it is free (as in beer).

To convert a system font to a usable format follow these steps.  
Start the Font Creator. Select File - New Font - Import an Existing System Font and select a font.
Accept the defaults. Assuming you have no desire to modify it click on the button "Export for GLCD".
Select the microC tab and press Save, following the usual file creation routine.

On a PC with Python 3 installed issue (to convert a file trebuchet.c to binary trebuchet)
```
$ ./CfontToBinary.py -i trebuchet.c -o trebuchet
```
The latter file can then be copied to the Pyboard and used by the driver.

This assumes Linux but CfontToBinary.py is plain Python and should run on other platforms. 

It's worth noting that though small fonts can be displayed they can be hard to read! Also fonts
can be subject to copyright restrictions. The provided 24 point [inconsolata](http://scripts.sil.org/OFL)
font is released under the the SIL Open Font License (OFL). It's a monospaced font: variable
width fonts are supported and are arguably prettier.

