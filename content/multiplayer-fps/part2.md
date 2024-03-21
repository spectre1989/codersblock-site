+++
title = 'Making a Multiplayer FPS in C++ Part 2: The Main Loop'
date = 2016-08-21T00:00:00Z
draft = false
aliases = ["/blog/multiplayer-fps-part-2"]
includeToc = false
+++

The previous part of this series was at best the "Hello, world!" stage of an online game. Now it's time to actually start actually laying the groundwork.

### The Input Loop
Back in the day when multiplayer games were only played on a LAN, the clients would collect their user input, and send it to the server. The server would wait until it had the input from all clients, and then tick the game simulation, and send back the new game state. This is viable on a LAN because latency is so low, but it's not workable today, input lag of even a hundred milliseconds would feel sluggish, let alone two or three.

For now, I'll be doing LAN-style netcode - don't worry, it shouldn't remain like this for long, but it'll help simplify things at this early stage.

### Starting Slow
I started with a text adventure-ish input loop, browse the repository at this commit here. This is how the server now begins (I won't include the code for creating and binding the socket from before):
```cpp
int8 buffer[SOCKET_BUFFER_SIZE];
int32 player_x = 0;
int32 player_y = 0;

bool32 is_running = 1;
while( is_running )
{
   // get input packet from player
   int flags = 0;
   SOCKADDR_IN from;
   int from_size = sizeof( from );
   int bytes_received = recvfrom( sock, buffer, SOCKET_BUFFER_SIZE, flags, (SOCKADDR*)&from, &from_size );
   
   if( bytes_received == SOCKET_ERROR )
   {
      printf( "recvfrom returned SOCKET_ERROR, WSAGetLastError() %d", WSAGetLastError() );
      break;
   }
```
The first thing I'll point out is how at this point I started using typedefs like int32, I just like to show exactly how many bits are being used for basic types, and I don't like writing unsigned int, I far prefer uint32. Generally my style is to use these typedefs for all game code, but when I'm using Windows API functions I use whichever silly types they specify in the documentation on MSDN, e.g. UINT, DWORD, etc.

The recvfrom is essentially the same as before, but now this will be called in a loop. The only actual game state at this stage is the player x and y values, so they live outside the loop. On to processing the client packet:
```cpp
// process input
char client_input = buffer[0];
printf( "%d.%d.%d.%d:%d - %c\n", from.sin_addr.S_un.S_un_b.s_b1, from.sin_addr.S_un.S_un_b.s_b2, from.sin_addr.S_un.S_un_b.s_b3, from.sin_addr.S_un.S_un_b.s_b4, from.sin_port, client_input );

switch( client_input )
{
   case 'w':
      ++player_y;
   break;

   case 'a':
      --player_x;
   break;

   case 's':
      --player_y;
   break;

   case 'd':
      ++player_x;
   break;

   case 'q':
      is_running = 0;
   break;

   default:
      printf( "unhandled input %c\n", client_input );
   break;
}
```
The server just expects a single character for input w/a/s/d to move, and q to quit. Now to create the state packet and send it back to the player:
```cpp
// create state packet
int32 write_index = 0;
memcpy( &buffer[write_index], &player_x, sizeof( player_x ) );
write_index += sizeof( player_x );

memcpy( &buffer[write_index], &player_y, sizeof( player_y ) );
write_index += sizeof( player_y );

memcpy( &buffer[write_index], &is_running, sizeof( is_running ) );

// send back to client
int buffer_length = sizeof( player_x ) + sizeof( player_y ) + sizeof( is_running );
flags = 0;
SOCKADDR* to = (SOCKADDR*)&from;
int to_length = sizeof( from );
if( sendto( sock, buffer, buffer_length, flags, to, to_length ) == SOCKET_ERROR )
{
   printf( "sendto failed: %d", WSAGetLastError() );
   return;
}
```
The game state is copied into the buffer using memcpy, and sent like we saw last time with sendto. Now for the client:
```cpp
int8 buffer[SOCKET_BUFFER_SIZE];
int32 player_x;
int32 player_y;

printf( "type w, a, s, or d to move, q to quit\n" );
bool32 is_running = 1;
while( is_running )
{
   // get input
   scanf_s( "\n%c", &buffer[0], 1 );

   // send to server
   int buffer_length = 1;
   int flags = 0;
   SOCKADDR* to = (SOCKADDR*)&server_address;
   int to_length = sizeof( server_address );
   if( sendto( sock, buffer, buffer_length, flags, to, to_length ) == SOCKET_ERROR )
   {
      printf( "sendto failed: %d", WSAGetLastError() );
      return;
   }
```
Input is collected from the console using scanf_s, straight in to the buffer, and that single byte is sent the same way as before with sendto.
```cpp
   // wait for reply
   flags = 0;
   SOCKADDR_IN from;
   int from_size = sizeof( from );
   int bytes_received = recvfrom( sock, buffer, SOCKET_BUFFER_SIZE, flags, (SOCKADDR*)&from, &from_size );
   
   if( bytes_received == SOCKET_ERROR )
   {
      printf( "recvfrom returned SOCKET_ERROR, WSAGetLastError() %d", WSAGetLastError() );
      break;
   }

   // grab data from packet
   int32 read_index = 0;

   memcpy( &player_x, &buffer[read_index], sizeof( player_x ) );
   read_index += sizeof( player_x );

   memcpy( &player_y, &buffer[read_index], sizeof( player_y ) );
   read_index += sizeof( player_y );

   memcpy( &is_running, &buffer[read_index], sizeof( is_running ) );

   printf( "x:%d, y:%d, is_running:%d\n", player_x, player_y, is_running );
}
```
The client waits for the state packet, unpacks it, and displays the result in the console, before continuing for the next iteration of the loop.

Speeding Up
This is all well and good so far, but I want my game to be real-time, which will require the loop to run many times per second. The client is also going to have to cease being a console application. Given that the server is the focus of this project, for now I'll use Unity to throw something together quickly and easily. The code for the client will be included in the repository, but I won't go through it here, it's all very simple though. Here are the changes I made to the server, browse the repository at this commit here.
```cpp
int8 buffer[SOCKET_BUFFER_SIZE];
float32 player_x = 0.0f;
float32 player_y = 0.0f;
float32 player_facing = 0.0f;
float32 player_speed = 0.0f;

bool32 is_running = 1;
while( is_running )
{
```
I thought I'd make the player object some kind of vehicle, so some extra game state will be needed. As before there'll be player_x and player_y, but now there'll be player_facing. This needs only be a single float32 to describe their rotation, as they will only rotate around the z-axis. Finally there's player_speed, this is how fast they move in whatever direction they're facing.

The player will push the w/s keys to speed up and slow down, and a/d keys to turn. Client input is received by the server in the same manner as before, but we'll pick up at the point where the input is processed:
```cpp
   // process input and update state
   int8 client_input = buffer[0];
   printf( "%d.%d.%d.%d:%d - %d\n", from.sin_addr.S_un.S_un_b.s_b1, from.sin_addr.S_un.S_un_b.s_b2, from.sin_addr.S_un.S_un_b.s_b3, from.sin_addr.S_un.S_un_b.s_b4, from.sin_port, client_input );

   if( client_input & 0x1 )   // forward
   {
      player_speed += ACCELERATION;
      if( player_speed > MAX_SPEED )
      {
         player_speed = MAX_SPEED;
      }
   }
   if( client_input & 0x2 )   // back
   {
      player_speed -= ACCELERATION;
      if( player_speed < 0.0f )
      {
         player_speed = 0.0f;
      }
   }
   if( client_input & 0x4 )   // left
   {
      player_facing -= TURN_SPEED;
   }
   if( client_input & 0x8 )   // right
   {
      player_facing += TURN_SPEED;
   }

   player_x += player_speed * sinf( player_facing );
   player_y += player_speed * cosf( player_facing );
```
Unlike the previous iteration, the player can now press multiple keys at once, so all four keys are combined into a single byte. The first four bits of the byte indicate if the keys for forward, back, left, and right respectively, are held. The rest of the byte is unused for now. These four inputs update the player speed and facing, and finally this is used to update the position of the player. This is a very wonky approximation of physics, but it can be improved upon later.
```cpp
   // create state packet
   int32 bytes_written = 0;
   memcpy( &buffer[bytes_written], &player_x, sizeof( player_x ) );
   bytes_written += sizeof( player_x );

   memcpy( &buffer[bytes_written], &player_y, sizeof( player_y ) );
   bytes_written += sizeof( player_y );

   memcpy( &buffer[bytes_written], &player_facing, sizeof( player_facing ) );
   bytes_written += sizeof( player_facing );
```
Finally the state packet is written, this time incorporating the player facing as well.

Fixing The Tick Rate
Currently the server is not measuring the time between loop iterations, so the simulation speed will be different depending on the hardware it's run on, so I'll fix the tick rate to 60hz.

What we'll do is, measure the time that each tick takes, and then wait until it's time to start the next tick. We could just spin in a loop until that time is up, but that'll waste a lot of processing power. We'll put our thread to sleep using the sleep function:
```cpp
VOID WINAPI Sleep(
  _In_ DWORD dwMilliseconds
);
```
This has two problems though, firstly we can only specify the time to sleep in milliseconds, so we'll have to spin in a loop for any time which is less than a millisecond. Secondly, the windows scheduler itself might only check to see if it needs to wake up our thread once every ten milliseconds. For this reason, we'll have to attempt to set the granularity of the scheduler using timeBeginPeriod:
```cpp
MMRESULT timeBeginPeriod(
   UINT uPeriod
);
```
Note - the documentation on MSDN says "You must match each call to timeBeginPeriod with a call to timeEndPeriod", as is often the case with MSDN this is not true! Windows will clean this up for you after the application exits, you only need to call timeEndPeriod if you no longer need the granularity, but your program will continue running.

We can record a timestamp using the Windows API function QueryPerformanceCounter: 
```cpp
BOOL WINAPI QueryPerformanceCounter(
  _Out_ LARGE_INTEGER *lpPerformanceCount
);
```
This returns a measurement of time, but that measurement is not known at compile time (e.g. microseconds, nanoseconds etc). To convert it to some denomination of seconds, we need to also call QueryPerformanceFrequency, which will tell us how many counts there are per second:
```cpp
BOOL WINAPI QueryPerformanceFrequency(
  _Out_ LARGE_INTEGER *lpFrequency
);
```
You can browse the code at the relevant commit here, I started with the following code before the server loop begins:
```cpp
UINT sleep_granularity_ms = 1;
bool32 sleep_granularity_was_set = timeBeginPeriod( sleep_granularity_ms ) == TIMERR_NOERROR;

LARGE_INTEGER clock_frequency;
QueryPerformanceFrequency( &clock_frequency );
```
Then at the start of a loop, we record the current time:
```cpp
while( is_running )
{
   LARGE_INTEGER tick_start_time;
   QueryPerformanceCounter( &tick_start_time );
```
In all the places where the player changes speed and turns, we need to take the length of a tick in to account (well, we don't need to at all actually, but this will massively reduce headaches if we decide to change the tick rate later):
```cpp
   player_speed += ACCELERATION * SECONDS_PER_TICK;
   // ...
   player_facing += TURN_SPEED * SECONDS_PER_TICK;
   // ...
   player_x += player_speed * SECONDS_PER_TICK * sinf( player_facing );
```
Then at the end of the tick we measure how much time has elapsed since the beginning of the loop, I wrote a convenience function for this:
```cpp
static float32 time_since( LARGE_INTEGER t, LARGE_INTEGER frequency )
{
   LARGE_INTEGER now;
   QueryPerformanceCounter( &now );

   return float32( now.QuadPart - t.QuadPart ) / float32( frequency.QuadPart );
}
```
Back to the code that goes at the end of the loop:
```cpp
   float32 time_taken_s = time_since( tick_start_time, clock_frequency );

   while( time_taken_s < SECONDS_PER_TICK )
   {
      if( sleep_granularity_was_set )
      {
         DWORD time_to_wait_ms = DWORD( ( SECONDS_PER_TICK - time_taken_s ) * 1000 );
         if( time_to_wait_ms > 0 )
         {
            Sleep( time_to_wait_ms );
         }
      }

      time_taken_s = time_since( tick_start_time, clock_frequency );
   }
```
Notice that sleep is only called if the call to timeBeginPeriod actually succeeded earlier, if for some reason it failed then we'll just have to spin in the loop. When calculating how many milliseconds to sleep for, if we have 1.99 milliseconds left until the next tick begins, then we only sleep for 1 millisecond, and then spin for the remaining 0.99. This is why the calculation of time_to_wait_ms is truncated rather than rounded.

And that's it!