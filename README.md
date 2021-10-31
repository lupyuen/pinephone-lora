# Experimenting with LoRa Backplate for PinePhone

Follow the updates on Twitter: https://twitter.com/MisterTechBlog/status/1454200499247349760

Schematic and Firmware: https://wiki.pine64.org/wiki/Pinedio#Pinephone_backplate

PinePhone I2C Port: https://wiki.pine64.org/index.php/PinePhone#Pogo_pins

# I2C-To-SPI Bridge

From https://github.com/zschroeder6212/tiny-i2c-spi/blob/master/src/main.c

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

When read, the bridge returns the bytes received from SPI:

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

# Probing LoRa Backplate with Bus Pirate

Based on http://dangerousprototypes.com/docs/I2C

```text
m
I2C > HW > 400
W
(1)
```

Enable pullup:

```text
P
```

Read Address 0x28 Register 0x01:

```text
[0x50 0x01 0x00] [0x51 r]
```

Read Address 0x00 Register 0x01:

```text
[0x00 0x01 0x00] [0x01 r]
```

Read Address 0x01:
```text
[0x01 r]
```

Read Address 0x28:

```text
[0x51 r]
```

