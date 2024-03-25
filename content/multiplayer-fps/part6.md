+++
title = 'Making a Multiplayer FPS in C++ Part 6: Cleanup'
date = 2018-01-11T00:00:00Z
draft = false
aliases = ["/blog/multiplayer-fps-part-6"]
includeToc = false
+++

In the previous part of this series, we added client-side prediction. At this point the codebase was getting a bit creaky with all the "todo" comments in the code, so I took a bit of time to address as many of them as possible (browse the code for this part [here](https://github.com/spectre1989/odin/tree/6d97c43ddfc0c21d99bb6cb63e332ca28de69fe5)).

### A Circular Buffer Utility
Circular buffers are a great way to avoid dynamic allocation/deallocation, you can usually calculate how many items you'd need at maximum and just do a single allocation. There were a few places where I was using one - input buffering on the server, client recording of inputs and predicted results for client-side prediction, and the packet buffers used for faking lag on the client. It would be nice to compress the code common to all three, into some kind of generic utility. The classic template container approach would be something like:
```cpp
template <typename T>
class Circular_Buffer
{
   T* m_items;
   uint32 m_capacity;
   uint32 m_head;
   uint32 m_size;
   
   void push(const T& item);
   void pop(T& item);
};
```
This has a problem, take for example the client-side prediction circular buffer which stores predicted state for the local player, and the user inputs used for each tick. The state, and input data, are two separate arrays:
```cpp
constexpr uint32 c_max_predicted_ticks = c_prediction_history_capacity * 2;
Player_State prediction_history_state[c_prediction_history_capacity];
Player_Input prediction_history_input[c_prediction_history_capacity];
uint32 prediction_history_head = 0;
uint32 prediction_history_tail = 0;
```
So to use the template approach we'd have to combine state and input into a struct:
```cpp
struct Predicted_Data
{
   Player_State state;
   Player_Input input;
};
Circular_Buffer<Predicted_Data> m_prediction_history;
```
This forces us to arrange circular buffer data as an ***array of structs***, rather than ***struct of arrays***. In future when we start worrying about packet loss, the client will include redundant input data in each input packet (i.e. it will send a packet with the input for tick `n`, along with the input for ticks `(n-1)`, `(n-2)`, etc, in case some of those previous packets didn't arrive). With an array of structs, when we iterate back through the player input we'd also pull the predicted player state in to the CPU cache even though we don't need to read it.

So to have a ***struct of arrays*** we could just have two circular buffer objects - `Circular_Buffer<Player_State>` and `Circular_Buffer<Player_Input>`. That's a bit messy, and means any logic done in the `Circular_Buffer` (e.g. checking if the buffer is full, or empty, calculating the new head index, etc) gets duplicated. It's also possible they could get out of sync if a function is called on one, but accidentally not called on the other.

Another option is some completely bananas template metaprogramming, which is hard to read, possibly bloats compile times, and ultimately still has ugly usage code. 

Though really, all the circular buffer needs to do is maintain the tail/write index, and head/read index, there's no actual need to contain the items as well. So what I actually did, was something like this:
```cpp
struct Circular_Index
{
   uint32 head;
   uint32 size;
   uint32 capacity;
};
void circular_index_create(Circular_Index* index, uint32 capacity);
void circular_index_push(Circular_Index* index);
void circular_index_pop(Circular_Index* index);
uint32 circular_index_tail(Circular_Index* index);
```
So the prediction history looks something like:
```cpp
constexpr uint32 c_prediction_history_capacity = c_ticks_per_second * 2;
Player_State* prediction_history_state;
Player_Input* prediction_history_input;
Circular_Index prediction_history_index;
circular_index_create(&prediction_history_index, c_prediction_history_capacity);
```
As a side-note, it occurs to me now that actually in the case of prediction history or input buffering, there's actually no need to pop the old items as they become irrelevant, we _could_ just overwrite the oldest ones. The only case where we _do_ care is the fake lag packet buffering. For example when sending a packet, it is held in a buffer for a bit and then sent, which simulates lag. In the case that this buffer is full, we log a warning, and then send the oldest packet early.

### Custom Allocators
I had a bunch of big-ish arrays on the stack, and some stuff in the graphics layer allocated with `new` and `delete`. Taking control of memory allocation is an opportunity for big performance wins, as well as making it easier to track memory usage and so on. For now I only need two allocators, ***permanent*** and ***temporary***. The former is for stuff which will be needed for the entire lifetime of the game, and the latter is just a scratch space.

Both of these are ***linear allocators***, which means we have a pointer to a block of free memory, and whenever an allocation is made we bump the pointer along. The underlying free memory block is allocated once up-front when the game starts - using `VirtualAlloc` to do this allows us to specify the base address of the allocated memory, this is particularly useful for debugging as (if allocations are deterministic) it can allow for memory breakpoints to work across multiple runs of the game.
```cpp
struct Memory_Allocator
{
   uint8* memory;
   uint8* next;
   uint64 bytes_remaining;
};

uint8* memory_allocator_alloc(Memory_Allocator* allocator, uint64 size)
{
   assert(allocator->bytes_remaining >= size);
   uint8* mem = allocator->next;
   allocator->next += size;
   return mem;
}
```
This doesn't mean we can never free anything, but it means we have to free memory in the reverse-order that it was allocated, or (as we'll likely do with the temporary allocator) free the entire thing. In future we'll probably want different allocators for different systems (e.g. graphics, gameplay, etc) but I try not to do things until they're actually needed.

It's important to mention that I'm not yet worrying about the alignment of allocated memory. It's generally good practice to naturally align data (meaning that for example a 4-byte integer is stored on a 4-byte boundary). Some platforms, like arm, will actually crash if you don't. On the other hand, x86 will do it, but it will incur a performance penalty. This penalty is supposedly dwindling with modern processors, so I'm planning to wait until there's a fair amount of game going on, and then turn on alignment and see how much difference it actually makes.

There are tons of different ways to implement an allocator, I try to use the simplest that will do what I need - simpler often means faster. One of the trickiest to implement is a general-purpose allocator (a replacement for `malloc`/`free`), but if possible I want to avoid _ever_ using one. A `malloc` style allocator would have no knowledge about the lifetime of memory being requested, or the relationships between allocations. We can take advantage of our knowledge of this stuff to carefully choose where and how allocations happen, improve performance, and reduce memory fragmentation. There may be a case where we _really_ need one, but I have yet to encounter one.

### Globals
At some point you're (probably?) going to need some globals. You can use a singleton and _pretend_ you don't have any globals, but they're still globals. I like to put mine in a struct so at least they're in one place, and it's easier to see at a glance if they're getting out of hand.

The aforementioned permanent and temporary memory allocators live in there, along with some stuff to do with timing. Lastly there's a function pointer to a logging function, this was something I needed because the server (being a console application) needs to send logging to stdout, whereas the client sends logging to `OutputDebugStringA` (the Visual Studio debug output window). Code used by both the server and client (e.g the socket code) doesn't need to figure out which to call, the function pointer takes care of it.
```cpp
struct Globals
{
   Memory_Allocator permanent_allocator;
   Memory_Allocator temp_allocator;
   Log_Function* log_function;
   LARGE_INTEGER clock_frequency;
   bool32 sleep_granularity_was_set;
};
```
### Socket Buffers
When packets are sent via a socket, it initially will be placed in an internal buffer which is managed by the underlying implementation. Received packets are also placed into a buffer. It's possible to find out how big these are with `getsockopt`, in my case the default is 64KiB. So if we assume the server is sending 1KiB to each connected client per tick, that means potentially being limited to a maximum of 64 players (actually less, because packet headers take up a bit of space).

I've upped these buffers to be 1MiB which I expect will be sufficient, but it's difficult to know at this early stage. Socket errors are logged, so if the buffers are exceeded then we'll be able to see. It isn't the end of the world if that does happen on occasion - to the game this will look the same as a burst of packet loss.

### Non-Unity Builds
Although I like to use Unity builds (bundling multiple cpp files into a single translation unit), plenty of people don't. So it'd probably be a good idea if it was possible to compile the project as individual cpp files too. I probably won't remember to do this all the time, but I'll check and fix issues periodically.