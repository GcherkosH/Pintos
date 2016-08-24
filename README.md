# Pintos

			+--------------------+
			| |
			| 	THREADS      |
			|   DESIGN DOCUMENT  |
			+--------------------+
				   
Gebrecherkos Haylay Alemayoh <cherkos.hailay (at) gmail.com>


			     ALARM CLOCK
			     ===========

---- DATA STRUCTURES ----

>> A1: Copy here the declaration of each new or changed `struct' or
>> `struct' member, global or static variable, `typedef', or
>> enumeration.  Identify the purpose of each in 25 words or less.

In struct thread of thread.h
-----------------------------

int64_t sleep_time;       
This keeps track of the time a thread is sleeping


In devices/timers.c 
---------------------

static struct list sleep_list;   
This is an ordered list of sleeping threads.

bool thread_wake_order(const struct list_elem* one,const struct list_elem* two,void* aux);
This function is used to order sleeping threads as they are being  entered into the sleep_list structure.

In function void timer_init (void) 
list_init(&sleep_list);      initialises the sleep_list



---- ALGORITHMS ----

>> A2: Briefly describe what happens in a call to timer_sleep(),
>> including the effects of the timer interrupt handler.

When a call to timer_sleep() is made, a check on valid ticks is done. 
If(tick=<0), timer_sleep() returns immediately.
If the ticks are valid, interrupts are disabled.
The current thread is added in ordered way  to the sleep_list (front thread has the smallest remaining sleep_time).
The thread is then blocked and interrupts restored to original state.

In the timer_interrupt() handle, 
The program checks if the sleep_list is not empty
If true, It then checks only the first element of the list.
If its sleep_time <= ticks, it is removed from the list and unblocked.

This process in the timer_interrupt() is repeated until there are no threads in the sleep_list.



>> A3: What steps are taken to minimize the amount of time spent in
>> the timer interrupt handler?

The blocked threads are kept in a sleep_list sorted in increasing sleep_time.
In the interupt handler implemetation, the handler checks only the first thread of the list.
It does not have to iterate through all blocked threads, everytime.


---- SYNCHRONIZATION ----

>> A4: How are race conditions avoided when multiple threads call
>> timer_sleep() simultaneously?

Race conditions are avoided by disabling the interrupts when blocking the current thread (sending thread to sleep) 
which makes the blocking nonpreemptive. 

>> A5: How are race conditions avoided when a timer interrupt occurs
>> during a call to timer_sleep()?

Here, race conditions are also avoided by disabling the interrupts.


---- RATIONALE ----

>> A6: Why did you choose this design?  In what ways is it superior to
>> another design you considered?

This design uses list formats from the list.h file.
When handling blocked threads, this design is superior to using the thread_foreach construct 
because it does not have to iterate through all blocked threads, everytime which makes the code 
run faster and more efficient.


			    BATCH SCHEDULING
			    ================
---- DATA STRUCTURES ----

>> A1: Copy here the declaration of each new or changed `struct' or
>> `struct' member, global or static variable, semaphore, `typedef', or
>> enumeration.  Identify the purpose of each in 25 words or less.

The definitions below were used.
#define BUS_CAPACITY 3   defines BUS_CAPACITY to be 3 tasks 
#define SENDER 0
#define RECEIVER 1
#define NORMAL 0
#define HIGH 1


int curr_direction = SENDER;  /* initialises the current direction of data transfer   */

struct semaphore bus_slots;  /*semaphore for BUS_CAPACITY slots*/
struct semaphore send_pri_task_sema; /* semaphore for priority tasks waiting to send data*/
struct semaphore rcv_pri_task_sema; /*semaphore for priority tasks waiting to recieve data*/
struct semaphore send_norm_task_sema; /*semaphore for tasks waiting to send data*/
struct semaphore rcv_norm_task_sema; /*semaphore for tasks waiting to receive data*/

struct lock bus_mutex;   /*ensures mutual exclusion to access the shared bus resource*/


---- SYNCHRONIZATION ----

>> C1: How does your solution guarantee that no more that 3 tasks
>> are using the bus in the same direction?

This is guaranteed by initialising the bus_slots counting semaphore to 3 in the statement
    sema_init(&bus_slots,BUS_CAPACITY); 



>> C2: What prevents tasks from opposite directions from using the
>> bus simultaneously?

When a thread enters the critical section, the first test is to check whether the 
bus is free or the task is moving in the current direction.

When tasks in opposite direction arrive, the if condition [curr_direction ==  task.direction] is false and also 
since there is traffic on the bus already, the current [bus_slots.value == BUS_CAPACITY] is also false. 
Therefore the if condition returns FALSE and the task does not access the bus and hence releases the lock.



>> C3: How does your solution grant priority to high priority tasks over
>> the waiting tasks in the same direction?

The second condition "(task.priority == HIGH || (send_pri_task_sema.value==0 && rcv_pri_task_sema.value==0))" 
when accessing a bus slot checks if the priority of the task is HIGH or there are no HIGH priority tasks in 
either the send direction or the receiver direction. 
If the priority is high in a given direction, then this task will get a slot before the normal priority tasks.


>> C4: How do you guarantee that despite having priority, high priority
>> tasks do not start using the bus while there are still still using
>> it in the oposite direction?

This is handled in the first condition "if(bus_slots.value == BUS_CAPACITY || curr_direction ==  task.direction )"
If a high priority tasks comes in opposite direction to the current use;
bus_slots.value != BUS_CAPACITY  since it was reduced by sema_down by task in different direction
curr_direction !=  task.direction, hence the if statement returns false.


---- RATIONALE ----

>> C6: Why did you choose this design? Did you consider other design 
>> alternatives? In what ways is it superior to another design you considered?

I chose the design,  using the narrow bridge concept because it gave a logical flow corresponding to  the stated 
batchScheduler problem.
