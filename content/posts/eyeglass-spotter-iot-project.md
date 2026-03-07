---
title: "Eyeglass Spotter - An IOT Project"
date: 2020-05-27
slug: "eyeglass-spotter-iot-project"
description: "Implementing a compact Bluetooth-based device that helps find misplaced spectacles"
summary: "Implementing a compact Bluetooth-based device that helps find misplaced spectacles"
categories: ["projects"]
draft: false
---

Many times, we misplace our important items and one of such is eyeglasses. In today’s world, more than 6 people out of 10 people wear spectacles. This includes a wide range of people from children to old age ones. To overcome this problem, we came up building a device which will help such peoples to find their spectacles easily. The device named Eyeglass spotter can be mounted on spectacles as well as eyeglasses and could be operated using any mobile phone having Bluetooth module. This device can locate misplaced eyeglasses within the domestic range of 10–12 meters. The approach towards device designing, assembling and testing of this device is discussed further in this blog.

## Complete System Overview

The following figure describes the complete structure for the functioning process of Eyeglass Spotter. The operating signal to the whole system is given through the mobile app. Bluetooth is used as the wireless mode of communication for Explicit Eyeglass spotter. The mobile Bluetooth signals are received by Bluetooth module of the device and further carried to a microcontroller for processing. The processed data is transferred to a transistor for the final operation. According to the received instruction, transistor switches on or off the buzzer and L.E.D and so eyeglasses can be spotted. Bluetooth module can receive signals within the domestic range of 10–12 meters.

![Complete Flow of the System diagram](/images/system_flow.png)

---

## Hardware Design and Components

The device is designed on **Printed Circuit Board (PCB)** to make the device small in size and even lighter in weight. All the components used are **Surface Mounting Devices (SMD)** to limit device size. All the components used in building the eyeglass spotter BlueTooth module are listed below.

1. **ATmega32**: It functions as the main brain of the system. ATmega32 is used to decode the signals received through the Bluetooth module and sent instruction accordingly to a transistor. SMD is purposely used to reduce the size of microcontroller and it is programmed using AVR Studio.
2. **Bluetooth Module**: It is the main mode of communication between the mobile Bluetooth set and Eyeglass spotter device. It receives signal wirelessly through the mobile device and sends it to the microcontroller for further execution.
3. **Transistor**: It works as a switch to operate the functioning of L.E.D. and Buzzer.
4. **Battery**: A 9V battery functions as the main power source of the Eyeglass spotter device.
5. **Voltage Regulator IC**: It regulates the voltage to 5V which is operating voltage for the microcontroller.
6. **LED and Buzzer**: LED helps in spotting Eyeglasses visually whereas Buzzer helps to spot the eyeglasses by listening buzzer noise of 5KHz.

### Circuit Design

The complete circuit diagram of eyeglass spotter device is shown bellow
> Note: Original circuit contains ATmega328 smd in place of ATmega32 microcontroller in the circuit diagram

![Circuit Diagram of Eyeglass Spotter Module](/images/circuit_diagram.png)

In the circuit diagram, the power supply is given to JP1 pin. Diode D1 is used for the safety purpose of the circuit to avoid the reverse voltage. Voltage regulator IC -U1 regulates the voltage to operating voltage i.e 5v for the microcontroller. The same power supply is given to even Bluetooth module, Buzzer and LED. Transistor-T2 operates as a switch to “on” and “off” buzzer and LED. Bluetooth module receives the wireless signals through a handset. Those signals are then executed by the microcontroller and like wisely the circuit of buzzer and L.E.D. is switched on and off through transistor. Hence, whenever “A”(1) is sent through the handset, buzzer and LED is set “on” and “B”(0) is to set buzzer and LED “off”.

Following image displays the PCB design of the complete circuit, designed on Eagle Autodesk.

![PCB Design](/images/pcb_design.png) 

### Microcontroller Code

The code for the microcontroller operation is shown below. Initially, output port and baud rate is defined and also, the initial status of LED is set to OFF. Then the defined while loop keeps the microcontroller receiving signals from Bluetooth module. In while loop, `Data_in` variable stores the characters received through the handset. The further if statements operate on the basis of the character stored in `Data_in` and set or re-set the `PA0` pin which enables or disables Bluetooth module. Similarly, `USART_SendString()` is used to revert back the acknowledgement message to the handset.

```c
#include <avr/io.h>

int main(void)
{
    char Data_in;
    DDRB = 0xff;        /* make PORT as output port*/
    USART_Init(9600);   /* initialize ISART with 9600 baud rate */
    LED = 0;
    while(1) {
        Data_in = USART_RxChar();   /* recieve data from Bluetooth device */
        if(Data_in == 'A') {
            LED |= (1<<PA0);    /* Turn ON LED and buzzer */
            USART_SendString("A");    /* revert back status of LED and  buzzer to handset */
        }
        else if(Data_in == 'B') {
            LED &= ~(1<<PA0);   /* Turn OFF LED and buzzer */
            USART_SendString("B");   /* revert back status of LED and buzzer to handset */
        }
        else
            USART_SendString(Data_in);  /* send message for selecting proper option */    
    }
}
```
---

## Fundamental Mobile Application

![Front-end of Application](/images/app_frontend.png)

The initial version of the mobile application was built on the MIT App Inventor. The application holds a button to scan the Eyeglass Spotter device for establishing an initial connection. Also, it has two basic buttons to turn ON and OFF the LED and buzzer mounted on eyeglass spotter device. Furthermore, it includes text in big font displaying the status of the eyeglass spotter device.

The backend of the application is designed using placeholders of MIT App Inventor. The first place holder functions for “SCAN” button using a `ListPicker` element for pairing the eyeglass spotter device with the handset. Furthermore, the second placeholder establishes the connection of the handset with the device Bluetooth module. The last two placeholders are used to sent the character text “A” or “B” to switch the LED and buzzer ON and OFF

![Back-end of an Application](/images/app_backend.png)


---

Here is a video demonstrating the working of the whole system
P.S: Unlike the battery used in the video, the use of Li-Po battery is recommended as it is small-sized and weighs less.

{{< youtube -QApMnUt0Uo >}}

---

## References

1. [ATmega32 Datasheet](https://www.microchip.com/en-us/product/ATmega32)
2. [HC-05 Bluetooth Module Documentation](https://components101.com/wireless/hc-05-bluetooth-module)
3. [MIT App Inventor Library](https://appinventor.mit.edu/explore/library)