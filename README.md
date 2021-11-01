# Experimenting with LoRa Backplate for PinePhone

![LoRa Backplate for PinePhone](https://lupyuen.github.io/images/pinephone-lora.jpg)

Follow the updates on Twitter: https://twitter.com/MisterTechBlog/status/1454200499247349760

LoRa Backplate for PinePhone consists of...

-   Semtech SX1262 LoRa Transceiver (SPI) and antenna

    https://www.semtech.com/products/wireless-rf/lora-core/sx1262

-   I2C Connector that matches the I2C Port (Pogo Pins) on PinePhone

    https://wiki.pine64.org/index.php/PinePhone#Pogo_pins

-   I2C-To-SPI Bridge (ATtiny84) that connects the I2C Connector to the LoRa Transceiver (SPI)

    https://github.com/zschroeder6212/tiny-i2c-spi

Schematic: https://wiki.pine64.org/wiki/Pinedio#Pinephone_backplate

Let's probe the LoRa Backplate with a Bus Pirate, to transmit some SX1262 commands and read the responses: http://dangerousprototypes.com/docs/I2C

# I2C-To-SPI Bridge

Arduino Source Code: https://github.com/zschroeder6212/tiny-i2c-spi/blob/master/src/main.c

The bridge runs at I2C Address 0x28:

```c
#include "USI_TWI_Slave.h"
#define I2C_ADDR 0x28

int main (void)
{   
    /* init I2C */
    usiTwiSlaveInit(I2C_ADDR);

    /* set received/requested callbacks */
    usi_onReceiverPtr = I2C_received;
    usi_onRequestPtr = I2C_requested;
```

The bridge transmits data to SPI when the first byte received is 0x01:

```c
#define CMD_TRANSMIT    1
#define CMD_CONFIGURE   2

void I2C_received(uint8_t bytes_recieved)
{
    uint8_t command = usiTwiReceiveByte();

    switch(command)
    {
        case CMD_TRANSMIT:
        {
            CS_PORT &= ~(1<<CS);

            for(int i = 1; i < bytes_recieved; i++)
            {
                uint8_t received_byte = SPI_transfer(usiTwiReceiveByte());
                SPI_buffer[received_index++] = received_byte;
```

When the bridge receives a read command, it returns the bytes received from SPI:

```c
void I2C_requested()
{
    usiTwiTransmitByte(SPI_buffer[transmit_index++]);
    if(transmit_index >= BUFFER_SIZE)
    {
        transmit_index = 0;
    }
}
```

`usi` functions are defined here: https://github.com/zschroeder6212/tiny-i2c-spi/blob/master/src/USI_TWI_Slave.c

# Probe LoRa Backplate with Bus Pirate

Based on:

-   http://dangerousprototypes.com/docs/I2C

-   https://lupyuen.github.io/articles/i2c#appendix-test-bme280-with-bus-pirate

Connect Bus Pirate to LoRa Backplate:

| Bus Pirate Pin | Backplate Pin
|:---:|:---:
| __`MOSI`__ | `SDA`
| __`CLK`__ | `SCL`
| __`5V`__ | `USB-5V`
| __`GND`__ | `GND`

Enter I2C mode, power on:

```text
m
I2C > HW > 400
W
```

Scan I2C bus:

```text
(1)
```

I2C scan results:

```text
Searching I2C address space. Found devices at:
0x00 (0x00 W) 0x01 (0x00
```

__Problem: Bus Pirate hangs while scanning. And I2C Address of LoRa Backplate should be 0x28, not 0x00. Why?__

Read Address 0x28:

```text
[0x51 r]
```

Returns:

```text
I2C START BIT
WRITE: 0x51 ACK 
READ: 0x00 
NACK
```

__Problem: Bus Pirate hangs while reading from LoRa Backplate. Why?__

TODO: Write and Read Address 0x28 Register 0x01:

```text
[0x50 0x01 0x00] [0x51 r]
```

TODO: Read Address 0x00?
```text
[0x01 r]
```

TODO: Write and Read Address 0x00 Register 0x01?

```text
[0x00 0x01 0x00] [0x01 r]
```

TODO: Enable pullup?

```text
P
```

![Probing LoRa Backplate with Bus Pirate](https://lupyuen.github.io/images/pinephone-probe.jpg)

# Test LoRa Backplate on PinePhone

On PinePhone with Manjaro Phosh, scanning the I2C Bus with `i2cdetect`:

```bash
[manjaro@manjaro-arm ~]$ i2cdetect -l
i2c-0	unknown   	DesignWare HDMI                 	N/A
i2c-1	unknown   	mv64xxx_i2c adapter             	N/A
i2c-2	unknown   	mv64xxx_i2c adapter             	N/A
i2c-3	unknown   	mv64xxx_i2c adapter             	N/A
i2c-4	unknown   	i2c-csi                         	N/A
i2c-5	unknown   	i2c-2-mux (chan_id 0)           	N/A

[manjaro@manjaro-arm ~]$ sudo i2cdetect 3
WARNING! This program can confuse your I2C bus, cause data loss and worse!
I will probe file /dev/i2c-3.
I will probe address range 0x08-0x77.
Continue? [Y/n]
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:                         -- -- -- -- -- -- -- --
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
70: -- -- -- -- -- -- -- --
```

__Problem: `i2cdetect` fails to detect the I2C address of the LoRa Backplate. Why?__

Has the LoRa Backplate been flashed with the right firmware?

I2C-To-SPI Bridge on ATtiny84: https://github.com/zschroeder6212/tiny-i2c-spi
