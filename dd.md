# CS 162 Project 2
> For project 2, we are going to start from Pintos skeleton code.
## Task 1 Efficient Alarm Clock
### Data Structures and Functions
> In `/pintos/src/threads/thread.c`
```C
 struct sleepy_thread {
    struct list_elem elem;
    int64_t sleep_time;
    int64_t start_time;
    struct thread curr_thread;
};
```
* Create a link list `sleep_list` in `thread.h` which stores all the sleeping threads.
* `sleep_list` is sorted by the time when each thread will be waken up so that each time we only need to wake up threads in the front.
* Create `list_insert_ordered(struct thread t)`, which just pushes the `thread t` on to the end of `ready_list` for now.
### Algorithms
**During the context switch**
```C
FOR x IN sleep_list:{
        IF timer_elapsed(x.start_time) >= sleep_time:{
            CALL list_insert_ordered(x.curr_thread);
        }
}
```
### Synchronization
* We will add a lock `sleep_thread_lock`.
    * call `lock_acquire()` when trying to waking up the sleeping thread.
### Rationale
*  We have a `sleep_list` where we have sleeping threads, and when we switch to a different thread, we check each thread from `sleep_list` and see if they have slept enough. If they have been sleeping for enough time, we insert it to the `ready_list`, which stores threads that have waited enough. 

## Task 2 Priority Scheduler
### Data Structures and Functions
> In /pintos/src/threads/thread.h 
```C
static int max_priority = 0;
```
* Modify thread struct
```C
Int curr_priority 
// Int old_priority, set to 0;
struct lock *locks_I_am_holding
```

### Algorithms
#### Choosing the next thread to run
* We keep a sorted list of ready threads based on thread priority. The next thread to run is the first thread on ready list. To keep the sorted ready list, we need to change:
```C
 thread_unblock(){
    list_insert_ordered(&ready_list, cur_thread());
}
```
> and 
```C
list_insert_odered() {
    ITERATE THROUGH ready_list
        if(elem->curr_priority < this_thread->curr_priority) {
            INSERT in front of (elem);
        }
   }
```
* Besides, In `thread.c`: change `thread_yield()`, call `list_insert_ordered() `during yield

#### Releasing a Lock
##### Approach 1
```C
On lock_release(){
        If (old_priority != -1) {
            Int temp = old_priority
            Old_priority = -1;
            set_prority(curr_thread, temp);
        }
}
```
* For priority donation, a special case to consider: T3 with original priority 3 holds lock LA and LB. T2 with priority 5 then tries to acquire LA, raising T3’s priority to 5. Then T1 with priority 8 tries to acquire LB, raising T3’s priority to 8. After T3 release lock LB, its priority should drop to 5 instead of 3.
* To address this problem, each thread needs to keep a list of locks it currently holds. Once a thread releases a lock, it loops through all the locks it holds and sets its priority to max(priorities of threads waiting for locks it holds)
##### Approach 2
```C
On lock_release(){
      Put the first thread Ta in lock’s wait list(lock->semaphore->waiters) on ready_list; 
      //wait list is sorted by priority
      If (Ta’s priority ==  priority of the thread(Tb) that just released the lock):
                  Decide current priority of Tb:
                  Tb_priority = 0;
                  For ( lock in all_locks_ Tb_now_holds):
                                Tb_priority = max(Tb_priority, lock_wait_list[0].priority); 
                  set_priority(Tb, Tb_priority);  
}
```

#### Acquiring a Lock
> On `lock_acquire()` 
```C
check if thread hold the lock’s curr_priority < curr_priority
//lock_thread->old_priority = lock_thread->curr_priority;
lock_thread->curr_priority = curr_thread()->curr_priority;
Remove lock_thread from ready_list;
Insert_in_orderd(&ready_list, lock_thread);
Insert_in_orderd(&lock_waitlist, cur_thread);
Block curr_thread();
```
#### Computing the effective priority
```C
next_priority = 0;
for ( lock in all_locks_ Tb_now_holds):{
    next_priority = max(next_priority, lock_wait_list[0].priority);
    }
```
#### Priority scheduling for semaphores and locks
*  As mentioned in releasing a Lock, a lock’s wait `list(lock->semaphore->waiters)`is sorted by thread priority. The first thread has the highest priority. Thus when a lock is released, semaphore would prefer the thread with highest priority. 
```C
sema_down(){
  list_insert_orderd(&lock_waitlist, cur_thread);
}
// we reuse the same inserting algorithm as ready list
```
#### Priority scheduling for condition variables
* Change in thread_unblock() would affect condition variables
#### Changing thread’s priority 
```C
set_priority(){ 
    //if (old priority == -1) {
        //old priority = curr_priority
//}
    Set the priority
    if (curr_thread()->curr_priorty >= max_priority) {
        max_priority = curr_thread()->curr_priorty;
} else {
    thread_yield();
    }
}
```

### Synchronization
* The priority scheduling part of our design above is already safe from interrupts.
* To make sure priority donation operations are always atomic and safe from race conditions,  we will disable interrupts whenever priority value is updated.

### Rationale
* For sleep list, ready list and waitlist of lock,  efficiency comparison between keeping a sorted list and unsorted list: 
    * Advantage of maintaining a sorted list: only need to pop from the front (the thread in the front always have highest priority)
    * Disadvantage of maintaining a sorted list: O(n) comparison for inserting.
* Since we need to query the highest priority in the wait list each time we compute a thread’s effective priority, it is more efficient to keep wait lists sorted.

## Additional Questions
* 1. According to switch.S, we push the return address and registers to the current thread’s stack. Since different threads have different registers and stacks, it is important for each thread to save their own registers and stacks. Also, if we check the thread struct, there is a variable called stack so that we can access it easily. Therefore, when we switch threads, we can access the stack pointer of the current thread. 
* 2. The page containing thread’s stack and TCB is freed inside `thread_schedule_tail()`, which is called by schedule as the last step of `thread_exit()`.  Since the execution of `thread_exit()` depends on a complete thread structure, we need to leave `palloc_free_page` to the very end so that `thread_exit()` can still function and doesn’t pull out the rug under itself.
* 3. It executes on the kernel stack.
* 4. Answered below
```C
struct argument {
        Struct lock *lock;
        Struct semaphore *sema;
};
void low(void* arg) {
    struct argument args = arg;
struct lock *lock  = args->lock;
struct semaphore sema = args->sema;
lock_acquire(lock);
sema_down(sema);
printf("this should be printed first\n");
sema_up(sema);
lock_release(lock);
}
void high(void* arg) {
    struct argument args = arg;
struct lock *lock  = args->lock;
lock_acquire(lock);
lock_release(lock);
}
void med(void* arg) {
    struct argument args = arg;
        Struct semaphore sema = args->sema;
        sema_down(sema);
        printf(“this should be printed second\n”);
        sema_up(sema);
}
int main(void) {
    Struct lock lock;
    Struct semaphore sema;
    lock_init(&lock);
    sema_init(&sema, 0);
    Struct argument arg = {&lock, &sema};
    thread_create(“low”, 0, low,arg);
    thread_yield();
    thread_create(“med”, 1, med, arg);
thread_yield();
    thread_create(“high”, 2, high, arg);
    thread_yield();
    sema_up(&sema);
}
```
* The expected output is:
    * this should be printed first
    * this should be printed second
* But actual output is:
    * this should be printed second 
    * this should be printed first
* In the test we create 3 threads, low priority, median priority and high priority. We let the low priority thread take the lock, and then wait on sema. Then we create a median priority thread, let it wait on sema. In the end, we create a high priority thread, let it grab the lock. High priority thread will now donate its priority to low priority thread. After all 3 threads get blocked, call sema up. This should wake up thread low first, but instead, it wakes up thread med, which indicates the bug.

















