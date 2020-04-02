## Changes we have made 
> We have followed the design doc notes and the advice our TA Alan gave during the design doc review session. Below is what we have changed specifically. Most of them are from the design doc notes.

### Task 1 **Efficient Alarm Clock**
#### Changed: 

* `struct sleepy_thread`
    * new struct means we have to `malloc + free`, so we did not use it
    * moved the following to the thread struct
        * `int64_t wake_time = start_time + sleep_time`
            * defined a comparator (`list_less_func`) that compares the `wake_time` of two threads
        * `struct list_elem sleep_elem`
            * to add thread to `sleep_list`
* `sleep_thread_lock`
    * interrupt handlers (ie timer_interrupt) cannot acquire locks
        * `timer_interrupt` is called 100 times/sec, if it blocks trying to acquire a lock, then system will suffer
    * synchronization: disable interrupts, add/remove to the `sleep_list`, enable interrupts
#### Added: 
* added to `timer_sleep` to determine where and when we are inserting to the list
* decided where and when we are removing from the list (ie waking up threads)
    * in `timer_interrupt()`
    * called at each `tick()`
    * waking up threads in `timer_interrupt` will be synchronous to the system clock (timer)


### Task 2 **Priority Scheduler**
#### Changed: 
* kept the `ready_list` **unsorted** because keeping the `ready_list` sorted by priority is really hard to do
    * finding the thread with the highest priority becomes O(N)
    * naturally got round-robin scheduling for multiple threads of max priority 
    * changed the `next_thread_to_run()` function in thread.c to find and remove the max instead of popping the head
* kept the wait-queues of of semaphores and CVs unsorted and use `list_max` to remove and unblock thread with highest priority in the wait-queue
#### Added:
* implemented **recursive** donation logic
    * priority donation chaining: on a call to lock_acquire, recursively donate our priority on want_lock->holder until want_lock == NULL
    
```    c    
        * struct lock *want_lock; 
        * want_lock is NULL if thread is not blocked trying to acquire a lock
        * else, want_lock = the lock that we are blocked trying to acquire
```
        
## A reflection on the project
### What exactly did each member do?
#### **David** 
Worked together with the group on task 1 and task 2. Helped debug for task 2. Did part 2 of task 3 with Yichuan. Helped on 3.f. Final clean up. Did the Final report.
#### **Yichuan** 
Worked together with the group on task 1 and task 2. Did part 2 of task 3 with David, helped on 3.f. 
#### **Yiwen** 
Worked together on task 1. Did priority donation part of task 2 and collaborated with other group members on lock release part of task 2. Did part 1 of task3. Helped on 3.f.
#### **Jun** 
Worked together on task 1 and task 2 via screen sharing and video call. For task 3, did the beginning of the number 2 together. Worked on problem 3. 
### What could be improved?
* We could have spent more time on the design document 
* For Scheduling Lab, we could have walked through the codebase together before working separately.
### What went well?
* Team collaboration for writing codes and debugging: During this trying time, every team member actively contributed to the codebase and communicated with each other. Even though we cannot meet, we used Skype and TeamViewer to debug together. 
* No more *git* issues.






