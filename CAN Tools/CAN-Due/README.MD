
I use the [GVRET](https://github.com/collin80/GVRET) software on an Arduino Due extensively with SavvyCan. 

I started with the [Open Inverter Wiki](https://openinverter.org/wiki/CAN_bus_with_Arduino_Due) article on CAN bus with Arduino Due.

The CAN transceivers I use contain a chip marked VP230 TI41M oAO7R from [Aliexpress](https://www.aliexpress.com/item/1005002843325046.html). The vendor description is "SN65HVD230 VP230 CAN Board Network Transceiver Evaluation Development Module For Arduino Controller Board DC 3V-3.6V" I have 5 of these in service at the moment and have had no issues at all.

While looking to create a bridge function by modifying the GVRET software I discovered there are lots more functions available than just RX/TX via SavvyCan. I can find no documentation on the web so have used OpenCode to create these User Guides. the .MD file is the master and the HTML has been created from that.

###Notes\
  The user guide is created from the Master branch of the software, NOT the release. The Release version does not have the bridge function.\
  The master will only compile correctly if the exact libraries specified and downloaded from the links in the readme.MD.\ 
  The names of the downloaded libraries must be changed to remove "master". Check in the software directly to check library names called if you have issues. I had problems because I'd downloaded an update to the CAN library.\
  The actual version of GVRET I'm running appears to be slightly different to the Master branch, although it has the same build number 343. On my hardware I need to earth the pins for bridging to take place, while OpenCode is adamant that the logic in the Master branch is that Open will cause bridging. \
