# CAN Bus
This is where I collect work on the can bus

A massive thanks to Project Gus for all the work he's been doing on a Kona.  
[Project Gus Blog](https://www.projectgus.com/tag/ev-conversion-project.html)


"P-CAN message chasing 11-01-25.txt" is information captured from the P-CAN bus Teed in on the connection between the front loom and the main loom.
The intention was to verify the source of a number of IDs that are currently listed in ProjectGus DBC file as unknown.


The ScanDoc folder contains D-CAN scans when running various scans from the ScanDoc tool.

B-CAN contains logs etc associated with the B-CAN. It started with investigation the BCM then includes identifying messages from the SKM and IGPM on the B-CAN
Note I have identified that frame 0x109 originates from the BCM on B-CAN and is very different to the 0x109 that appears on the P-CAN.
