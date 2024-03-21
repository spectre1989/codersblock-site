+++
title = 'Making a Multiplayer FPS in C++ Part 3: Multiple Players'
date = 2016-08-30T00:00:00Z
draft = false
aliases = ["/blog/multiplayer-fps-part-3"]
includeToc = false
+++

In the previous part of this series we ended up with a basic outline of an online game with some very simple game logic, a fixed tick rate, and support for a single connected client. In this part, we'll be adding support for multiple clients. You can see the repository as it was at this stage of development here.

The first thing we'll need is to actually keep track of the clients who send the server packets. At the moment we store their IP address and port we get from recvfrom, but now we'll need some kind of list of IP addresses and ports of the clients so they can be sent state packets en masse. For this I created an IP_Endpoint struct:
```cpp
struct IP_Endpoint
{
   uint32 address;
   uint16 port;
};
```
Then we need an array of them - the size of which dependent upon game design, for now I'm using 32, but I'll store it in a const so it's easily changeable. Now that we're having multiple players, we'll also need their x, y, facing, and speed values which make up their state. I'll group those in a Player_State struct:
```cpp
struct Player_State
{
   float32 x, y, facing, speed;
};
```
Currently our code waits for user input before executing the next tick, but this won't work for multiple users (even on a LAN if we have enough users). Eventually clients will deliver input packets slightly ahead of the tick they are needed for, but for now we'll just store whatever the most recent input was for each player. Then on each tick we blast through all of them, updating each Player_State struct accordingly. So for that reason I created a Player_Input struct:
```cpp
struct Player_Input
{
   bool32 up, down, left, right;
};
```
But we have a problem - when a player sends us a packet, how do we know if they've sent us a packet before? We could search through the array of IP_Endpoints for their address and port, this would probably be OK for a game with a relatively small number of players on each server, but for something like an MMO it would negatively affect performance.

An easy way to solve this is to assign each player a unique identifier when they join the server, which is actually an index into an array. Any packet sent from the client to the server must include this ID, the server can then check that the address and port that the packet came from, matches those in the IP_Endpoint array (otherwise a player could pretend to be someone else by sending a different ID).

For this, we'll now need to start having multiple packet types. Previously, every packet from the client to the server contained user input, and every packet from the server to the client contained game state. That will no longer be the case, we'll need packets for the following:

client joining server (requesting ID)

server telling client their ID

client leaving server (a timeout would do, but this is better)

client sending user input to server

server sending game state to client

Every packet will have to start with a number describing the type of packet it is, for now this needs only be one byte as we have less than 256 packet types. I'll use an enum for this, or rather, two enums - one for client messages, and one for server messages:
```cpp
enum class Client_Message : uint8
{
   Join,      // tell server we're new here
   Leave,      // tell server we're leaving
   Input       // tell server our user input
};

enum class Server_Message : uint8
{
   Join_Result,// tell client they're accepted/rejected
   State       // tell client game state
};
```
Having a message telling the server that the client is leaving is a good idea, but what happens if the client crashes? We'll need a simple timeout system. Clients will always be sending a steady stream of input packets, so we'll just have a per-client counter which is incremented every tick, and reset back to zero when an input packet is received. This will tell us how long it's been since we've heard from that client. 

So now, just before entering the main loop on the server, we'll have these arrays:
```cpp
   IP_Endpoint client_endpoints[MAX_CLIENTS];
   float32 time_since_heard_from_clients[MAX_CLIENTS];
   Player_State client_objects[MAX_CLIENTS];
   Player_Input client_inputs[MAX_CLIENTS];
```
When we only supported one client, the server would wait for input from the client before carrying on with the tick. Part of the reason for this is that recvfrom is a blocking function, it won't return until a packet is received. We don't want our server to ever wait like this, so we could call recvfrom on a different thread, but an easier method which will do fine for now is to switch the socket to non-blocking mode, like this:
```cpp
u_long enabled = 1;
ioctlsocket( sock, FIONBIO, &enabled );
```
Any calls to recvfrom made when there are no packets to consume, will return SOCKET_ERROR, and WSAGetLastError will return WSAEWOULDBLOCK. So now on each tick, we just call recvfrom until is returns SOCKET_ERROR, and then get on with updating the game state and sending it back to the clients:
```cpp
while( true )
{
   int flags = 0;
   SOCKADDR_IN from;
   int from_size = sizeof( from );
   int bytes_received = recvfrom( sock, buffer, SOCKET_BUFFER_SIZE, flags, (SOCKADDR*)&from, &from_size );
   
   if( bytes_received == SOCKET_ERROR )
   {
      int error = WSAGetLastError();
      if( error != WSAEWOULDBLOCK )
      {
         printf( "recvfrom returned SOCKET_ERROR, WSAGetLastError() %d\n", error );
      }
      
      break;
   }

   IP_Endpoint from_endpoint;
   from_endpoint.address = from.sin_addr.S_un.S_addr;
   from_endpoint.port = from.sin_port;

   switch( buffer[0] )
   {
      case Client_Message::Join:
      {
         // ...
      }
      break;

      case Client_Message::Leave:
      {
         // ...
      }
      break;

      case Client_Message::Input:
      {
         // ...
      }
      break;
   }
}
```
The case for Client_Message::Join looks like this:
```cpp
uint16 slot = uint16( -1 );
for( uint16 i = 0; i < MAX_CLIENTS; ++i )
{
   if( client_endpoints[i].address == 0 )
   {
      slot = i;
      break;
   }
}

buffer[0] = (int8)Server_Message::Join_Result;
if( slot != uint16( -1 ) )
{
   buffer[1] = 1;
   memcpy( &buffer[2], &slot, 2 );

   flags = 0;
   if( sendto( sock, buffer, 4, flags, (SOCKADDR*)&from, from_size ) != SOCKET_ERROR )
   {
      client_endpoints[slot] = from_endpoint;
      time_since_heard_from_clients[slot] = 0.0f;
      client_objects[slot] = {};
      client_inputs[slot] = {};
   }
}
else
{
   buffer[1] = 0;

   flags = 0;
   sendto( sock, buffer, 2, flags, (SOCKADDR*)&from, from_size );
}
```
First we look for an empty slot, this is determined by finding an IP_Endpoint in the array with an address of 0 (these are zero-initialised during startup, and set to 0 when a client leaves or times out). If an available slot is found, then this is used for the client ID. A Join_Result message is sent back to the client, the second byte of the packet indicates whether joining was successful, and if so, the assigned client ID follows. For now I'm using a 16-bit unsigned integer - I figure for development purposes we're unlikely to need more than ~65000 client IDs. If the slot was found, we store the IP_Endpoint in the array, and zero-initialise their Player_State and Player_Input.

The case for Client_Message::Leave just zeroes the IP_Endpoint for that client:
```cpp
uint16 slot;
memcpy( &slot, &buffer[1], 2 );

if( client_endpoints[slot] == from_endpoint )
{
   client_endpoints[slot] = {};
}
```
Finally the case for Client_Message::Input grabs the user input (this time in the fourth byte of the packet) and does some bitwise operations to convert back into individual key presses:
```cpp
uint16 slot;
memcpy( &slot, &buffer[1], 2 );

if( client_endpoints[slot] == from_endpoint )
{
   uint8 input = buffer[3];

   client_inputs[slot].up = input & 0x1;
   client_inputs[slot].down = input & 0x2;
   client_inputs[slot].left = input & 0x4;
   client_inputs[slot].right = input & 0x8;

   time_since_heard_from_clients[slot] = 0.0f;
}
```
Next comes the actual update loop:
```cpp
for( uint16 i = 0; i < MAX_CLIENTS; ++i )
{
   if( client_endpoints[i].address )
   {
      if( client_inputs[i].up )
      {
         client_objects[i].speed += ACCELERATION * SECONDS_PER_TICK;
         if( client_objects[i].speed > MAX_SPEED )
         {
            client_objects[i].speed = MAX_SPEED;
         }
      }
      if( client_inputs[i].down )
      {
         client_objects[i].speed -= ACCELERATION * SECONDS_PER_TICK;
         if( client_objects[i].speed < 0.0f )
         {
            client_objects[i].speed = 0.0f;
         }
      }
      if( client_inputs[i].left )
      {
         client_objects[i].facing -= TURN_SPEED * SECONDS_PER_TICK;
      }
      if( client_inputs[i].right )
      {
         client_objects[i].facing += TURN_SPEED * SECONDS_PER_TICK;
      }

      client_objects[i].x += client_objects[i].speed * SECONDS_PER_TICK * sinf( client_objects[i].facing );
      client_objects[i].y += client_objects[i].speed * SECONDS_PER_TICK * cosf( client_objects[i].facing );

      time_since_heard_from_clients[i] += SECONDS_PER_TICK;
      if( time_since_heard_from_clients[i] > CLIENT_TIMEOUT )
      {
         client_endpoints[i] = {};
      }
   }
}
```
We iterate over each IP_Endpoint and for those which are in use, update their Player_State based on their current Player_Input. Clients timing-out is also handled here. There is a minor issue here that we iterate over the entire IP_Endpoint array, including all of those which aren't in use. Not to worry, I will deal with this at a later date.

Then the actual state packet is created. For now, everyone gets sent the same game state packet. It's possible that some games benefit from different players being sent different subsets of the game state, e.g. for anti-cheat reasons, or because the entire game state is huge (tangent - this is why I'm always skeptical of off-the-shelf networking systems, different games have different networking requirements):
```cpp
buffer[0] = (int8)Server_Message::State;
int32 bytes_written = 1;
for( uint16 i = 0; i < MAX_CLIENTS; ++i )
{
   if( client_endpoints[i].address )
   {
      memcpy( &buffer[bytes_written], &i, sizeof( i ) );
      bytes_written += sizeof( i );

      memcpy( &buffer[bytes_written], &client_objects[i].x, sizeof( client_objects[i].x ) );
      bytes_written += sizeof( client_objects[i].x );

      memcpy( &buffer[bytes_written], &client_objects[i].y, sizeof( client_objects[i].y ) );
      bytes_written += sizeof( client_objects[i].y );

      memcpy( &buffer[bytes_written], &client_objects[i].facing, sizeof( client_objects[i].facing ) );
      bytes_written += sizeof( client_objects[i].facing );
   }
}
```
You might be thinking - why not write this state to the packet during the previous loop? Well, we actually could right now, and it would work fine. The problem is that at some point in the hopefully not-too-distant future, players will start to be able to shoot at each other and so forth. That means that we can't definitely be sure that a player hasn't been destroyed until the end of the tick, so only at that point can we be sure that they should be included in the game state packet.

Finally the state packet is sent to all the clients:
```cpp
int flags = 0;
SOCKADDR_IN to;
to.sin_family = AF_INET;
int to_length = sizeof( to );

for( uint16 i = 0; i < MAX_CLIENTS; ++i )
{
   if( client_endpoints[i].address )
   {
      to.sin_addr.S_un.S_addr = client_endpoints[i].address;
      to.sin_port = client_endpoints[i].port;

      sendto( sock, buffer, bytes_written, flags, (SOCKADDR*)&to, to_length );
   }
}
```
So without too much work, we have a server which supports multiple users.