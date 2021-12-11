---
attachments: [Clipboard_2021-11-26-12-48-36.png, Clipboard_2021-11-28-17-06-15.png, Clipboard_2021-11-28-17-06-57.png, Clipboard_2021-12-07-13-05-47.png, Clipboard_2021-12-07-13-06-16.png, Clipboard_2021-12-07-13-06-47.png, Clipboard_2021-12-07-13-07-26.png, Clipboard_2021-12-07-13-36-27.png, Clipboard_2021-12-10-15-09-48.png, Clipboard_2021-12-10-15-13-17.png, Clipboard_2021-12-11-10-21-39.png, Clipboard_2021-12-11-10-51-43.png, Clipboard_2021-12-11-11-11-56.png, Clipboard_2021-12-11-12-11-57.png]
tags: [Notebooks, Notebooks/AvansertNettverk]
title: Network Device Internals
created: '2021-11-26T11:08:56.453Z'
modified: '2021-12-11T11:13:55.036Z'
---

# Network Device Internals

A generic router architecture can be shown in the diagram below. 

![](@attachment/Clipboard_2021-11-26-12-48-36.png)

It is divided into a *routing, management control plane* (software) and a *forwarding data plane* (hardware).

- **Input port**: (1) Terminates an incoming physical link at a router. (2) It also performs link-layer functions needed to interoperate with the link layer at the other side of the incoming link. (3) A lookup function is also performed. It is here that the forwarding tabel is consulted to determine the router output port to detrmine the router output port to which an arriving packet will be forwarded via the swithching fabric.

  -  The three functions are represented by the three boxes in the input port diagram above.

  - Control packets are forwareded from an input port to the routing processor.

- **Switching fabric**: The swithcing fabric connects the router's input ports to its output ports.

- **Output ports**: Stores packets received from the switching fabric and transmits these packets on the outgoing link by performing the necessary link-layer and physical-layer functions.

- **Routing processor**: Performs control-plane functions. It executes the routing protocols, maintains routing tables and attached link state information, and computes the forwarding table for the router. SDN routers use the routing processor for communicating with the remote controller in order to receive forwarding table entries computed by the remote controller. It also performs the network management functions.

The dataplane operates at the nanosecond time scale. The route's control functions - executing the routing protocols, responding to attached links that go up or down, communication with the remote controller and performing management functions - operate at the millisecond or second timescale. The **control plane** functions are thus usually implemented in software and executed on the routing processor.

**Destination-based forwarding**: packet indicates its final destination. In the input port, the ruter looks up the final destination and determines the output.

**Generalized forwarding**: inspect other things about the packet and not only the destination. This may include packet origin and priority. Any number of factors may contribute to the chosen output port.

### Input port processing and destination-based forwarding

If our router has four links and the packets are to be forewarded as follows:

![](@attachment/Clipboard_2021-11-28-17-06-15.png)

We could have the following forwarding table:

![](@attachment/Clipboard_2021-11-28-17-06-57.png)

This is called *prefix matching* where the destination is matched with the table. If there is a match the router forwards the packet to that link. 

For example, if we have the address: 
  11001000 00010111 00010110 10100001

The address matches the first prefix, so the link will be 0.

When there are multiple matches, the router uses the **longest prefix matching rule**; that is, it finds the longest matching entry in the table and forwards the packet to the link interface associated with the longest prefix match.

### Switching
The switching fabric is where the packets are forwarded from an input port to an putput port.

- **Switching via memory**
  - Switching under direct controll of the CPU. The packet is copied from the input port into the processor memory. The processor extracts the destination address from the header, look up the output port, and copy the packet to the output port's buffers. 
  - If the momroy bandwidth is such that a maximum of B packets per second can be written into, or read from, memory, then the overall forwarding throughput must be less than B/2. Two packet cannot be forwarded at the same time

![](@attachment/Clipboard_2021-12-07-13-06-47.png)

- **Switching via a bus**
  - Not involving the routing processor. Input port pre-pend a switch-internal label (header) to the packet indicating the local output port to which this packet is being transferred and transmitting the packet onto the bus. All output ports receive the packet, but only the port that matches the label will keep the packet. The label is removed in the output port. Only one packet can be in the bus at once. The switching speed of the router is limited to the bus speed.

![](@attachment/Clipboard_2021-12-07-13-07-26.png)

- **Switching vi an interconnection network**
  - Each vertical bus intersects each horizontal bus at a crosspoint, which can be opened or closed at any time by the switch fabric controller. 
  - When a packet arrives from port A and needs to be forwarded to port Y, the switch controller closes the crosspoint at the intersection of busses A and Y, and port A then sends the packet onto its bus, which is picked up (only) by bus Y. Packets from and to other ports may still be used.

![](@attachment/Clipboard_2021-12-07-13-36-27.png)


### Where does queuing occur?

Queuing may occur at both the input ports and the output ports. As these queues grow lagre the router's memory can eventually be exhaused and packet loss will occur when no memory is available to store arriving packets. 

**Input queueing**

If the switching fabric is not fast enough the packet queueing will occur at the input ports. 

In a crossbar switching fabric with FCFS, if two packets are at the front of two input queues are destined for the same output port, then one of the packets will be blocked and must wait at the input queue.

**Output queueing**

Suppose the switch speed is faster than the output link. In this case, in the time it takes to send a single packet onto the outgoing link, new packets will arrive at this output ports. Eventually, the number of queued packets can grow large enough to exhaust available memory at the output port.

When there is not enough memory to buffer an incoming packet, a decision must be made to either drop the arriving packet (**drop tail**) or remove one or more already-queued packets to make room for the newly arrived packet. 

### Packet scheduling

In a queue in the outport port there will be a packet scheduler that determines which packet will be transmitted first.

**First-in-First-Out (FIFO)**

Packets arriving will wait if the link is busy transmitting another packet. FIFO selects packets for link transmission in the same order in which they arrived at the output linke queue.

**Priority Queuing**

![](@attachment/Clipboard_2021-12-10-15-09-48.png)

Packets are classified into priority classes upon arrival at the queue. Real-time voice-over-IP packets might receive priority over non-real traffic such as SMTP or IMAP e-mail packets. Each priority class typically has its own queue. When choosing to transmit, the priority queuing discipline will transmit a packet from the highest priority class that has a nonempty queue. 

![](@attachment/Clipboard_2021-12-10-15-13-17.png)

**Round Robin and Weighted Fair Queuing (WFQ)**

For roundr robin queuing, packets are sorted into classes as with priority queuing. The scheduler alternates service among the classes. 

A generalized for of round robin is weighted fair queuing (WFQ).

![](@attachment/Clipboard_2021-12-11-10-21-39.png)


Illustrated above, the arriving packets are classified and queued in the appropriate per-class waiting area. WFQ will serve classes in a cicular manner. It will immediately move to the next class in the service sequence when it finds an empty class queue.

WFQ differs from round robin in that each class may receive a differential amount of service in any interval of time. Each class i is assigned a weight w. 

## Content-addressable memory introduction

Content-addressable memories (CAMs) are hardware search engines that are much faster than algorithmic approaches for search-intensive applications. The two most common search-intensive tasks that use CAMs are packet forwarding and packet classification in Internet routers. 

### CAM application: Router address lookup

The table below represents a rang on input addresses. Line 1 indicates that all addresses in the range 10100-10111 are forwarded to port A. If the router receives a packet with the incoming address 01101, the address lookup matches both Line 2 and Line 3. Line 2 is selected because it has the most defined bits, indicating it is the most direct route to the destination. This is **long-prefix matching**

![](@attachment/Clipboard_2021-12-11-10-51-43.png)

The routing parameters that determine the complexity of the implementation are the entry size, the table size, the searche rate, and the table update rate. 
- IPv4 address: 32 bits
- IPv6 address: 128 bits, may be 288-578 bits for QoS and source address
- Routing tables are about 30 000 entries and growing

Terabit-class routers must perform hundreds of millions of searches per second in addition to thousands of rouuting table updates per second.

Software search, like binary searching, is O(log n) in addition to the extra time required to insert a new entry in the table. Almost all algorithmix approaches are too slow to keep up with projected routing requirenments. CAMs use hardware to complete a search in a single cycle, resulting in constant O(1) time. This is accomplised by adding comparison circuits  to every cell of hardware memory. The strength of CAMs over algoritmic approaches is their high search throughput. The current bottlenexk is the large power consumption due to the large amount of comparison circuitry in parallel. 

### Content-Addressable Memory

There are two basic forms of CAM: binary and ternary.
- Binary: 0 and 1
- Ternary: 0, 1 and X (don't care) 

Ternary is most used. The table below is a 4x5 ternary CAM with a NOR-based architecture that represents the routing table above.

![](@attachment/Clipboard_2021-12-11-11-11-56.png)

The CAM core cells are arranged into foru horizontal words, each five bits long. Core cels contain both storage and comparison circuitry. The search lines run vertically in the figure and broadcast the search data to the CAM cells. The matchlines run horizontally across the array and indicate whether the search data mathces the row's word. An activated matchline indicates a match and a deactivated matchline indicates a non-match, called a mismatch in the CAM literature. The matchlines are inputs to an encoder that generates the address corresponding to the match location.

The encoder selecsts numberically the smallest numbered matchline of the two activated matchlines. This mach address is used as the input address to a RAM that contains a list of outputs ports as shown in the figure below. 

![](@attachment/Clipboard_2021-12-11-12-11-57.png)

This CAM/RAM system is a complete implementation of an address lookup engine.






















