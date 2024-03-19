+++
title = 'Client-Side Prediction With Physics In Unity'
date = 2018-04-14T00:00:00Z
draft = false
aliases = ["/blog/client-side-prediction-in-unity-2018"]
+++

TL;DR
I made a demo showing how to do client-side prediction with physics-based player movement in Unity - GitHub.

 
UPDATE: 
Aran Koning submitted a pull request which improves upon the code in this article using even newer and shinier Unity features. By putting the client and the server cubes in different physics scenes, there’s no need to selectively disable one cube or the other, which causes issues with determinism.

Introduction
Back in early 2012 I wrote a blog post about kind-of-but-not-really implementing client-side prediction of physics-based player movement in Unity. That nasty workaround that I described, is no longer necessary thanks to Physics.Simulate(). That old post is still one of the most popular posts on here, but with Unity today, that information is just sort of wrong. So here goes, the 2018 edition.

 
Client-Side What?
In a competitive multiplayer game, you want to avoid cheating as much as you can. Usually that means a server-authoritative network model, where clients send their input to the server, and the server turns that input into player movement, and sends a snapshot of the resulting player state back to the client. This causes a delay between pressing a key, and seeing the results, which feels unacceptably laggy for any kind of action game. Client-Side Prediction is a very common technique which hides this lag by predicting what the resulting movement will be, and showing it straight away. When the client receives the results from the server, it compares with what the client predicted, if they are different there has been a misprediction, and it corrects the predicted state.

The snapshots received from the server always arrive in the past, relative to the clients predicted state (i.e. if the round-trip from client to server and back, is 150ms, then each snapshot will be at least 150ms in the past). As a result of this, when the client needs to correct a misprediction, it must rewind to this point in the past, and then replay through all of the inputs in-between to get back up to where it was. If player movement in your game is physics-based, then that's why Physics.Simulate() is needed - to simulate multiple ticks in a single frame. If your player movement instead just uses Character Controllers (or a capsule cast, etc) then you should be fine without Physics.Simulate() - and I suspect performance would be better.

I'll be using Unity to attempt to recreate a networking demo I really liked from a while back, called "Zen of Networked Physics" by Glenn Fiedler. The player has a physics cube which they can apply forces to, pushing it around, and the demo simulates various network conditions like latency and packet loss.

 
Getting Started
The first thing to do is disable auto physics simulation. Though Physics.Simulate() allows us to tell the physics system when to simulate, by default it will simulate automatically based on the project's fixed delta time. So we'll want to disable that in Edit->Project Settings->Physics, untick "Auto Simulation".

To begin, we'll do a simple single-player implementation. Input is sampled (w, a, s, d, and space to jump), and this is turned in to simple forces which are applied to the Rigidbody with AddForce().

public class Logic : MonoBehaviour
{
   public GameObject player;

   private float timer;

   private void Start()
   {
      this.timer = 0.0f;
   }

   private void Update()
   {
      this.timer += Time.deltaTime;
      while (this.timer >= Time.fixedDeltaTime)
      {
         this.timer -= Time.fixedDeltaTime;

         Inputs inputs;
         inputs.up = Input.GetKey(KeyCode.W);
         inputs.down = Input.GetKey(KeyCode.S);
         inputs.left = Input.GetKey(KeyCode.A);
         inputs.right = Input.GetKey(KeyCode.D);
         inputs.jump = Input.GetKey(KeyCode.Space);

         this.AddForcesToPlayer(player.GetComponent<Rigidbody>(), inputs);

         Physics.Simulate(Time.fixedDeltaTime);
      }
   }
}

Player movement, no networking yet
 

Sending Input To The Server
Now we need to send the input to the server, where it will also execute this movement code, take a snapshot of the cube state, and send it back to the client.

// client
private void Update()
{
   this.timer += Time.deltaTime;
   while (this.timer >= Time.fixedDeltaTime)
   {
      this.timer -= Time.fixedDeltaTime;
      Inputs inputs = this.SampleInputs();

      InputMessage input_msg;
      input_msg.inputs = inputs;
      input_msg.tick_number = this.tick_number;
      this.SendToServer(input_msg);

      this.AddForcesToPlayer(player.GetComponent<Rigidbody>(), inputs);

      Physics.Simulate(Time.fixedDeltaTime);

      ++this.tick_number;
   }
}
Nothing special here, the only thing I will call out is the addition of the tick_number variable. This is here so that when the server sends snapshots of cube state back to the client, we can figure out what client tick that state corresponds with, so we can compare that state against what the client predicted (that'll come a bit later).

// server
private void Update()
{
   while (this.HasAvailableInputMessages())
   {
      InputMessage input_msg = this.GetInputMessage();

      Rigidbody rigidbody = player.GetComponent<Rigidbody>();
      this.AddForcesToPlayer(rigidbody, input_msg.inputs);

      Physics.Simulate(Time.fixedDeltaTime);

      StateMessage state_msg;
      state_msg.position = rigidbody.position;
      state_msg.rotation = rigidbody.rotation;
      state_msg.velocity = rigidbody.velocity;
      state_msg.angular_velocity = rigidbody.angularVelocity;
      state_msg.tick_number = input_msg.tick_number + 1;
      this.SendToClient(state_msg);
   }
}
The server just waits for input messages, whenever it gets one, it simulates a tick, simple. It then takes a snapshot of the resulting cube state, and sends it back to the client. You might notice that the tick_number in the state message is one greater than the tick_number in the input message. This is because I personally find it most intuitive to think of "the state of the player at tick 100" to be "the state of the player at the beginning of tick 100". Therefore the state for the player at tick 100, combined with the input from the player at tick 100, produces the new state for the player at tick 101.

Staten + Inputn = Staten+1
I'm not saying everyone should think of it that way, consistency is all that matters.

I should also mention that I'm not really sending these messages over a real socket, I'm faking it by putting them into queues, and simulating some latency and packet loss. The scene contains two physics cubes - one for the client, and one for the server. When updating the client cube, I disable the server cube GameObject, and vice versa.

I'm not however, simulating any network jitter or out-of-order packet delivery, so that's why I'm making an assumption that every input message which arrives is newer than the last. The reason for this fakery is mainly that it allows us to run the "client" and "server" in the same instance of Unity very easily, so we can composite the client and server cubes into the same scene.

You might also notice that if an input message gets dropped and doesn't reach the server, that results in the server simulating a smaller number of ticks than the client, and will therefore produce different state. That's true, but even if we were to simulate the gaps, the inputs might be wrong anyway - also resulting in different state. This is an issue we'll deal with later.

I'll also point out that this example only has one client which simplifies things, if you had multiple clients then you'd need to either a) make sure you only had one player cube enabled on the server when calling Physics.Simulate() or b) if the server has received input for multiple cubes, simulate them all together.


75ms latency (150ms round-trip)
0% packet loss
Yellow cube - server player
Blue cube - last received snapshot on client

Looks pretty good so far, but I've been a bit selective with what I captured in order to hide quite a big problem.

 
Determinism Fail
Have a look at this instead:


Uh oh..

That was captured with no packet loss, yet the simulations still diverge with exactly the same inputs. I'm not exactly sure why this happens - PhysX is supposed to be reasonably deterministic so I do find it surprising that the simulations diverge so often. It could be related to how I'm toggling the cube GameObjects on and off all the time, and therefore perhaps this would be much less of a problem with two separate instances of Unity. It could also be a bug, if you can spot it in the code on GitHub then let me know.

In any case, mispredictions are a fact of life with client-side prediction, so now lets deal with them.

 
Can I Get A Rewind?
The process is pretty simple - when the client predicts movement, it stores a buffer of state (position and rotation) and inputs. When a state message is received from the server, it compares the received state with the predicted state in the buffer. If they're different by too large a margin, then override the client cube state in the past, and then re-simulate all of the ticks in-between.

// client
private ClientState[] client_state_buffer = new ClientState[1024];
private Inputs[] client_input_buffer = new Inputs[1024];

private void Update()
{
   this.timer += Time.deltaTime;
   while (this.timer >= Time.fixedDeltaTime)
   {
      this.timer -= Time.fixedDeltaTime;
      Inputs inputs = this.SampleInputs();

      InputMessage input_msg;
      input_msg.inputs = inputs;
      input_msg.tick_number = this.tick_number;
      this.SendToServer(input_msg);

      uint buffer_slot = this.tick_number % 1024;
      this.client_input_buffer[buffer_slot] = inputs;
      this.client_state_buffer[buffer_slot].position = rigidbody.position;
      this.client_state_buffer[buffer_slot].rotation = rigidbody.rotation;

      this.AddForcesToPlayer(player.GetComponent<Rigidbody>(), inputs);

      Physics.Simulate(Time.fixedDeltaTime);

      ++this.tick_number;
   }

   while (this.HasAvailableStateMessage())
   {
      StateMessage state_msg = this.GetStateMessage();

      uint buffer_slot = state_msg.tick_number % c_client_buffer_size;
      Vector3 position_error = state_msg.position - this.client_state_buffer[buffer_slot].position;

      if (position_error.sqrMagnitude > 0.0000001f)
      {
         // rewind & replay
         Rigidbody player_rigidbody = player.GetComponent<Rigidbody>();
         player_rigidbody.position = state_msg.position;
         player_rigidbody.rotation = state_msg.rotation;
         player_rigidbody.velocity = state_msg.velocity;
         player_rigidbody.angularVelocity = state_msg.angular_velocity;

         uint rewind_tick_number = state_msg.tick_number;
         while (rewind_tick_number < this.tick_number)
         {
            buffer_slot = rewind_tick_number % c_client_buffer_size;
            this.client_input_buffer[buffer_slot] = inputs;
            this.client_state_buffer[buffer_slot].position = player_rigidbody.position;
            this.client_state_buffer[buffer_slot].rotation = player_rigidbody.rotation;

            this.AddForcesToPlayer(player_rigidbody, inputs);

            Physics.Simulate(Time.fixedDeltaTime);

            ++rewind_tick_number;
         }
      }
   }
}
The buffered inputs and state are stored in a really simple circular buffer, just using the tick id as the index. I've chosen a physics tick rate of 64hz, so with a buffer of 1024 elements that gives us 16 seconds of space, considerably more than we'd ever really need.


Corrections enabled!

 

Sending Redundant Inputs
Input messages are typically very small - buttons held can be combined into a bitfield which takes up a few bytes depending on the game. We also have a tick number in ours, this is 4 bytes, but we could easily compress this by using a wrapping 8 bit value instead (0-255 is perhaps too small a range, we could increase this to 9 or 10 bits to be safe). In any case, the small relative size of these messages means we can afford to send many inputs in each message (in case any of those previous inputs have been dropped). How far back to go? Well, if the client knows the tick number of the last state message it got from the server, there's no point going back further than that tick. You could also put a limit on the number of redundant inputs that get sent by the client, I haven't done that for this demo, but it's probably a good idea for actual shipping code.

while (this.HasAvailableStateMessage())
{
   StateMessage state_msg = this.GetStateMessage();

   this.client_last_received_state_tick = state_msg.tick_number;
This is a simple modification, the client just makes a note of the tick number of the latest state message it receives.

Inputs inputs = this.SampleInputs();

InputMessage input_msg;
input_msg.start_tick_number = this.client_last_received_state_tick;
input_msg.inputs = new List<Inputs>();
for (uint tick = this.client_last_received_state_tick; tick <= this.tick_number; ++tick)
{
   input_msg.inputs.Add(this.client_input_buffer[tick % 1024]);
}
this.SendToServer(input_msg);
The input message sent by the client now contains a list of inputs, rather than just one. The tick number part of that message now takes on a new meaning, as the tick number of the first input in that list.

while (this.HasAvailableInputMessages())
{
   InputMessage input_msg = this.GetInputMessage();

   // message contains an array of inputs, calculate what tick the final one is
   uint max_tick = input_msg.start_tick_number + (uint)input_msg.inputs.Count - 1;

   // if that tick is greater than or equal to the current tick we're on, then it
   // has inputs which are new
   if (max_tick >= server_tick_number)
   {
      // there may be some inputs in the array that we've already had,
      // so figure out where to start
      uint start_i = server_tick_number > input_msg.start_tick_number ? (server_tick_number - input_msg.start_tick_number) : 0;

      // run through all relevant inputs, and step player forward
      Rigidbody rigidbody = player.GetComponent<Rigidbody>();
      for (int i = (int)start_i; i < input_msg.inputs.Count; ++i)
      {
         this.AddForcesToPlayer(rigidbody, input_msg.inputs[i]);

         Physics.Simulate(Time.fixedDeltaTime);
      }
      
      server_tick_number = max_tick + 1;
   }
}
When the server receives an input message, it knows the tick number of the first input, and the number of inputs in the message. Therefore, it can calculate the tick of the final input in the message. If this final tick is greater than or equal to the server tick number, then it knows this message contains at least one input that it hasn't seen before. If that's the case, it simulates all the new inputs.

You might notice that if we limited the number of redundant inputs in the input message, it would be possible that if enough input messages were dropped, we'd end up with a simulation gap between the server and client. Meaning, the server might simulate tick 100, send out the state message for the beginning of tick 101, and then receive an input message which starts at 105. In the above code the server will jump to 105, it won't attempt to simulate the ticks in-between using the last known inputs. Whether or not you choose to do this is really up to you, and depends on the game you're making. Personally, I'd rather not have the server guess, and send a player potentially coasting across the map because of bad network conditions. I think it's actually better to root the player to the spot for that period of time until their connection recovers.

In the original "Zen of Networked Physics" demo there was also a feature where the client would send "important moves", which means it would send redundant inputs only if they were different from the input which came before. This could also be referred to as delta-compression of the inputs, and would result in even smaller input messages. I haven't done it here though, as this demo doesn't include any bandwidth optimisation.


Before sending redundant inputs:
With 25% packet loss
The cube's movement is slow and jerky, it keeps rubber-banding backwards.


After sending redundant inputs:
With 25% packet loss
There are still jerky corrections, but the cube moves at the appropriate speed.

 

Varying Snapshot Rate
The rate at which the server sends snapshots to the client is variable in this demo. With a lower rate, it will take longer on average for the client to receive a correction from the server. So when the client mispredicts, it will diverge more before getting a state message, resulting in a more noticeable correction. With a higher snapshot rate, the loss of a packet is a lot less significant, as the client doens't have to wait as long before the next snapshot turns up.


64hz snapshot rate


16hz snapshot rate


2hz snapshot rate

It's clear that a higher snapshot rate is better, so I'd say, send them as fast as you can. It just depends on how much traffic that adds up to, the cost of that traffic if you're running dedicated servers, the computational cost for the servers, and so on.

 
Correction Smoothing
We get mispredictions and jarring corrections happening - more than we'd like. Without proper access to the Unity/PhysX integration, my abilities to debug the mispredictions are pretty nonexistent. I've said this before but I'll say it again - if you can spot something I'm doing wrong with the physics, please let me know.

I've worked around this problem by papering over the cracks with good old fashioned smoothing! When a correction occurs, the client just smoothes the player position and rotation towards the correct state over multiple frames. The actual physics cube is corrected immediately (and is invisible), but there is a second cube for display purposes only, which allows for the smoothing.

Vector3 position_error = state_msg.position - predicted_state.position;
float rotation_error = 1.f - Quaternion.Dot(state_msg.rotation, predicted_state.rotation);

if (position_error.sqrMagnitude > 0.0000001f ||
   rotation_error > 0.00001f)
{
   Rigidbody player_rigidbody = player.GetComponent<Rigidbody>();

   // capture the current predicted pos for smoothing
   Vector3 prev_pos = player_rigidbody.position + this.client_pos_error;
   Quaternion prev_rot = player_rigidbody.rotation * this.client_rot_error;

   // rewind & replay
   player_rigidbody.position = state_msg.position;
   player_rigidbody.rotation = state_msg.rotation;
   player_rigidbody.velocity = state_msg.velocity;
   player_rigidbody.angularVelocity = state_msg.angular_velocity;

   uint rewind_tick_number = state_msg.tick_number;
   while (rewind_tick_number < this.tick_number)
   {
      buffer_slot = rewind_tick_number % c_client_buffer_size;
      this.client_input_buffer[buffer_slot] = inputs;
      this.client_state_buffer[buffer_slot].position = player_rigidbody.position;
      this.client_state_buffer[buffer_slot].rotation = player_rigidbody.rotation;

      this.AddForcesToPlayer(player_rigidbody, inputs);

      Physics.Simulate(Time.fixedDeltaTime);

      ++rewind_tick_number;
   }

   // if more than 2ms apart, just snap
   if ((prev_pos - player_rigidbody.position).sqrMagnitude >= 4.0f)
   {
      this.client_pos_error = Vector3.zero;
      this.client_rot_error = Quaternion.identity;
   }
   else
   {
      this.client_pos_error = prev_pos - player_rigidbody.position;
      this.client_rot_error = Quaternion.Inverse(player_rigidbody.rotation) * prev_rot;
   }
}
When a misprediction occurs, the client keeps track of the difference in position/rotation after the correction. If the total distance of position correction is greater than 2 metres then it just snaps - that'll look pretty bad being smoothed anyway, best just get back on track ASAP.

this.client_pos_error *= 0.9f;
this.client_rot_error = Quaternion.Slerp(this.client_rot_error, Quaternion.identity, 0.1f);

this.smoothed_client_player.transform.position = player_rigidbody.position + this.client_pos_error;
this.smoothed_client_player.transform.rotation = player_rigidbody.rotation * this.client_rot_error;
Each frame the client lerps/slerps towards the correct position/rotation by 10%, the usual exponential moving average approach. This isn't frame rate independent, but it will do for the purposes of this demo.


250ms latency
10% packet loss
Without smoothing, corrections are very noticeable


250ms latency
10% packet loss
With smoothing, corrections are much harder to spot

The end result works pretty well - I'd like to try making a version of this that actually sends packets rather than all the fakery, but it's at least a proof of concept for doing client-side prediction, of proper physics objects, with Unity, without having to resort to physics plugins and the like.