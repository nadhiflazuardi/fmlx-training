## How does CAN work?
---
CAN communication uses pins called CANH (high) and CANL (low) that act as a bus where each device can connect its pins to.

Each data sent in CAN protocol involves message id and the message itself

Only one device can use the bus and send data at a time

Every node can see every message

## CAN Frame Structure
---
### SOF (Start of Frame)
- In idle state, both CANH and CANL lines are at about 2.5 V, so bus is actively "broadcasting" recessive bits
- When there is a dominant bits after at least 3 recessive bits, that must mean a node (or more) has started transmitting and the dominant bit is interpreted as the Start of Frame
### Arbitration
- When there is more than one node starting to transmit at the same time, arbitration process will be done on the identifier bits to see who wins. 
- Nodes will continue to transmit bits until there is a case where a node transmits recessive but receives dominant. In this case, the node has lost and must wait for the bus to go idle
### Control
- Metadata of the frame
- Includes 
	- identifier (IDE) if the frame is standard (0) or extended (1)
	- reserved bit for future use
	- DLC (data length code)

### Data
- 0-8 byte (classic CAN)
- Bit stuffing applies
- Can be empty for status and request purpose
- Ideally has the same size for each message ID but not strictly enforced
### CRC
### ACK
### EOF (End of Frame)
### Intermission



### Controller and Transceiver
Each device on the CAN bus (node) has controller and transceiver.
Controller:
- Message framing -> formatting the data
- Arbitration -> higher priority message gets access to the bus
- Error handling -> CRC check
- Message filtering -> Only process incoming messages with certain ID

Transceiver:
- Signal translation -> convert digital signals into voltage signals
- Bus driving -> drives CANH and CANL to create dominant and recessive state on the bus
- Receiving -> Detects voltage difference and convert to digital signal for controller
- Protection -> Protects controller from electrical disturbance


[[VCAN in Linux]]