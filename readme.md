Teensy Threading Library
===================================================

Teensy Threading Library implements preemptive threads for the Teensy
3.x platform from PJRC (https://www.pjrc.com/teensy/index.html). It supports
a native interface and a `std::thread` interface.

Simple example
------------------------------

```C++
#include <Threads.h>
volatile int count = 0;
void thread_func(int data){
  while(1) count += data;
}
void setup() {
  threads.addThread(thread_func, 1);
}
void loop() {
  Serial.println(count);
}
```

Or using std::thread

```C++
#include <Threads.h>
volatile int count = 0;
void thread_func(int data){
  while(1) count += data;
}
void setup() {
  std::thread th1(thread_func, 1);
  th1.detach();
}
void loop() {
  Serial.println(count);
}
```

Usage
-----------------------------

A global variable `threads` of `class Threads` is used to control the threading
action. The library is hard-coded to support 8 threads, but this may be changed
in the source code of Threads.cpp.

Each thread has it's own stack which can be allocated by the user or automatically
on the heap (see addThread() notes below).

Threads are created by `threads.addThread()` with parameters:

```
addThread(func, arg, stack_size, stack)

  func : function to call to perform thread work. This has the form of
         "void f(void *arg)" or "void f(int arg)" or "void f()"
  arg  : (optional) the "arg" passed to "func" when it starts.
  stack_size: (optional) the size of the thread stack [1]
  stack : (optional) pointer to a buffer to use as stack [2]
  Returns an ID number or -1 for failure

  [1] If stack_size is 0 or missing, then 1024 is used.
  [2] If stack is 0 or missing, then the buffer is allocated from the heap.
```

All threads start immediately and run until the function terminates with
a return.

If a thread ends because the function returns, then the thread will be reused
by a new function.

If a stack has been allocated by the library and not supplied by the caller, 
it will be freed when a new thread is added, not when it terminates.

The following functions of `class Threads` control threads. Items in all caps are
const members of `Threads` and are accessed as in `Threads::EMPTY`.

```C++
class Threads {
  // Get the id of the currently running thread
  int id();

  // Get the state; see class constants. Can be EMPTY, RUNNING, ENDED, SUSPENDED.
  int getState(int id);
  // Explicityly set a state. See getState(). Call with care.
  int setState(int id, int state);
  // Wait until thread ends, up to timeout_ms milliseconds. If ms is 0, wait
  // indefinitely.
  int wait(int id, unsigned int timeout_ms = 0);
  // Permanently stop a running thread. Thread will end on the next thread slice tick.
  int kill(int id);
  // Suspend a thread (on the next slice tick). Can be restarted with restart().
  int suspend(int id);
  // Restart a suspended thread.
  int restart(int id);
  // Set the slice length in ticks (1 tick = 1 millisecond)
  void setTimeSlice(int id, unsigned int ticks);

  // Yield current thread's remaining time slice to the next thread, causing
  // immedidate context switch
  void yield();
  // Wait for milliseconds using yield(), giving other slices your wait time
  void delay(int millisecond);

  // Start/restart threading system; returns previous state: STARTED, STOPPED, FIRST_RUN
  // can optionally pass the previous state to restore
  int start(int new_state = -1);
  // Stop threading system; returns previous state: STARTED, STOPPED, FIRST_RUN        
  int stop();
};
```

In addition, the Threads class has a member class for mutexes (or locks):

```C++
  class Mutex {
    int getState(); // get the lock state; 1=locked; 0=unlocked
    int lock(unsigned int timeout_ms = 0); // lock, optionally waiting up to timeout_ms milliseconds
    int try_lock(); // if lock available, get it and return 1; otherwise return 0
    int unlock();   // unlock if locked
  };
  class Scope {
      Scope(Mutex& m);
      ~Scope();
  };

  // example:
  Threads::Mutex mylock;
  mylock.lock();
  x = 1;
  mylock.unlock();
  if (y) {
    Threads::Scope m(mylock); // lock on creation
    x = 2;
  }                           // unlock at destruction
```

Usage notes
-----------------------------

The optimizer sometimes has strange side effects because it thinks variables
can't be changed within code. For example, if one thread modifies a variable
and another thread checks for it, the second thread's check may be optimized
away. Here is an example:

```
int state = 0;            // this should be "volatile int state"
void thread1() { state = processData(); }
void run() {
  while (state < 100) {   // this line changed to 'if'
    // do something
  }
}
```

In the code above, seeing that `state` is not changed in the loop, the 
optimizer will convert the `while (state<100)` into an `if` statement. Adding
`volatile` to the declaration of `int state` will usually help (as in
`volatile int state`).

Alternative std::thread interface
-----------------------------

The library also supports the construction of minimal `std::thread` as indicated 
in C++11. `std::thread` always allocates it's own stack of the default size. In
addition, a minimal `std::mutex` and `std::lock_guard` are also implemented.
See http://www.cplusplus.com/reference/thread/thread/

Example:

```C++
void run() {
  std::thread first(thread_func);
  first.detach();
}
```

The following members are implemented:

```C++
namespace std {
  class thread {
    bool joinable();
    void detach();
    void join();
    int get_id();
  }
  class mutex {
    void lock();
    bool try_lock();
    void unlock();
  };
  template <class Mutex> class lock_guard {
    lock_guard(Mutex& m);
  }
}
```

Notes on implementation
-----------------------------

Threads take turns on the CPU and are switched by the `context_switch()`
function, written in assembly. This function is called by the SysTick ISR. The
library overrides the default `systick_isr()` to accomplish switching. On the
Teensy by default, each tick is 1 millisecond long. By default, each thread
runs for 100 ticks, or 100 milliseconds, but this can be changed by
`setTimeSlice()`.

Much of the Teensy core software is thread-safe, but not all. When in doubt,
stop and restart threading in critical areas. In general, functions that share
global variables or state should not be called on different threads at the
same time. For example, don't use Serial in two different threads
simultaneously; it's ok to make calls on different threads at different times.

The code comments on the source code give some technical explanation of the 
context switch process:

```C
/*
 * context_switch() changes the context to a new thread. It follows this strategy:
 *
 * 1. Abort if called from within an interrupt
 * 2. Save registers r4-r11 to the current thread state (s0-s31 is using FPU)
 * 3. If not running on MSP, save PSP to the current thread state
 * 4. Get the next running thread state
 * 5. Restore r4-r11 from thread state (s0-s31 for FPU)
 * 6. Set MSP or PSP depending on state
 * 7. Switch MSP/PSP on return
 *
 * Notes:
 * - Cortex-M4 has two stack pointers, MSP and PSP, which I alternate. See the 
 *   CPU reference manual under the Exception Model section.
 * - I tried coding this in asm embedded in Threads.cpp but the compiler
 *   optimizations kept changing my code and removing lines so I have to use
 *   a separate assembly file. But if you try C++, make sure to declare the
 *   function "naked" so the stack pointer SP is not modified when called.
 *   This means you can't use local variables, which are stored in stack. 
 *   Also turn optimizations off using optimize("O0").
 * - Function is called from systick_isr (also naked) via a branch. Again, this is
 *   to preserve the stack and LR. I override the default systick_isr().
 * - Since Systick can be called from within another interrupt, I check
 *   for this and abort.
 * - Teensy uses MSP for it's main thread; I preserve that. Alternatively, I
 *   could have used PSP for all threads, including main, and reserve MSP for
 *   interrupts only. This would simplify the code slightly, but could introduce
 *   incompatabilities.
 * - If this interrupt is nested within another interrupt, all kinds of bad
 *   things can happen. This is especially true if usb_isr() is active. In theory
 *   we should be able to do a switch even within an interrupt, but in my
 *   tests, it would not work reliably.
 */
```

Todo
-----------------------------

1. Optimize assembler and other switching code.
2. Support unlimited threads.
3. Check for stack overflow during context_change() to aid in debugging; or
   have a stack that grows automatically if it gets close filling.
4. Fully implement the new C++11 std::thread or POSIX threads. 
   See http://www.cplusplus.com/reference/thread/thread/.
5. Time slices smaller than 1 millisecond for high responsiveness. By comparison, 
   typical Linux switches every 100 milliseconds.

Other
-----------------------------

This project came about because I was coding a Teensy application with
multiple things happening at the same time, whistfully reminiscing about
multithreading available in other OSs. I searched for threading tools, but
found nothing. This combined with boredom and abundant free time resulting in
complete overkill for the solution and thus this implementation of preemptive
threads.

Copyright 2017 by Fernando Trias. All rights reserved.
Revision 1, January 2017