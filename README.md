openavrusb-library
==================

This C# library provides a transparent interface to a micrcontroller running firmware from the openavrusb-firmware repository.

The firmware implements a USB HID device, allowing driverless operation under windows. See the examples for more info.

## GPIO Example

This example shows how to use the simple and powerful functions provided by the interface library, to control devices with general purpose digital input/output pins (GPIO).

The GPIO block includes three main functions; set port direction, write port and read port. The 20 available GPIO pins are arranged into three ports denoted by letters. Each of the pins in these ports is individually addressable by its number. (eg PC3, PB7, etc)

### Step 1: Set the pin directions

Each individual pin can be set as either input or output. Input pins can be configured as tri-state (high impedance floating input) or with a weak internal pullup resistor. When any pins are written or direction-set, a mask byte is used to ensure only the correct pins are affected. In this example we will set the upper four pins of port D to output, and the lower four to input. The output pins will be in the high state, and the inputs will have pullups enabled.

```
//Create the USB interface object first of all
//No params are required if only one HU320 is present
HU320 usbI = new HU320();
 
//0xF0 is used to mask the upper bits when setting output pins
//Writing 1's to the direction register sets the pins as outputs
usbI.GPIO_setDirPort('D', 0xFF, 0xF0);
usbI.GPIO_writePort('D', 0xFF, 0xF0); //set upper pins high
 
//0x0F is used to mask the lower bits when setting input pins
//Writing 0's to the direction register sets the pins as inputs
usbI.GPIO_setDirPort('D', 0x00, 0x0F);
//writing 1's to pins defined as input enables the pullup resistors
usbI.GPIO_writePort('D', 0x0F, 0x0F);
```

### Step 2: Perform a read operation

The read operation simply gets the current value of the port register. For output pins, the current state can still be read back with this function.

```
//create a byte variable for the data to be stored
byte val;
 
//read the value of the port and store it in 'val'
usbI.GPIO_readPort('D', out val);
```

## LCD Example

The interface library defines three functions for LCD connectivity; initialise, clear and write. The write method is overloaded for to provide the following functions:


* Write a string directly to the LCD, starting from the top line (line 0) on the left hand side (column 0). When using this overload, a string the full length of the LCD can be used to fully populate values in one step.
* Write a byte array to the LCD, starting from the top line (line 0) on the left hand side (column 0). Using a byte array allows character sets other than ASCII. For example, the HD44780 controller also supports Japanese character fonts which can be used in this way.
* Write the byte array to a specific location. The line number (0 or 1) can be defined, as well as the column number. 20x4 LCDs are programmed as 40x2, as specified by the HD44780 controller documentation. The first and last line pairs are written as one line.
* Write a string to a specific location, as per the byte array above.

```
//Create the USB interface object first of all
//No params are required if only one HU320 is present
HU320 usbI = new HU320();
 
//First initialise the LCD
usbI.LCD_init();
 
//write a string directly to the LCD at location 0,0.
usbI.LCD_write("write a string to the LCD");
 
//Write data as a byte array to the LCD screen
byte[] data = { 65, 66, 67, 68, 69 };
usbI.LCD_clear();
usbI.LCD_write(data);
 
//Write the byte array to a specific location, line 0, column 4
usbI.LCD_clear();
usbI.LCD_write(data, HU320.LCD_line.line0, 4);
 
//Write a string to a specific location, line 1, column 6
usbI.LCD_clear();
usbI.LCD_write("string to write", HU320.LCD_line.line1, 6);
 
//close the handle to the device
usbI.close();
```

## I2C Example

With the standard binary programmed into the AVR, the following calls can be made using the .NET component to affect an interface to the popular 24C32 series of I2C eeprom chips. The I2C block in the .NET library contains a handful of very powerful functions which provide a flexible I2C interface. For the majority of I2C applications, including this one, only three commands (setup, read, write) are required for a fully functioning interface.

### Step 1: Setting up the I2C interface in the hardware

The I2C interface lines SDA (bi-directional data) and SCL (clock) are implemented on pins PB5 and PB6 respectively. In order to meet the requirements of I2C, the bus is created in a wired-or configuration. This means that the lines are normally held high with a single pullup resistor at the master node. (These pullups are included on the Helion Microsystems HU320 breakout PCB). Either the master or slaves can pull the lines low with an open-collector switch.

To put the I2C pins into the correct states for communication, the I2C initialisation routine must be called, this couldn't be simpler:

```
//Create the USB interface object first of all 
//No params are required if only one device is present 
HU320 usbI = new HU320();   //initialises the I2C interface 
usbI.I2C_setup(); 
```

### Step 2: Data and addresses to send to the device

Each I2C device must have a unique address on the bus. For many devices (such as 24C32) four bits of the seven bit address are fixed internally, and the lower three remaining bits are pin selectable. In this example the full 7 bit address is 0x53 (or in binary 1010011).

When writing to an I2C device, first the physical device is selected using the address in the previous paragraph, then the internal memory location to write to is selected (sometimes this will be referred to as the 'instruction'). This memory location field can be 8 or 16bits, the 24C32 has a 16 bit address. The .NET library includes overloads of the read and write instructions to cope with both of these options as well as an arbitrary function to accomodate any device.

If the address is unknown, a bus scan can be performed. Essentially every device address is polled, and those that are acknowledged by a slave are recorded and returned as valid addresses.

```
//run a bus scan to determine the addresses of connected devices
byte[] validAddresses;
usbI.I2C_busScan(out validAddresses);
 
//define the addresses and data to write
byte deviceAddress = 0x53;
ushort memoryAddress = 0x0000;
byte[] data = { 0, 1, 2, 3, 4, 5, 6 };
 
//Perform all I2C write functions in a single line!
usbI.I2C_write(deviceAddress, memoryAddress, data);
```

### Step 3: Read the data back from the device

Reading data back from the device is just as simple as the write function, just supply the HU320 library with the device address and memory location, and let it do the work for you.

```
//define an array to catch the returned data
byte[] outData;
 
//define how many bytes to attempt to read
int numBytesToRead = 7;     //same as were written
 
//Perform the I2C read functions
usbI.I2C_read(deviceAddress, address, numBytesToRead, out outData);
```


### Step 4: Check for errors

Errors?! What? We dont get errors!

However, it is worth checking to ensure that the communications have not been compromised by some outside factors, such as; noise on the lines, unstable power supplies, gremlins chewing on the wires etc.

The errors which will be detected are;

* Missing acknowledge - the slave did not understand the message
* Arbitration lost - another master on the bus attempted communcation at the same time
* Setup failure - the I2C lines could not be set into their initial states, a slave is holding a line low

```
//The errors are reported as an enumerated type
HU320.I2C_error error = usbI.I2C_checkError();
```

### Step 5: Close the device connection

Calling the close method disposes of all handles and connections to the USB device. It is good practice to do this when our program is complete.

```
//dispose of the connection before quitting
usbI.close();
```

