# Networking

### TCP/IP model vs OSI model
There's some confilict, but generally, OSI is more general.   
Mapping from TCP/IP to OSI:  
- Application -->  [application, presentation layer, and most of session]
- Transport --> [transport, the graceful close function of the session layer]
- Internet --> subset of Network
- Link --> [data link layer, physical layer, some protocols of the network layer]

### OSI
**note**: numbered layers are NOT USED in TCP/IP model  
  
#### L4 Transport Layer
Responsible for delivering data to the appropriate application process on the host computers.
Provides quality of life features:  
- reliablility  
- connection-oriented communication (in case of TCP)    
- flow control: rate of data transmission between two nodes    
- congestion avoidance: prevents buffer-overrun    
protocols: TCP, UDP (higher throughput lower latency, but packet loss ok), ...  

##### UDP
connection-less, can boradcast and multicast too:  
- boradcast: sent packets can be addressed to be receivable by all devices on the subnet  
- multicast: packet can be sent to very large numbers of subscribers  

#### L3 Network Layer
Host addressing (e.g. ip) and cross-network message forwarding (e.g. through routers/gateways)  
protocols: IPv4/IPv6, ICMP, WireGuard, IPsec  

