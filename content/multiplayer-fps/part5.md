+++
title = 'Making a Multiplayer FPS in C++ Part 5: Client-Side Prediction'
date = 2017-12-31T00:00:00Z
draft = false
aliases = ["/blog/multiplayer-fps-part-5"]
includeToc = false
+++

In the previous part of the series, we rebuilt the game client in C++. Now we crack on with the real reason we bothered to do that - ***client-side prediction***. Currently the "game" only really works with close to no latency (i.e. client and server running on the same machine or network). Now is the time to introduce some artificial lag, *dealing* with that lag is really where netcode gets interesting.

In the real world, packets sent over the internet are unpredictable. Sometimes they don't arrive at all (packet loss), sometimes they do arrive, and sometimes they can even be duplicated and arrive multiple times (though I've never detected this actually happening, but it's important to know that it is possible and should probably be guarded against).

Latency is also unpredictable, multiple packets from one machine to another do not necessarily (and in my experience, *rarely*) take the same amount of time to travel from A to B. This is referred to as jitter. Even when network conditions are stable, in a short space of time (a couple of seconds, say) there can be quite a difference between the quickest and slowest packet. This can result in packets arriving in a different order from which they were sent, or arriving in clumps.

For now the fake lag I'll introduce is very rudimentary - there'll be predictable artificial latency added, but no packet loss or jitter just yet, though in time they'll be needed too. For now this is very simple, incoming/outgoing packets enter a buffer and stay there for however long the fake lag period is - I've gone for a fairly absurd 200ms in both directions, so a 400ms round-trip time (RTT). Once this delay has expired, they can be retrieved with a call to `socket_receive` in the case of incoming packets, and outgoing packets are sent via the underlying Winsock socket.

(You can see the code roughly where it was for this post [here](https://github.com/spectre1989/odin/tree/28261d9e6d68385647efade2dbd5837d06faf055), though there was a fair amount of overlap with the next article)

The way I've implemented this is to use the preprocessor symbol `FAKE_LAG` to insert extra code into builds which need the feature. In those builds, the original `Socket` struct (and all of the functions for manipulating it) is wrapped in a namespace called `Internal`, then another Socket struct is added which contains an `Internal::Socket`, along with buffers for sent and received packets. There's also a set of functions to work with them which match the interface of those for Socket, so it's just a case of turning fake lag on or off and it should just work without having to change anything else.
```cpp
#ifdef FAKE_LAG
namespace Internal
{
#endif
struct Socket
{
   SOCKET sock;
};
#ifdef FAKE_LAG
}

struct Socket
{
   Internal::Socket sock;
   Packet_Buffer send_buffer;
   Packet_Buffer recv_buffer;
};
#endif
```
The packet buffers are implemented just as a circular buffer, we know how big a packet can be (up to 1k), we know how many we send per second (60), and we know how much lag we'll have (200ms). This means we'll need a circular buffer of 12x1k buffers, but it's probably a good idea to have a couple spare just in case. 
```cpp
constexpr uint32 c_packet_buffer_size = (c_ticks_per_second * c_fake_lag_s) + 2;
struct Packet_Buffer
{
   uint8 packets[c_packet_buffer_size][c_socket_buffer_size];
   uint32 packet_sizes[c_packet_buffer_size];
   IP_Endpoint endpoints[c_packet_buffer_size];
   Time times[c_packet_buffer_size];

   uint32 head;
   uint32 tail;
};

bool packet_buffer_push(Packet_Buffer* packet_buffer, uint8* packet, uint32 packet_size, IP_Endpoint* endpoint)
{
   if ((packet_buffer->tail - packet_buffer->head) == c_packet_buffer_size)
   {
      return false;
   }

   uint32 tail = packet_buffer->tail % c_packet_buffer_size;
   memcpy(packet_buffer->packets[tail], packet, packet_size);
   packet_buffer->packet_sizes[tail] = packet_size;
   packet_buffer->endpoints[tail] = *endpoint;
   packet_buffer->times[tail] = time_now() + c_fake_lag_s;

   ++packet_buffer->tail; // note: this will take over 2 years to overflow

   return true;
}

bool packet_buffer_pop(Packet_Buffer* packet_buffer, uint8* out_packet, uint32* out_packet_size, IP_Endpoint* out_endpoint)
{
   if (packet_buffer->head == packet_buffer->tail)
   {
      return false;
   }

   uint32 head = packet_buffer->head % c_packet_buffer_size;
   if (packet_buffer->times[head] <= time_now())
   {
      memcpy(out_packet, 
         packet_buffer->packets[head], 
         packet_buffer->packet_sizes[head]);
      *out_packet_size = packet_buffer->packet_sizes[head];
      *out_endpoint = packet_buffer->endpoints[head];

      ++packet_buffer->head;

      return true;
   }

   return false;
}
```
Right now these are hardcoded in quite a rubbish way, but later this will be tidied up.

With this fake lag activated, input feels sluggish as you'd expect.

{{<asset_img "/images/mpfps_05/anim001.gif">}}
<sub>Square colour indicates what input is currently being pressed: White - up (increase speed) Red - down (decrease speed/stop) Blue - nothing\
Square colour indicates what input is currently being pressed:\
White - up (increase speed)\
Red - down (decrease speed/stop)\
Blue - nothing</sub>

***(The following section on handling of player inputs contains some mistakes, see [here](../part8) for corrections)***

Client-Side Prediction allows us to make input feel responsive, despite the lag. The client *predicts* the results of their input so the player sees the results instantly. They also maintain a historic buffer of inputs and their predicted results. When a state packet is received from the server, they compare the predicted results against the servers actual results. If the client's predicted results don't match, then a ***misprediction*** has occurred, and they replay the historic buffer from the corrected state, back up to the client's current predicted state. For the client to predict the results of their input, they need to run the same movement logic on the client as will run on the server - this is why it's helpful to have the server and client as part of a single codebase, hence rebuilding the client in C++ in the last part of the series.

This means that effectively the client has to predict forwards in time relative to the other players they are observing, and this amount of time is proportional to their latency. If the client has a round-trip time to the server of 10 ticks, then they must run at least 10 ticks ahead of their last received state from server (and 5 ticks ahead of where the server is now, because the packet will take 5 ticks to get there). I think of this as 3 timestreams:

* ***server time*** - the current time on the server (the client can only approximate what tick the server is currently on)
* ***predicted time*** - the client's locally predicted time (server_time + (rtt * 0.5) + extra_for_jitter)
* ***observed time*** - the stuff the client is observing rather than predicting (e.g. other players, AI characters, server-side physics objects, etc), and interpolates between snapshots of visual state (server_time - (rtt * 0.5) - extra_for_jitter_and_interpolation)

{{<asset_img "/images/mpfps_05/img001.png">}}
<sub>The client is predicting 5 ticks ahead of the server.\
They send input for tick 50, it reaches the server 5 ticks later.\
The server sends a state packet for tick 50 back to the client, which arrives when the client is predicting to tick 60.</sub>

{{<asset_img "/images/mpfps_05/img002.png">}}
<sub>The client has predicted up to tick 53.</sub>

{{<asset_img "/images/mpfps_05/img003.png">}}
<sub>A state packet is received from the server for tick 51, the client compares the local player state with what they predicted, they don't match, there has been a misprediction.</sub>

{{<asset_img "/images/mpfps_05/img004.png">}}
<sub>The client re-simulates the remaining predicted ticks.</sub>

I've implemented the prediction history as another circular buffer. This buffer just needs to be big enough to allow for the highest round-trip time that we plan to support, I've set this at 2 seconds for now, which will allow for a round-trip time of almost 4 seconds - much bigger than we'd ever really need.
```cpp
Player_State local_player = {};
constexpr uint32 c_max_predicted_ticks = c_ticks_per_second * 2;
Player_State prediction_history_state[c_max_predicted_ticks];
Player_Input prediction_history_input[c_max_predicted_ticks];
uint32 prediction_history_head = 0;
uint32 prediction_history_tail = 0;
uint32 prediction_history_head_tick_number = 0;

// ...
// on receiving a state packet, calculate where the client should be predicting to
uint32 ticks_to_predict = (uint32)(estimated_rtt_s / c_seconds_per_tick);
ticks_to_predict += 2; // add a little extra just in case
target_tick_number = received_tick_number + ticks_to_predict;

// ...
// prediction occurs during the game loop
// note: it's better to speed up/slow down the local player, rather than doing all the prediction in one go
while (tick_number < target_tick_number)
{
   prediction_history_state[prediction_history_tail] = local_player;
   prediction_history_input[prediction_history_tail] = current_player_input;
   prediction_history_tail = (prediction_history_tail + 1) % c_max_predicted_ticks;

   tick_player(&local_player, current_player_input);
   ++tick_number;
}
++target_tick_number;

// ...
// on receiving a state packet, check for and correct mispredictions
uint32 prediction_history_size = prediction_history_tail > prediction_history_head 
   ? prediction_history_tail - prediction_history_head 
   : c_max_predicted_ticks - (prediction_history_head - prediction_history_tail);
// it only occurs to me now that the code above will break if the buffer is ever full
// I'll fix that for the next installment!

while (prediction_history_size && 
   prediction_history_head_tick_number < received_tick_number)
{
   // discard this one, not needed
   ++prediction_history_head_tick_number;
   prediction_history_head = (prediction_history_head + 1) % c_max_predicted_ticks;
   --prediction_history_size;
}

if (prediction_history_size &&
   prediction_history_head_tick_number == received_tick_number)
{
   float32 dx = prediction_history_state[prediction_history_head].x - received_local_player_state.x;
   float32 dy = prediction_history_state[prediction_history_head].y - received_local_player_state.y;
   constexpr float32 c_max_error = 0.01f;
   constexpr float32 c_max_error_sq = c_max_error * c_max_error;
   float32 error_sq = (dx * dx) + (dy * dy);
   if (error_sq > c_max_error_sq)
   {
      log("[client]error of %f detected at tick %d, rewinding and replaying\n", sqrtf(error_sq), received_tick_number);

      local_player = received_local_player_state;
      uint32 i = prediction_history_head;
      while (true)
      {
         prediction_history_state[i] = *local_player;

         tick_player(local_player, &prediction_history_input[i]);

         i = (i + 1) % c_max_predicted_ticks;
         if (i == prediction_history_tail)
         {
            break;
         }
      }
   }
}
```
There's an important distinction to make with how you think of "state at tick n", and that is whether you consider it to be the state at the beginning or end of the tick. Either is valid, but you have to be consistent. I choose to think of it as the state at the beginning of the tick, and the input for that tick is the input which occurs during that tick. So State<sub>n</sub> + Input<sub>n</sub> = State<sub>n+1</sub>.

Correcting a misprediction introduces a problem - the client only receives *visual* state - that's what a ***snapshot*** means when talking about ***Snapshot Interpolation*** network models. In order to get back in sync with the server after a misprediction, the client really needs all the state which is relevant to the prediction. In our case we're predicting movement, so velocity is needed. In an FPS game we'd need data related to shooting - what gun is the player holding, how many bullets do they have, when was the gun last fired, etc.

We don't want to send this prediction-relevant data for all players though, bytes in each state packet from server to client is at a premium. Well, it isn't yet, but it will be. The solution is simple but involves a bit more work on the server - each player receives a bespoke state packet. In that packet they receive their own, fuller set of prediction-relevant state, followed by a series of visual state for all the other players.

{{<asset_img "/images/mpfps_05/img005.png">}}

The final piece of this particular puzzle, is input buffering on the server. When the client predicts forward, they do so such that their input for a given tick, arrives at the server just in time for it to be used.

{{<asset_img "/images/mpfps_05/img006.png">}}
<sub>How we wish packets behaved.</sub>

In reality, because of our best friend - jitter - sometimes an input arrives just in time, sometimes it's late, sometimes it never shows up, and sometimes it's 2 or 3 ticks early.

{{<asset_img "/images/mpfps_05/img007.png">}}
<sub>How packets actually behave.</sub>

To deal with this, the server has a buffer for each client, to store early inputs so that they can be used later. The size of this buffer is proportional to how much jitter we plan to allow. Looking at the code now, I've chosen a whole second, which is absurdly large in terms of time, but the memory usage is still negligible so I'll leave that as it is for now.

Note that in the past the client just sent an input packet to the server, and the server just took whatever the most recent input was that it had from the client, and used that when ticking the simulation. Now that we have prediction, we really care about what tick a particular input packet was meant for, so that will need to now be part of the input packet.

{{<asset_img "/images/mpfps_05/img008.png">}}
<sub>New input packet layout.</sub>

Initially I thought of implementing input buffering as a circular buffer of queues. Each future tick gets a queue of input packets, and when the server receives an input packet from a client, it adds it to the queue of inputs for that tick. When the server is about to tick the simulation, it runs over the input queue for that tick, for each item in the queue it copies the input data over the inputs used for that player for the previous tick. Any player who didn't get their input to the server in time, just carries on with their input from the last tick, it'll probably just result in a slight misprediction which will be corrected soon.
```cpp
// ...
// on receiving an input packet
input_queues[received_input_tick][tail] = Buffered_Input(received_input, from_player);

// ...
// before starting each tick
foreach (Buffered_Input buffered_input : input_queues[tick])
{
   player_inputs[buffered_input.player] = buffered_input.input;
}
```
The problem with this, is that it assumes we'll only get one input per-client. Packet duplication is rare, so it wouldn't cause a significant hit to CPU or memory usage to have the very occasional duplicate input packet. However, what if a hacker started sending malicious packets to clog up the server? We'd need a way of checking if we already have input from this particular client for this particular tick. If we're going to do that sort of checking, then we may as well just have an array per tick, of user inputs, with the player ID as the index. Well, we'll actually need two arrays, an input array, and an array of bools using the same index, and meaning "did I get anything from this client". This is because a "zero" input is entirely valid, the player could be holding no keys down at all, so we need this second array to allow for the concept of a "null" input, when we never got an input from this player at all.
```cpp
// ...
// on receiving an input packet
buffered_inputs[received_input_tick][from_player] = received_input;
buffered_inputs_test[received_input_tick][from_player] = true;

// ...
// before starting each tick
for (int i = 0; i < num_players; ++i)
{
   if (buffered_inputs_test[tick][i])
   {
      player_inputs[i] = buffered_inputs[tick][i];
   }
} 
```
That's it for now for client-side prediction, though I'll certainly return to this to tighten a few things up. Enjoy 2018 everyone.