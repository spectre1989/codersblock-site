+++
title = 'Making a Multiplayer FPS in C++ Part 4: Rebuilding The Client in C++'
date = 2017-07-03T00:00:00Z
draft = false
aliases = ["/blog/multiplayer-fps-part-4"]
includeToc = false
+++

In the previous part of this series we had an outline of an online game with a fixed tick rate, and support for multiple clients to connect and move around. In this part we bin the horrible client made in Unity, and implement one in C++. You can browse the repository at this stage of development here.

The reason I want to rebuild the client in C++ is that for upcoming work on client-side prediction, we'll need the server and client to have a certain amount of shared code. It's possible to re-implement this shared code in C# for the client, but it would be easier to maintain if we just had one codebase written in C++. Plus I just prefer doing stuff in C++, so there.

Unlike the server, the client will need to show visual stuff, which means it'll need to be a windowed application. I'm not a fan of making things by plumbing together a bunch of frameworks, and opening a window is actually deceptively simple. I won't go in to details here, there are plenty of guides out there for opening a window, I recommend the first few days of Casey Muratori's Handmade Hero series.

The other thing we'll need for 'visual stuff' is a way of getting our 'stuff' on screen. I'm intending for this to be a 3D game of some sort, so we'll need a way of displaying 3D graphics. Again, there are a lot of frameworks out there, but I have yet to find one I really like. So I'll be using Vulkan, a relatively new low-overhead, cross-platform (my main reason for choosing it over DirectX 12) graphics API. The code for this lives in client_graphics.cpp, I won't go in to detail about this (it's not really the point of this series), if you want to learn about Vulkan then I'd recommend looking at some of these learning resources.

So, I'll start from client.cpp line 105, this is where the windows set up stuff ends. We start with initialising the state we need for graphics. For the Unity client, each player was represented by a capsule, for this new client each player is represented by a rectangle (no actual 3D yet, that's a job for future Joe). For now I've implemented this with a single vertex buffer with 4 vertices per player to form each rectangle, each frame the vertices are updated and drawn.
```cpp
constexpr uint32 c_num_vertices = 4 * c_max_clients;
Graphics::Vertex vertices[c_num_vertices];
```
We have our array of vertices, we know the maximum vertices we'll need is the maximum number of clients multiplied by 4. 
```cpp
struct Vertex
{
   float32 pos_x;
   float32 pos_y;
   float32 col_r;
   float32 col_g;
   float32 col_b;
};
```
The Graphics::Vertex struct is defined in client_graphics.cpp. Moving on in client.cpp:
```cpp
srand((unsigned int)time(0));
for (uint32 i = 0; i < c_max_clients; ++i)
{
   // generate colour for client
   float32 r = 0.0f;
   float32 g = 0.0f;
   float32 b = 0.0f;

   float32 temp = (float32)(rand() % 100) / 100.0f;

   switch (rand() % 6)
   {
      case 0:
         r = 1.0f;
         g = temp;
      break;

      case 1:
         r = temp;
         g = 1.0f;
      break;

      case 2:
         g = 1.0f;
         b = temp;
      break;

      case 3:
         g = temp;
         b = 1.0f;
      break;

      case 4:
         r = 1.0f;
         b = temp;
      break;

      case 5:
         r = temp;
         b = 1.0f;
      break;
   }

   // assign colour to all 4 verts, and zero positions to begin with
   uint32 start_verts = i * 4;
   for (uint32 j = 0; j < 4; ++j)
   {
      vertices[start_verts + j].pos_x = 0.0f;
      vertices[start_verts + j].pos_y = 0.0f;
      vertices[start_verts + j].col_r = r;
      vertices[start_verts + j].col_g = g;
      vertices[start_verts + j].col_b = b;
   }
}
```
For each player we generate a random fully saturated colour, it looks a bit ugly, a proper HSV implementation would make it a lot cleaner. The colour is applied to all 4 of the players verts, and all verts will have position (0, 0) if there is no player to show.
```cpp
constexpr uint32 c_num_indices = 6 * c_max_clients;
uint16 indices[c_num_indices];

for(uint16 index = 0, vertex = 0; index < c_num_indices; index += 6, vertex += 4)
{
   // quads will be top left, bottom left, bottom right, top right
   indices[index] = vertex;              // 0
   indices[index + 1] = vertex + 2;      // 2
   indices[index + 2] = vertex + 1;      // 1
   indices[index + 3] = vertex;          // 0
   indices[index + 4] = vertex + 3;      // 3
   indices[index + 5] = vertex + 2;      // 2
}
```
We create the indices which will be uploaded to the index buffer to draw the rectangles. This won't need to change after it's set initially.
```cpp
Graphics::State graphics_state;
Graphics::init(window_handle, instance, c_window_width, c_window_height, 
               c_num_vertices, indices, c_num_indices, &graphics_state);
```
Finally we initialise the graphics system, which stores some relevant state in a Graphics::State struct which is used later to draw.
```cpp
// init winsock
if (!Net::init())
{
   log_warning("Net::init failed\n");
   return 0;
}
Net::Socket sock;
if (!Net::socket_create(&sock))
{
   log_warning("create_socket failed\n");
   return 0;
}
```
I've taken the winsock code needed by both the client and the server, and moved it to common_net.cpp. This just has some basic utlities for creating sockets, sending/receiving packets, etc. Some client/server specific netcode is in client_net.cpp and server_net.cpp respectively, this is just to stop the compiler complaining that I never call some of those functions when compiling one application or the other.
```cpp
uint8 buffer[c_socket_buffer_size];
Net::IP_Endpoint server_endpoint = Net::ip_endpoint_create(127, 0, 0, 1, c_port);

buffer[0] = (uint8)Client_Message::Join;
if (!Net::socket_send(&sock, buffer, 1, &server_endpoint))
{
   log_warning("join message failed to send\n");
   return 0;
}
```
This is the buffer we'll use for reading/writing packets. The first thing to do is tell the server we want to join.
```cpp
struct Player_State
{
   float32 x, y, facing;
};
Player_State objects[c_max_clients];
uint32 num_objects = 0;
uint16 slot = 0xFFFF;

Timing_Info timing_info = timing_info_create();
```
The client has it's own version of Player_State which only contains the state which the server sends to the client (on the server there is an additional speed field), and an array of these to store the state of all the player objects. We store the slot assigned to us by the server, as this is needed for us to send input packets. Lastly the Timing_Info struct contains some data about the frequency of the performance counter, and whether or not our desired sleep granularity was set - these are both used to wait for the end of the tick at the end of each main loop iteration (more detail on this in part 2).
```cpp
// main loop
g_is_running = 1;
while( g_is_running )
{
   LARGE_INTEGER tick_start_time;
   QueryPerformanceCounter(&tick_start_time);
```
At the beginning of each tick we sample the clock.
```cpp
   // Windows messages
   MSG message;
   UINT filter_min = 0;
   UINT filter_max = 0;
   UINT remove_message = PM_REMOVE;
   while( PeekMessage( &message, window_handle, filter_min, filter_max, remove_message ) )
   {
      TranslateMessage( &message );
      DispatchMessage( &message );
   }
```
Standard windows message pump stuff (look at win32 programming resources for further information on what this is).
```cpp
   // Process Packets
   Net::IP_Endpoint from;
   uint32 bytes_received;
   while (Net::socket_receive(&sock, buffer, c_socket_buffer_size, &from, &bytes_received))
   {
      switch (buffer[0])
      {
```
This loop will keep grabbing packets from the socket for as long as there are any available.
```cpp
         case Server_Message::Join_Result:
         {
            if(buffer[1])
            {
               memcpy(&slot, &buffer[2], sizeof(slot));
            }
            else
            {
               log_warning("server didn't let us in\n");
            }
         }
         break;
```

See server.cpp for corresponding code where this message is sent. The first byte is a 0 or 1 indicating whether we were allowed to join. If that byte is 1, then the next 2 bytes are a uint16 for our slot.
```cpp
         case Server_Message::State:
         {
            num_objects = 0;
            uint32 bytes_read = 1;
            while (bytes_read < bytes_received)
            {
               uint16 id; // unused
               memcpy(&id, &buffer[bytes_read], sizeof(id));
               bytes_read += sizeof(id);

               memcpy(&objects[num_objects].x, 
                      &buffer[bytes_read], 
                      sizeof(objects[num_objects].x));
               bytes_read += sizeof(objects[num_objects].x);

               memcpy(&objects[num_objects].y, 
                      &buffer[bytes_read], 
                      sizeof(objects[num_objects].y));
               bytes_read += sizeof(objects[num_objects].y);

               memcpy(&objects[num_objects].facing, 
                      &buffer[bytes_read], 
                      sizeof(objects[num_objects].facing));
               bytes_read += sizeof(objects[num_objects].facing);

               ++num_objects;
            }
         }
         break;
```

A state packet contains all the player objects, so we just read until we run out of bytes. The server sends the slot (called id here) which we currently don't use, in future we will use this field to figure out which object the player actually owns.
```cpp
      }
   }

   // Send input
   if (slot != 0xFFFF)
   {
      buffer[0] = (uint8)Client_Message::Input;
      int bytes_written = 1;

      memcpy(&buffer[bytes_written], &slot, sizeof(slot));
      bytes_written += sizeof(slot);

      uint8 input =  (uint8)g_input.up | 
                    ((uint8)g_input.down  << 1) | 
                    ((uint8)g_input.left  << 2) | 
                    ((uint8)g_input.right << 3);
      buffer[bytes_written] = input;
      ++bytes_written;

      if (!Net::socket_send(&sock, buffer, bytes_written, &server_endpoint))
      {
         log_warning("socket_send failed\n");
      }         
   }
```

Each tick we send our input. When key events are sent to the window, relevant information is stored in this global g_input struct. This packet just has the four buttons encoded in a bitfield.
```cpp
   // Draw
   for (uint32 i = 0; i < num_objects; ++i)
   {
      constexpr float32 size = 0.05f;
      float32 x = objects[i].x * 0.01f;
      float32 y = objects[i].y * -0.01f;

      uint32 verts_start = i * 4;
      vertices[verts_start].pos_x = x - size; // TL (hdc y is +ve down screen)
      vertices[verts_start].pos_y = y - size;
      vertices[verts_start + 1].pos_x = x - size; // BL
      vertices[verts_start + 1].pos_y = y + size;
      vertices[verts_start + 2].pos_x = x + size; // BR
      vertices[verts_start + 2].pos_y = y + size;
      vertices[verts_start + 3].pos_x = x + size; // TR
      vertices[verts_start + 3].pos_y = y - size;
   }
```
Here we run through each object we read from the most recent state packet, and compute the 4 vertex positions to represent it.
```cpp
   for (uint32 i = num_objects; i < c_max_clients; ++i)
   {
      uint32 verts_start = i * 4;
      for (uint32 j = 0; j < 4; ++j)
      {
         vertices[verts_start + j].pos_x = vertices[verts_start + j].pos_y = 0.0f;
      }
   }
```
Then zero the x and y positions of all vertices for player objects which have no actual player to show.
```cpp
   Graphics::update_and_draw(vertices, c_num_vertices, &graphics_state);

   wait_for_tick_end(tick_start_time, &timing_info);
}
```
Then pass the updated vertices to the graphics system to update and draw, and finally wait for the tick to end, before starting all over again!
```cpp
   buffer[0] = (uint8)Client_Message::Leave;
   int bytes_written = 1;
   memcpy(&buffer[bytes_written], &slot, sizeof(slot));
   Net::socket_send(&sock, buffer, bytes_written, &server_endpoint);
```

Just before the application exits, the client sends a message to the server to say we're going. If we don't send this, the server will time us out after some time, but it's preferable to notify the server properly so the slot that was allocated to this client can be re-used immediately.

This part doesn't feel like much of an improvement, just rebuilding what we already had. However, it's a necessary step for things we'll want to do later.