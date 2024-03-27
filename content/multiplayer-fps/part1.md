+++
title = "Making a Multiplayer FPS in C++ Part 1: Hello Multiplayer World"
filename = "Part 1: Hello Multiplayer World"
date = 2016-08-14T00:00:00Z
draft = false
aliases = ["/blog/multiplayer-fps-part-1"]
includeToc = false
+++

I write netcode because I love it. The problems that have to be solved in online games are particularly interesting to me - how to hide lag, how to replicate game state, how to compress data to save bandwidth, etc. I've spent most of my career working on multiplayer games of one sort or another - I built a couple of MMO clients at an indie startup, I was part of the online services team of Total War: Arena, and most recently I was on the network team for Halo Wars 2 DLC. My work has always been confined to just the client, or just services, or just some particular area. It's a necessity of modern game development that you can't do everything (or even some of everything), unless you're at a really small studio. I'd like to though, so I'm going to try to do it in some small way, and write about it here.

In particular my interest is in networking action games, and specifically the king of multiplayer action games - twitchy first-person shooters.

### The Plan
The fundamental (and most interesting in my view) part of any online game is the ***network model*** - i.e. how is the game state replicated between different players, and how do you go about letting all the players affect their shared game world.

The first question is topology - ***peer-to-peer*** or ***client-server***? The former has all players connected to each-other, and in the latter all players connect to a common server machine which acts as a relay for data. Generally speaking, peer-to-peer doesn't scale well. It has some real advantages, but I'd like to be able to handle 20 or 30 players, perhaps more, so that alone means client-server is the only option.

Then there's the question of trust - how much can the server trust the client. You can let the client tell the server "I'm over here now, and I'm doing xyz", or you can have the client send their input to the server, and the server tells them the result. It is easier to trust the client, but it makes it far easier for players to cheat. 

Due to potential cheating, trusting the client really isn't an option for a competitive game. Even for a purely co-operative game, cheating can really ruin the whole experience for those who want to play properly. Maybe it's appropriate for a game which will only really be played with friends or something.

Hybrid solutions are also possible, where the server trusts the client a bit, but performs validation. For example the client tells the server where they are, and the server does its best to make sure they're not moving too fast and haven't walked through any solid objects.

I don't like the hybrid solution to be honest. If the validation is too strict, it may detect false positives when looking for cheating. On the other hand, if it's not strict enough, it'll miss things. For those reasons, I like servers to be fully authoritative, the downside of which is that the server ends up doing more work, as it has to fully run the game simulation.

When synchronising the server and client, there are a few options to consider:

* ***Snapshot Interpolation*** - server sends regular snapshots of visual state (e.g. position and rotation), the client interpolates between them
{{<asset_img "/images/mpfps_01/img001.png">}}
* ***State Synchronisation*** - server sends a fuller set of game state (e.g. sending position, rotation, and velocity), and the simulation runs on both the client and the server which fills in the gaps between the received sets of state
{{<asset_img "/images/mpfps_01/img002.png">}}
* ***Deterministic Lockstep*** - clients send their input to eachother, and then all step the simulation forward as one

I think snapshot interpolation is the best option, though it does have some downsides. The interpolation itself does introduce some extra latency (i.e. the client always needs at least 2 state packets in hand, to interpolate ***from*** and ***to***.)

State synchronisation however can be a real pain - you have to carefully pick which parts of the state to send to the client (you want to save bandwidth). A change to game code can require a change to network code, this is very brittle, and asking for multiplayer bugs which are difficult to diagnose. In order for the client to run the simulation for replicated objects, the state sent from server to client is larger than with snapshots. In addition to this, the data sent to the client has to be more accurate, and cannot be as compressed as snapshots can be. As a result of both of these points, individual objects have to be updated less frequently, this means that when the client and server do diverge, it will take longer to correct - resulting in unsightly popping.

The last option in the list is only really suitable for RTS games, or LAN-games. More info on those strategies [here](http://gafferongames.com/networked-physics/introduction-to-networked-physics/).

### It Starts With Sockets
Sending data from one machine to another is done by bundling together a series of bytes in a ***packet***, and sending it via a ***socket*** on the sending machine, to another socket on the receiving machine. That packet might reach its destination, or it might not. It might even arrive multiple times. A series of packets could very well arrive in a completely different order from which they were sent. This sounds pretty chaotic, which might lead you to consider using TCP sockets, which guarantee that packets will arrive reliably at their destination, once, in the order in which they were sent.

### TCP: Not Even Once
For any kind of action game, TCP really isn't viable. I'd argue against using it even for an MMORPG with typical lock-on combat and so on. When the server is sending its stream of state packets, if one of them is dropped, then with TCP not only is it resent, but the order of packets must also be preserved. If a client doesn't receive one of these packets, the underlying implementation of TCP will not make the subsequent state packets available until the dropped packet has been successfully resent. From the perspective of the client, this looks like a halt in game state packets, and then a bunch all arrive at once. This is very unhelpful because the client usually only cares about the most recent state packet, and would rather just skip over the dropped packet. The only way to really deal with this is to have the client always run far enough behind the server that the player won't notice when a packet gets dropped, but this is completely unworkable for something like a twitchy multiplayer FPS (which will live or die on the quality of its netcode).

### UDP to the Rescue
The socket protocol I'll be using is UDP, which behaves a lot more like the underlying internet protocol. You can have duplicate packets, and you can have packets dropped, but a packet will either get there in it's entirety or not at all. There will be some kinds of packets which we will absolutely need to reliably reach their destination, possibly in the correct order. However, it's a bad idea to use TCP alongside UDP, as this can [actually induce packet loss in UDP](https://www.isoc.org/inet97/proceedings/F3/F3_1.HTM). Instead we'll be creating our own reliability system with UDP.

### Hello Multiplayer World
So now to start with some actual code. I've created a [github repository](https://github.com/spectre1989/odin) for this which I've called "Odin" (I got into a pretentious habit of naming frameworks after Norse gods). Though the code will move past this point, you can browse the code at this specific commit [here](https://github.com/spectre1989/odin/tree/c75d394ef2dfe9d71b8229836a304449f36e1940).

I created a shiny new file 'server.cpp', and stuck in a bog-standard main function which will be the entry point of the server program. Depending on how complicated things get, multiple server programs may eventually exist, but for now there's just one.

To use sockets on Windows, we'll be using Winsock, so the first thing to do is include the Winsock header <winsock2.h>. I'll also include <stdio.h> for things like printf. Before we can use any Winsock functions, we need to first call WSAStartup. Looking at the MSDN documentation, the function signature looks like this:
```cpp
int WSAStartup(
  _In_  WORD      wVersionRequested,
  _Out_ LPWSADATA lpWSAData
);
```
We need to pass a WORD as the version we want, and a pointer to a WSADATA struct which is populated by the call (though I've never actually used it). It will return 0 on success (yay Windows!), and if there is a problem we can get the error code with WSAGetLastError. The only question is what version we ask for, according to the documentation the current version is 2.2, and is supported from Windows 98 onwards, so we'll go for that.
```cpp
WORD winsock_version = 0x202;
WSADATA winsock_data;
if( WSAStartup( winsock_version, &winsock_data ) )
{
	printf( "WSAStartup failed: %d", WSAGetLastError() );
	return;
}
```
The documentation also says we should have a call to WSACleanup when we're done with Winsock, you'll see those in the github repository, but I've since learned that it's unneccesary to call WSACleanup on exit, as Windows will clean up Winsock for us anyway. It would only be useful to us if we wanted to unload Winsock but keep the program running for some other purpose. Now to create a socket, this can be done using the socket function - Winsock does give alternatives, but this is the one I use:
```cpp
SOCKET WSAAPI socket(
  _In_ int af,
  _In_ int type,
  _In_ int protocol
);
```
The first argument af stands for address family, and we'll be passing AF_INET for IPv4, though we might support IPv6 later. The second is type, for a UDP socket this needs to be SOCK_DGRAM. Finally protocol needs to be IPPROTO_UDP. If this returns INVALID_SOCKET then it has failed, and we can get more information about what went wrong with WSAGetLastError again.
```cpp
int address_family = AF_INET;
int type = SOCK_DGRAM;
int protocol = IPPROTO_UDP;
SOCKET sock = socket( address_family, type, protocol );

if( sock == INVALID_SOCKET )
{
	printf( "socket failed: %d", WSAGetLastError() );
	return;
}
```
When a packet is sent, not only is it sent to a particular machine, but also on a particular port. This allows a program to only receive the packets that it was actually meant to see. Before the server can start receiving packets from clients, the socket needs to be bound to a specific port. We need to pick one to use - anything below 1024 is reserved, I chose 9999. Here's the function signature for bind from MSDN:
```cpp
int bind(
  _In_ SOCKET                s,
  _In_ const struct sockaddr *name,
  _In_ int                   namelen
);
```
Again, it will return 0 on success, otherwise we can get the error with WSAGetLastError:
```cpp
SOCKADDR_IN local_address;
local_address.sin_family = AF_INET;
local_address.sin_port = htons( 9999 );
local_address.sin_addr.s_addr = INADDR_ANY;
if( bind( sock, (SOCKADDR*)&local_address, sizeof( local_address ) ) == SOCKET_ERROR )
{
    printf( "bind failed: %d", WSAGetLastError() );
    return;
}
```
Computer hardware doesn't always agree on the order in which to store bytes, usually they're either big-endian or little-endian (more on that [here](https://en.wikipedia.org/wiki/Endianness)). Machines with different endianness should still be able to communicate via sockets though, and for this reason there is a specified network byte order, which is big-endian. The assignment of the port number is wrapped in a call to htons (***h***ost ***t***o ***n***etwork ***s***hort), this converts the port number from whatever endianness the executing machine has (little-endian in my case), to big-endian.

The assignment of INADDR_ANY is to allow the socket to accept packets on all interfaces. We could if we wanted, bind the socket to only receive packets from other processes running on the same machine. In our case though, we want to accept connections from other processes on the same machine, or machines on the local network, or other machines via the internet.

Now to start receiving packets. For this we use recvfrom, which has the following function signature:
```cpp
int recvfrom(
_In_        SOCKET          s,
_Out_       char            *buf,
_In_        int             len,
_In_        int             flags,
_Out_       struct sockaddr *from,
_Inout_opt_ int             *fromlen
);
```
We need to pass in a buffer where it will store data. It's important that this is no smaller than the maximum size of packet which we will read. We'll be limiting the size of the packets we send, not just because we cannot exceed our [MTU](https://en.wikipedia.org/wiki/Maximum_transmission_unit), but mainly because we'll need to do everything we can to limit bandwidth. The smaller our state packets, the more frequently we can get away with sending them, and/or the more clients we can have connected to the same instance. For now I'll just go for a buffer of a kilobyte, we can return to this buffer size later.

The call to recvfrom also tells us who sent the packet to us. This is one of the main differences between TCP and UDP - we would need a TCP socket per client, whereas with UDP we can have all the clients send packets to a single socket. In future when we receive a packet, we'll use this to figure out which player it came from.
```cpp
char buffer[SOCKET_BUFFER_SIZE];
int flags = 0;
SOCKADDR_IN from;
int from_size = sizeof( from );
int bytes_received = recvfrom( sock, buffer, SOCKET_BUFFER_SIZE, flags, (SOCKADDR*)&from, &from_size );

if( bytes_received == SOCKET_ERROR )
{
   printf( "recvfrom returned SOCKET_ERROR, WSAGetLastError() %d", WSAGetLastError() );
}
else
{
   buffer[bytes_received] = 0;
   printf( "%d.%d.%d.%d:%d - %s", 
   from.sin_addr.S_un.S_un_b.s_b1, 
   from.sin_addr.S_un.S_un_b.s_b2, 
   from.sin_addr.S_un.S_un_b.s_b3, 
   from.sin_addr.S_un.S_un_b.s_b4, 
   from.sin_port, 
   buffer );
}
```
Here all I'm doing is interpreting the data as a string, so I write a null terminator (0) to the end of the data received, and display it with printf. Bonus points to anyone who spotted the bug with how I'm displaying the port number there, I've forgotten to convert the port number from network byte order, back to host byte order. This would be done with a call to ***ntohs***.

That's it for now, the server just starts up, waits for a message from a client. When it receives one it prints the message, then exits. Now for the client.

The initial call to WSAStartup and creating our socket is the same as for the server, however we don't need to bind the socket. If you plan send data on a socket before receiving, then Windows will implicitly bind the socket for us to some unused port. This is handy for running multiple instances of the client on one computer for testing purposes. To send data we'll call ***sendto***:
```cpp
int sendto(
_In_       SOCKET                s,
_In_ const char                  *buf,
_In_       int                   len,
_In_       int                   flags,
_In_       const struct sockaddr *to,
_In_       int                   tolen
);
```
My use of it looked like this:
```cpp
SOCKADDR_IN server_address;
server_address.sin_family = AF_INET;
server_address.sin_port = htons( PORT );
server_address.sin_addr.S_un.S_addr = inet_addr( "127.0.0.1" );

char message[SOCKET_BUFFER_SIZE];
gets_s( message, SOCKET_BUFFER_SIZE );

int flags = 0;
if( sendto( sock, message, strlen( message ), flags, (SOCKADDR*)&server_address, sizeof( server_address ) ) == SOCKET_ERROR )
{
	printf( "sendto failed: %d", WSAGetLastError() );
	return;
}
```
So we just grab an input message from the user with ***gets_s***, and send it to the server, and then exit. The server will display that message, and then exit. Not much of a game yet, but as Karl Pilkington once said - "every step starts with a step, or whatever".

To compile I just have two batch files, one called 'shell', and another called 'build'. I start up a new instance of command prompt in the Odin directory, run shell (which runs my Visual Studio installation's vcvarsall), and then run 'build' (which directly uses cl to compile my cpp files). I've been watching Casey Muratori's excellent [Handmade Hero](https://handmadehero.org/) series for a little while now, I've been influenced somewhat by his way of doing things - these batch files are an example of that. If you want to just create a Visual Studio solution then that's cool, whatever works for you.