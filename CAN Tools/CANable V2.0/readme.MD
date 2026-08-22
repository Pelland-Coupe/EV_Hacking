# Warning there are some faulty boards out there.
If it doesn't work for you and you're new to CAN it's likely to be the board not you.


For my first experiments with CAN bus and Savvycan I purchases an CANable V2.0 from a seller on Aliexpress.\ 
This proved to be a mistake. It would not work no mater what I tried.\
I did software upgrades, changed from Slcan to CandleLight and back again. Each time it would load and verify correctly but no sent or receive CAN frames.\
Checking with a 'scope showed no TX on the CAN bus.

I abandoned this for a while and tried CANDue which got me lots of success.

After gaining experience with Savvycan & CANDue I looked at the CANable board again and finally found the issue.\
The board was populated with the wrong CAN driver chip. It has a CA-IS3050G where as the PCB is laid out for the CA-IS3052G.\
On the 50G TX is pin 3 and RX is pin 2. On the 52G TX is pin 2 and RX is pin 3.

In my case rather than changing the chip I lifted the pins and air wired them to the correct place.

The board now works flawlessly with CandleLight and creates a native CAN interface under Linux.
