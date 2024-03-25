+++
title = 'Making a Multiplayer FPS in C++ Part 8: Client-Side Prediction Revisited'
date = 2019-06-07T00:00:00Z
draft = false
aliases = ["/blog/multiplayer-fps-part-8"]
includeToc = false
+++

I was lucky enough to go to GDC 2018, and while there I had the pleasure of sitting down with Glenn Fiedler in a really rather swanky tea place. For those not already aware, Glenn is pretty much "the netcode guy". If you’re looking on the web for information about netcode and network models, then you’ll almost certainly read [some of his articles](https://gafferongames.com/). They contain a lot more detail than most about the actual implementation of the ideas, you can watch all the GDC videos about networking that you like, at some point it’s great to see some actual code. Even the [Barnier paper from 2001](https://developer.valvesoftware.com/wiki/Latency_Compensating_Methods_in_Client/Server_In-game_Protocol_Design_and_Optimization) was pretty high level really - you could download the Source Engine source code, but it’ll take a while to get a foothold in a new large codebase. In 2009 when I was in my first job in the industry as a lowly intern, I was tasked with making an MMO client. I’d never even opened a socket at that point, in my googling I found Glenn’s site, I soon got pretty obsessed with netcode, and the rest is history.

The reason for meeting Glenn was to find out about his new company [Network Next](https://networknext.com/) - which is also well worth checking out, the implications for multiplayer games are huge. The conversation turned to netcode, and he rapidly ran through various snapshot-interpolation related things that he thought I might need to know. In amongst that stuff, he showed me that I’d misunderstood something about inputs and client-side prediction.

### I Was Wrong
The mistake I’d made (and you can see this if you read [part 5](../part5)) was that I thought the clients and server should run at the same tick-rate, and that all client inputs should be simulated in lock-step. For example, if the server was about to simulate tick 100, it would hopefully have received inputs from all connected clients for tick 100, and it would simulate all the players for tick 100, then serialise and send out a snapshot back to all clients. If the server had not received an input from one or more clients for tick 100, then it would just copy whatever input it had from a previous tick. If an input arrived late - too bad, the offending client would probably be getting a correction.

This was wrong. The clients run at whatever speed they can or want to, and send their frame deltas in their input packet, along with the button presses and so on. When the server receives the input packet, it can just carry out the input straight away. So there kind of is no ***tick 100***, the server tick rate just governs how frequently non-player game state (e.g. NPCs) gets updated, but for players it’s effectively more a sample rate. 

The beauty of this, is that there’s no such thing as a late input. Even a dropped packet wouldn’t matter most of the time - in each input packet you also provide a bunch of redundant inputs from previous ticks (input data is small, so you can cram in a lot), so if some packets go missing, the server will pick up the lost moves from a later packet. The other advantage is that the client is now no longer forced to tick at the same rate as the server. If the server is ticking at 60, but a player has a low power machine which can only manage 30, they can still play. On the other hand, if their machine is a beast, they can run at 120 and feel the benefit of those extra frames.

The downsides though - firstly, a player running at a higher frame rate will put more strain on the server. A player running at 120 ticks per second will incur 4 times the input processing cost on the server than a player running at 30. For this reason it's a good idea to cap the client simulation frame rate to something sensible. If a player is running at a slower rate than the server then (I think) a solution to this might be to interpolate the player position on the server. So if the server is ticking at 60hz, and the client at 20hz, the server would simulate that client’s inputs, but interpolate the resulting position over 3 ticks.

Second - speed hacks are back. When the simulation is ticked in lock-step it pretty much neutralises speed hacking. However, I think you can solve this problem at the same time as dealing with jitter. As I’ve mentioned in [part 5](../part5) network jitter can be greater than you might expect, 1-2 frames at 60hz is a good rule of thumb. Jittering of the input packets arriving at the server will cause the player’s positions to jitter on the server, and then this jittery movement will be observed by players when snapshots make their way to the clients. The solution to this is to measure the jitter for each connected client, and buffer their input packets accordingly. This adds additional latency between clients predicting their actions locally, and having those actions confirmed on the server. By buffering on an individual basis, only the players with jittery connections will be affected. The server can attempt to eat through this buffer at a steady rate. It should be able to speed up and slow down slightly to adapt to changing latency/jitter, but not so much that a speed-hacking client could get much benefit, their buffer would just fill up.

### Implementation
(Code at this commit on GitHub [here](https://github.com/spectre1989/odin/tree/9ff85e2a07b590af4b9a4463319dcf762958b2af))

So I’ll need to make some changes, though I’m still going to assume no jitter, packet loss, or packet duplication for now. This means that for now I won’t be buffering the inputs on the server for jitter/hacking, and I won’t bother sending redundant inputs in each input packet.

Previously (in my incorrect implementation) my input packets contained the client input and tick number, but also a timestamp. The latter would then be echoed back to the client in the next state packet. This was used to estimate the round-trip time to the server, so that the client could run far enough ahead that its inputs wouldn’t be late.
```cpp
send_input_msg(
    uint32 client_slot,
    uint8 input,
    uint32 timestamp,
    uint32 tick_number);
```
Now that I know clients don’t "run ahead", I’ll just need to add the client delta time to this, and lose the timestamp. Another thing to point out here is that the client provides a tick number in this packet, that was because the client and server ran at the same tick rate, but that is now not necessarily the case. However, I can’t lose this field from the packet, it will still be needed for correcting client-side prediction when mispredictions occur. When the server tells the client about the game state, the client uses this to find out if their prediction of the local player was correct. The client needs some way of matching that state up with it’s prediction history buffer. So this tick number becomes a local tick number of sorts, which just increases with each client tick - I’ll call this `prediction_id` so it’s super obvious what it’s for.
```cpp
send_input_msg(
    uint32 client_slot,
    float32 dt,
    uint8 input, // bitfield of buttons currently held
    uint32 prediction_id);
```
The client stores its prediction history in a circular buffer. The index into the buffer to use is `prediction_id % buffer_capacity`. If this buffer has a size which is a power of two, then this modulo operation can be done faster with `prediction_id & (buffer_capacity - 1)`. With a buffer capacity of 512, assuming we cap client frame rate at 120hz, that’s a bit more than 4 seconds worth of buffer, which should be plenty.
```cpp
struct Predicted_Move
{
    float32 dt;
    Input input;
};

constexpr uint32 c_buffer_size = 512;
constexpr uint32 c_buffer_mask = c_buffer_size - 1;

Predicted_Move predicted_move[c_buffer_size];
Player_State predicted_move_result[c_buffer_size];

Player_State player_state;
uint32 current_prediction_id;
```
The client loop will look something like this:
```cpp
while (true)
{
    // send input to server
    send_input_msg(client_slot, dt, input, current_prediction_id);

    // update player state
    tick_player(&player_state, dt, input);

    uint32 buffer_index = current_prediction_id & c_buffer_mask;

    // record dt and input in the buffer for this tick, so it can be replayed if necessary
    predicted_move[buffer_index].dt = dt;
    predicted_move[buffer_index].input = input;
    
    // record resulting player state so we can tell when the client has gone out of sync with the server
    predicted_move_result[buffer_index] = player_state;
    
    ++current_prediction_id;
}
```
So what the buffers are storing is - at prediction id `x`, dt was `a`, input was `b`, and the resulting player state was `c`.

The server loop will look like this:
```cpp
while (true)
{
    while (Msg* msg = get_next_message())
    {
        switch (msg->type)
        {
            // ...
            
            case Input:
            {
                tick_player(&player_state[msg->slot], msg->dt, msg->input);
                player_prediction_id[slot] = msg->prediction_id;
            }
            break:
            
            // ...
        }
    }
    
    for (int32 slot = 0; slot < c_max_clients; ++slot)
    {
        if (client[slot])
        {
            send_state_msg(slot, player_prediction_id[slot], &player_state[slot], &other_game_state);
        }
    }
}
```
Then when the client receives a state message from the server:
```cpp
uint32 buffer_index = msg->prediction_id & c_buffer_mask;
Vec_3f error = predicted_move_result[buffer_index].position - msg->player_state.position;
if (error.length_squared() > c_max_prediction_error_squared)
{
    player_state = msg->player_state;
    
    for (uint32 replaying_prediction_id = msg->prediction_id + 1;
        replaying_prediction_id < current_prediction_id;
        ++replaying_prediction_id)
    {
        buffer_index = replaying_prediction_id & c_buffer_mask;
        
        tick_player(&player_state, predicted_move[buffer_index].dt, predicted_move[buffer_index].input);
        
        predicted_move_result[buffer_index] = player_state;
    }
}
```
#### A Change of Perspective
So far my player has been a weird sort of third-person cube car thing. It’s about time to actually switch to a first-person controller as the project demands.

{{<youtube "ANEuTkpDmeg">}}

This requires some changes to the networking of inputs, as we now have to incorporate mouse input. Along with sending the buttons held down, we also need the pitch/yaw of the player. Actually - we don’t yet _need_ the pitch as it’s not used for movement, but we will need it for shooting, so I’ll just add it now.
```cpp
send_input_msg(
    uint32 client_slot,
    float32 dt,
    uint8 buttons, // bitfield of buttons currently held
    float32 pitch,
    float32 yaw,
    uint32 prediction_id);
```
You could be forgiven for thinking that for preventing cheating you should not send raw pitch and yaw, and instead send deltas from the previous pitch and yaw. This is a waste of time, anyone aim-botting etc will be able to figure out sending deltas, while you run the risk of legitimate players having their pitch/yaw get out of sync with the server because of a burst of packet loss or a bug.

That’s about it for the re-working of client-side prediction. It’s still not finished of course, we need to deal with packet loss, jitter, potential malicious packets, cheating, and probably more. It’s enough for now to move forward.