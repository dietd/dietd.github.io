---
title: "Thread Scheduler"
layout: page
---

Task 1: Efficient Alarm Clock
====================================


**1. Data structures and functions:**
* We need to reimplement the timer_sleep() function, so we need to make changes to make it more efficient.
* We need a sleep_list that stores the threads that have called timer_sleep.
* Functions: timer_sleep(), timer_interrupt()
```C
struct list sleep_list;
struct lock sleep_lock;
struct thread {
int64_t wake_time;
}
```


**2. Algorithms:**
* If timer_sleep() is called with a zero or negative argument, then you should just return immediately.
* Disable interrupts so that the timer_interrupt() function doesn’t run when we are running. (timer_interrupt() manipulates the list without a lock).
* We have the thread acquire the sleep_lock (Other threads can run timer_sleep() concurrently).
* Set the wake_time of the thread to timer_ticks() + amount of inputted ticks. 
* Move the elem of the current thread from the ready_list to the sleep_list.
* Release the sleep_lock.
* Block the thread.


* In timer_interrupt() we just run through the sleep list and unblock if the current ticks stored in the kernel stack is greater than or equal to the set wake_time of the thread.
* We disable interrupts while running this function, so no other threads are calling timer_interrupt at the same time. 
**3. Synchronization:**
* Utilize interrupt disabling and enabling for each function except for timer_interrupt().
* Adding a lock to prevent issues with interrupts occurring while the sleep_list is being modified or traversed (which would cause unwanted list element removals and invalid pointers)


**4. Rationale:**
*Why is it better than the alternative?
        * You don’t have to sort the list by wake ticks, which takes nlogn time while our algorithm only takes n time.
* Shortcomings?
        * It might wait more time then the specified sleep time and thus the thread won’t wake up exactly when it needs to.


Task 2: Priority Scheduler
====================================
**1. Data structures and functions:**


```C
struct thread {
    int eff_priority;                 /* Stores the effective priority */
    int base_priority;             /* Stores the original priority */


    struct list donor_list;               /* Donor_list of threads priorities */
    struct list_elem donor_elem;    /* Donor_elem used for donations */
    struct lock * cur_lock;
};
```


To modify:
```C
static void init_thread (struct thread *t, const char *name, int priority);      /* Initialize the base priority and the effective priority */
static struct thread *next_thread_to_run (void);         /* Change to use list_max */
void thread_set_priority (int new_priority);             /* Change to set the eff_priortiy and have the thread yield */
int thread_get_priority (void);                        /* Change to return the eff_priority */


void sema_up (struct semaphore *);           /* Change to use list_max */
void lock_acquire (struct lock *);               /* Add to held locks list, add to donor list of the thread that is holding the lock */
bool lock_try_acquire (struct lock *);        /* Tries to acquire the lock based of the effective priorities of the threads */
void lock_release (struct lock *);              /* Need to revert the priority if it had been donated to */
```


**2. Algorithms:**
To add:
```C
/* Attempts to donate t1’s priority to t2’s priority. Does nested donations if needed. Add this to lock_acquire().*/
void donate_priority(struct thread * t1, struct thread * t2){
* Check if t1 and t2 are equal if so then return.
* if t1’s priority is less than t2’s priority then don’t donate and return
* add t1 to t2’s donor_list
* set t2 priority to t1’s priority
* if (t2’s cur_lock’s thread is not null) then donate priority from t2 to the thread holding t2’s lock.
}
/* Revert the current thread priority to its original priority after it has released the lock it held and get the next best donated priority. Add this to lock_release().*/
void revert_priority() {
* Iterate through the donor list and remove the threads that were waiting on that lock from the list.
* If the donor list’s size is 0 then we want to revert to the base priority of the thread.
* Then set the current thread’s priority to the maximum priority of the donor list.
}
```


*Choosing the next thread to run:
        * Use list_max to find the highest priority thread in our ready_list, passing in our helper function that compares the priorities.


*Acquiring a Lock
* Get the lock and put the thread on the donor list and change the donated value to true.
* Call donate_priority() on the current thread and the thread that is holding the lock (see algorithm for donate_priority() above).


* Releasing a Lock
* Set the thread holding the lock to be none.
* Call revert_priority() on the current thread, which reverts it’s priority to its base and finds the next best priority (see algorithm for revert_prioity() above).
* sema_up calls the next max priority thread to be the one that is holding the lock.


* Computing the effective priority
* If the lock_acquire function is being run by thread A, if the lock is currently held by some other thread B, we check to see if the effective priority of B is less than the base priority of A (see our algorithm for donate_priority() and revert_priority() above).


* Priority scheduling for semaphores and locks
* In the sema_up function instead of using list_pop_front() we change this to using list_max() in order to get the thread with maximum effective priority more efficiently.


* Priority scheduling for condition variables
         * We will change the cond_wait() statement and use list_max() on the waiters list of the cond struct instead of just popping the thread off the front. This will ensure that we are getting the maximum priority each time the conditional variable is signaled.


* Changing thread’s priority
* In thread_set_priority() there are two cases to consider in order to change the base priority:
        * If there are no donors we have to change both the effective priority and the base priority to the input.
        * If there are donors, we don’t have to change anything just return for correctness.
        * We should call thread_yield assuming the effective priority has increased. That way the scheduler knows that there may be a new maximum priority thread to be run, thus enforcing preemption. 


* Choosing the next thread to run:
* Use list_max() on the effective priorities of the threads to get the next scheduled thread in next_thread_to_run().


**3. Synchronization:**
* We use interrupt disable and enable to ensure that the program runs atomically. If we do not, our thread_set_priority() could fail due to priority donation during the thread switch. This would cause incorrect behavior if not implemented.
* Code in synch.c such as semaphores and locks are already implemented with correct enabling and disabling of interrupts, we could basically add in our code without issues.


**4. Rationale:**
*Why is it better than the alternative?
        *  It does not need to sort the list everytime a new thread or when we release or acquire a lock, because sorting takes O(nlogn) time to run.
* Shortcomings?
        * Our implementation runs in O(n) time because we usually use list_max to get the maximum priority. In future iterations what we might need to do is to not use lists and use max heaps and trees instead of lists, which have an inherently slow runtime.


Additional Questions:
====================================
1. In class, we studied the three most important attributes of a thread that the operating system stores when the thread is not running: program counter, stack pointer, and registers. Where/how are each of these three attributes stored in Pintos? You may find it useful to closely read switch.S and the schedule function in thread.c. You may also find it useful to review Lecture 4 on September 10, 2019.


When a thread is not running the program counter, stack pointer, and register all get pushed to the TCB in the thread’s respective stack.When the thread is restarted, the CPU’s stack pointer is restored to that thread’s stack pointer, and that thread’s program counter, stack pointer, and register information are all popped off and stored in the CPU’s registers.


2. When a kernel thread in Pintos calls thread_exit, when/where is the page containing its stack and TCB (i.e.,struct thread) freed? Why can’t we just free this memory by calling palloc_free_page inside the thread_exit function?


When the kernel thread calls thread_exit, the page with its struct thread is freed in thread_schedule_tail(). We don’t call palloc_free_page inside thread_exit because this call would also destroy thread_exit early, as thread_exit hasn’t completed yet and we don’t want to destroy the same function we’re currently executing.


3. When the thread_tick function is called by the timer interrupt handler, in which stack does it execute?


It is executed in the kernel stack, more specifically the current thread stack.


4. Consider a fully-functional correct implementation of this project, except for a single bug, which exists in the sema_up() function. According to the project requirements, semaphores (and other synchronization variables) must prefer higher-priority threads over lower-priority threads. However, the implementation chooses the highest-priority thread based on the base priority rather than the effective priority. Essentially, priority donations are not taken into account when the semaphore decides which thread to unblock. Please design a test case that can prove the existence of this bug. Pintos test cases contain regular kernel-level code (variables, function calls, if statements, etc) and can print out text. We can compare the expected output with the actual output. If they do not match, then it proves that the implementation contains a bug. You should provide a description of how the test works, as well as the expected output and the actual output.


We would have 4 threads with priorities A (31), B (32), C (33), D (34). Each of them before they exit will print out their name. We will also have a lock and a semaphore defined for each of them.


Have A acquire the lock first, then try to down the semaphore. How have B down the semaphore as well. This means that A and B are on the semaphore list. Now have C acquire the lock. C will then donate it’s priority to A because A currently has the lock. Thus, A will have priority 33. Let D up the semaphore twice. A should then release the lock allowing C to run. B will also be running and printing. 


What we expect when we run D is that the output would print out the thread names with A before B. However because we don’t pop the threads off the semaphore list by effective priority we will have that B will print out before A, since B’s base priority > A’s base priority.
