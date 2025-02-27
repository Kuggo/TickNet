# TickNet V1.0:

**Ticknet** is a wireless communication protocol for CPUs in Minecraft. Here you can find all the schematics for all Network Interfaces/Nodes and their respective specifications. 

TODO explain the organization of the repository. where we can find what and stuff
To contribute link to section of community contributions

Continue reading below for a complete guide on Ticknet!

---
## Index

### 1. Introduction
- **1.1 What is Ticknet?** (Short description of the protocol)
- **1.2 Why Ticknet?** (Problems it solves, motivation for its design)
### 2. How Ticknet works?
- **2.1 Relationship to TickLink Technology** (Where Ticknet begins)
- **2.2 Channels and Addressing** (How nodes send and receive messages)
- **2.3 Collision Prevention** (How to avoid collisions)
- **2.4 Reliability and Delivery Model** (What guarantees exist and what can fail)
### 3. Ticknet Protocol Specification
- **3.1 Packet Structure**
    - 3.1.1 Header Format
    - 3.1.2 Payload
- **3.2 Addressing Scheme**
    - 3.2.1 Static Addressing
    - 3.2.2 Dynamic Addressing
- **3.3 Compatibility & Future proofing**
	- 3.3.1 Versioning
	- 3.3.2 Channel addresses
	- 3.3.3 Alternative protocol compatibility
### 4. Ticknet Interfacing
- **4.1 Commands available**
- **4.2 Example usage**
	- 4.2.1 Setup/terminate session
	- 4.2.2 Sending a transmission
	- 4.2.3 Receiving a transmission
### 5. Extensions & Adapters
- **5.1 Word Sizes**
- **5.2 Dynamic addresses**
- **5.3 No PutByte/GetByte needed**
- **5.4 Byte stream transmissions**
### 6. Usage Guide
- **6.1 Setting Up a Ticknet Node**
    - 6.1.1 Required Components
    - 6.1.2 Address Configuration
- **6.2 Testing if it works**
### Build explanation?
### 7. Future Upgrades and Roadmap
- **7.1 Potential Features for Future Versions**
	- 7.1.1 Secure Networks
	- 7.1.2 Direct connections
- **7.2 Community Contributions and Modifications**
### 8. Glossary & References
- **8.1 Glossary of Terms**
- **8.2 References and External Links**

---
## 1. Introduction

### What is Ticknet?
**Ticknet** is a wireless communication protocol for CPUs in Minecraft, designed to provide instant, reliable messaging across an entire dimension without requiring Redstone wiring or chunk loading. Built entirely using vanilla mechanics, Ticknet enables direct communication between nodes without hierarchy, ensuring that any node can send and receive messages efficiently in a cooperative network environment.

### Why Ticknet?
Traditional non instant Redstone wiring introduces a 2-game tick delay every 18 blocks of distance, making long-range communication slow and impractical for CPU-based systems. Ticknet eliminates this limitation by using [**TickLink**](#ticklink) technology, a mechanism that leverages how Minecraft processes game logic to allow instantaneous communication anywhere in the same dimension. Since it operates entirely within the game’s existing mechanics, it does not require mods, plugins, or external tools.

Ticknet’s flat network topology means that every node operates independently without requiring hierarchical structures, relays, or leader elections. Nodes can communicate directly without intermediaries, ensuring a **fast and scalable** system. Communication supported is unicast and broadcast. Multicast is not directly supported in this version, but potentially in the future it can.

Ticknet follows a **best-effort** delivery model, similar to UDP, where packets are sent to listening nodes without built-in acknowledgments. However, unlike traditional unreliable protocols, it ensures no packet duplication/reordering and **no packet loss** under normal conditions.

This protocol’s design and specifications allows for extensibility, future proofing, as well as the existence of competing standards that will want to use the same base technology and the same channel range.

## 2. How Ticknet Works

### 2.1 Relationship to TickLink Technology

Ticklink was a technology developed that allowed data to be sent and received from anywhere in the same dimension in minecraft. It has currently some very powerful features, namely:
- Constant delay transmissions
- Wireless medium
- Channel isolation
- Collision prevention
In order to efficiently be integrated with Ticknet more hardware was developed and the last 2 features on the above list were added. All of these properties when combined with the protocols and specifications of Ticknet allow for a system that spans the first 4 layers of the OSI model.

### 2.2 Channels and Addressing

Packets are sent over **channels**. Channels with different address are independent and don't interfere with each other. There is an infinite range of channels that can be used. In fact in the most pure way a channel is only 1 bit, but 8 subchannels that share the same starting bits as the main channel are used to send data in parallel.

All transmissions are done byte by byte, 1 byte every **transmission cycle**. Transmission cycles are a the time instant (game tick) in which all transmitters and receivers send/detect the channel's state. These happen continuously with a 1 second interval between cycles.

Each node is identified with a **static address** and that value is used to identify which channel belongs to which node. 
Sending a packet to a specific node is as simple as broadcasting on its assigned channel. 

It is important to mention that channels can be eavesdropped by any other node, but in a cooperative environment as this was designed for, that is not a problem. We also use this same property to allow broadcasting by assigning channel 0 to be the broadcast channel. Multicast is not directly supported but an extension that assigns channels to groups can be implemented in the future at little cost.

### 2.3 Collision Prevention

When multiple nodes transmit at the same time in a channel, nodes who will be receiving that channel will experience what is called a collision. Using TickLink hardware, a collision will cause the resulting byte on the channel to be the equivalent of xoring all bytes from all sources.
They are undesirable because they corrupt transmissions, but there are 2 ways to negate them:
- **Collision Detection:** When a collision is detected, the packet is invalidated and the transmissions are repeated later to ensure no loss of data.
- **Collision Prevention:** If collisions don't happen in the first place, then no data is lost (assuming a cooperative environment)

The best option is to prevent collisions to maximize how much useful time there is for transmissions and that is what Ticknet does. 
To achieve there is a special purpose bit named ping bit that controls which node is entitled to transmit. Whenever a node wants to transmit to a channel they start pinging that bit, and the same hardware will provide 1 to the node that is allowed to transmit and 0 to those that must wait.
Internally there is a FIFO queue that works in modulus 16 and keeps track of the current head. If there are more than 16 people wanted to transmit in the same channel 2 people would get assigned permission and a collision would happen. Due to the low likelihood of that event, nothing in the current version is done to avoid it and that is another reason why the protocol is not perfect but **best-effort**. In future the limit of 16 can be raised (at the cost of bigger hardware) to allow for more traffic intense networks.

This system is designed for a cooperative environment as the head could never let go of the priority to transmit stopping others from ever getting access. So after a transmission they stop pinging and if potentially they want to transmit again they go to the end of the queue.

### 2.4 Reliability and Delivery Model

Ticknet operates as a **connectionless** communication protocol, meaning that nodes do not establish persistent connections but instead transmit messages to any listening recipient on a given **channel**. This makes it functionally similar to **UDP**, where messages are sent without requiring acknowledgment or retransmission. However, unlike UDP, Ticknet guarantees that all successfully transmitted messages arrive in order and without duplication. It was mentioned that it was **best-effort** because in some scenarios the messages may be lost:
- **The receiver is not online or actively listening.** If a node is offline or not tuned to the channel at the time of transmission, the message will not be received.
- **The receiver lacks memory to store the message.** Each node has a finite buffer for incoming messages. If this buffer is full, new messages cannot be stored, resulting in data loss.



## 3. Ticknet Protocol Specification
### 3.1 Packet Structure

A packet is a data structure containing information about a message that is being sent on a channel. It contains a header which contains information needed to decode the rest of the packet and the payload which is the useful data that the network is tasked to transmit. 
There are no checksums as it is traditional because no packet interference is expected.
#### 3.1.1 Header
The header contains 3 fields each taking up 1 byte:
- **Magic Number:** A non zero number that starts the packet so that receivers can know when a transmission is happening. If another protocol that uses the same address range (or even Ticknet on future versions) with a different packet structure can show that by picking a different magic number. If an unknown magic number is detected then the packet is ignored. The value for Ticknet is **69**.
- **Packet Length:** The total length of the packet, which includes header length and payload length. This value must be be within **\[2, 26\]**. The length right after the magic number is important for nodes to know how long to ignore/capture the data.
- **Sender Address:** The address of the node sending the message.
Destination address is not on the packet because it is implicitly sent on the channel (it is the channel address of where it was sent)
#### 3.1.2 Payload
The bytes immediately after the header are considered payload and there are as much of them as stated in the header. 
Messages bigger than the maximum allowed must be broken down into multiple packets as Ticknet does not natively support it.

### 3.2 Addressing scheme
#### 3.2.1 Static addressing
Current version of Ticknet uses static address to identify a network node. Those addresses are 8-bit wide and there are 254 of them available (range of \[1, 254\]). Address 0 is reserved for broadcast and address 255 is reserved for the event of lack of address space.
Addresses must be unique and constant during the uptime of a node. As such, management of taken/available addresses is important to consider. Ticknet, says nothing regarding this, and it is subject to the individual servers and communities to manage them as they wish.
#### 3.2.2 Dynamic addressing
Dynamic addressing is not directly supported in this version of Ticknet, but there is nothing stopping the use of an adapter that provides a session long unique 8-bit address that the node will use as its "static address".
Look in (TODO) for more information about Extensions/adapters or here TODO for future plans

### 3.3 Compatibility & Future proofing
#### 3.3.1 Versioning
Ticknet strives for future and backwards compatibility between minor versions, but in the case of big changes that will restructure packets a new magic number must be used to indicate that the format following it is a different one. This means only Nodes in the same versions will be able to communicate as unsupported version messages will be ignored. 
Magic numbers must be nonzero and number 255 is reserved for the event of lack of numbers. If 255 is used, then the following byte is used to distinguish between protocols/versions. This is not strictly enforced due to the lack of demand and is more of a guideline to follow.
When the time comes, universal commands to ping nodes and ask for information such as versions supported will be implemented, but for now the lack of versions does not justify implementing it.
#### 3.3.2 Channel addresses
 For future proofing, if every address is taken, then address 255 is used and the next byte after that can be used as a new extended address. This is not enforced in the current version.
#### 3.3.3 Alternative protocol compatibility
The above restrictions are for protocols that wish to use the same range of channel addresses as Ticknet, but there are alternative ways with less restrictions to not interfere with Ticknet.
Prefixing channel numbers with a magic number would be the desired way as that adds little hardware cost, keeps the same speed and properties, and allows independent network space with zero collisions or shared network cycles.
Alternatively, the offset in network clocks, could also work. Transmission cycles are 1hz and there are 19 other potencial game ticks that Ticknet will not be using the channels for that can be used by other protocols instead. The cost of this approach is also minimal as long as every node follows that. 
Faster protocols that have more frequent cycles for more dense data transmission are also viable as long as they don't share cycles with Ticknet, or have a different magic number

## 4. Ticknet Interfacing
### 4.1 Commands available
The internal hardware is hidden behind a simple interface that CPUs must use. This means no shared resources are needed and the hardware will contain its own memory/buffers and will manage everything autonomously. 

That interface has 2 I/O ports. The first one is the command port and is used to input command codes and receive feedback from the operations being executed. The second one is the data port and is used to input data to send and receive data transmitted.
Before sending a command to execute on the command port, the arguments (if applicable) must be passed beforehand on the data port. When an output port is updated in response to a command, a signal is generated (for interrupt/acknowledge based systems).

Below is a table containing all the commands, the arguments, and further down a brief description of what the command does and how.

TODO: update table for possible errors, explain the errors and feedback of the command port
 and the bitmask each does

| Command Name | Command Code | Input Data | Output Data    |
| ------------ | ------------ | ---------- | -------------- |
| Setup        | 0            | -          | -              |
| Terminate    | 1            | -          | -              |
| PutByte      | 2            | DataByte   | -              |
| Send         | 3            | Address    | -              |
| GetByte      | 4            | -          | Source \| Data |
| NextPacket   | 5            | -          | -              |
| GetSize      | 6            | -          | PacketSize     |
|              |              |            |                |

**Setup**: starts the NIF and gets it online (internal clocks align, channel's ping bit starts along with random noise, starts receiving from global/own channel)

**Terminate**: stops the NIF and makes it offline (internal clocks align, channel's ping bit stops along with the random noise, stops receiving transmissions from global/own channel, and memory reset)

**PutByte**: Takes a data byte and writes it to the transmission buffer to later be send to the NIF whose address the user NIF is currently bound to. Excess data sent beyond the limit will not be stored and sent.

**Send**: Flushes the transmission buffer and sends the store data to the specified channel on Input Data. Metadata and sender address are included int the transmission by default.

**GetByte**: Reads the next byte from the current received transmission. If a transmission is fully read, a NextPacket command needs to be issued for meaningful data to be read, otherwise it will return 0s. 
The first byte of a transmission will be the length, and second will be the sender address. After that, payload data will follow byte by byte.

**NextPacket**: Discards the remaining bytes unread from the current packet and prepares automatically the next one. If there is no next packet, operation will fail (Every call to GetByte will be 0s). When all packet bytes are read, this command must be called to proceed to next one.

**GetSize**: Returns how many bytes are left to read from current packet.
### 4.2 Example usage
There is no human code for interacting with the interface, but the following examples are written in the URCL assembly language, providing a low level view of how CPUs might interact with the network interface to perform some basic operations.
We will assume that Command port of the TN interface will be named `TN_cmd` and Data port will be named `TN_data`.
Additionally to make it easier, we will directly use macros to map command names to their command codes:
```
// Mapping commands to their command codes
@DEFINE SETUP 0
@DEFINE TERMINATE 1
@DEFINE PUT_BYTE 2
@DEFINE SEND 3
@DEFINE GET_BYTE 4
@DEFINE NEXT_PACKET 5
@DEFINE GET_SIZE 6

// Eg: Using @SETUP will replace it with 0
```

#### 4.2.1 Setup/terminate session
Here is how you can setup your node to listen:
(operations that might fail are being ignored in favor of clean code)
```
OUT %TN_cmd @SETUP
IN R0 %TN_cmd       // waiting for command result and discarding it
// interface is now online and listenning to incoming transmissions

//After some time we turn it off
OUT %TN_cmd @TERMINATE
IN R0 %TN_cmd       // waiting for command result and discarding it
```
#### 4.2.2 Sending a transmission
Broadcasting "Hello" to all members of the network:
```
OUT %TN_cmd @SETUP
IN R0 %TN_cmd       // waiting for command result and discarding it
// interface is now online and listenning to incoming transmissions

OUT %TN_data 'H'       // sending 'H' to the transmission buffer
OUT %TN_cmd @PUT_BYTE  // puting that byte
IN R0 %TN_cmd          // waiting for command result and discarding it

OUT %TN_data 'e'
OUT %TN_cmd @PUT_BYTE
IN R0 %TN_cmd
OUT %TN_data 'l'
OUT %TN_cmd @PUT_BYTE
IN R0 %TN_cmd
OUT %TN_data 'l'
OUT %TN_cmd @PUT_BYTE
IN R0 %TN_cmd
OUT %TN_data 'o'
OUT %TN_cmd @PUT_BYTE
IN R0 %TN_cmd

OUT %TN_data 0
OUT %TN_cmd @SEND
IN R0 %TN_cmd

//After some time we turn it off
OUT %TN_cmd @TERMINATE
IN R0 %TN_cmd       // waiting for command result and discarding it
```
TODO make this with loop and DW
#### 4.2.3 Receiving a transmission
Any transmission (sent directly or broadcasted) is received in the same way with no distinction. This program will save the sender of the transmission and print the message character by character in a loop.
```
OUT %TN_cmd @SETUP
IN R0 %TN_cmd       // waiting for command result and discarding it
// interface is now online and listenning to incoming transmissions

.busy_waiting
OUT %TN_cmd @NEXT_PACKET  // try to get the next packet
IN R1 %TN_cmd             // save result of operation in R1
BNZ .busy_waiting R1      // if error code != 0 try again

OUT %TN_cmd @GET_SIZE
IN R0 %TN_cmd  // waiting for operation
IN R1 %TN_data // length in R1

OUT %TN_cmd @GET_SIZE
IN R0 %TN_cmd  // waiting for operation
IN R2 %TN_data // sender in R2

.loop
OUT %TN_cmd @GET_BYTE // perform getbyte operation
IN R0 %TN_cmd         // waiting for operation
IN R2 %TN_data        // fetch the byte
OUT %text R2          // print character
DEC R1 R1             // decrement the length of remaining message
BRP .loop R1          // if length > 0 go back

//After some time we turn it off
OUT %TN_cmd @TERMINATE
IN R0 %TN_cmd       // waiting for command result and discarding it
```


---

