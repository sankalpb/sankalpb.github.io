---
layout: post
title: Common synchronisation primitives and their application.
category: CS
---
### Fair reader writer problem

#### Reader preference
    
    lock(2)     lock(1)                 
    //do write    if (readcount++)       
    unlock(2)       lock(2)              
                unlock(1)               
        	// do read                      
        	lock(1)                 
                  if (!readcount--)     
                    unlock(2)          
                unlock(1)                                                               

#### Writer preference

     lock(1)                 lock(2)               
       if !(writecount++)    if (!readcount++)     
          lock(2)               lock(3)            
     unlock(1)               unlock(2)             
     lock(3)                 //                    
     unlock(3)                                     
     lock(1)                lock(2)                
       if !(--writecount)   if !(--readcount)      
          unlock(2)            unlock(3)           
     unlock(1)              unlock(2)

#### Neither preferred

     lock(4)        lock(4); lock(3)	 
     lock(2)        if !(readcount++)	 
     unlock(4)        lock(2)		 
     //             unlock(3); unlock(4)	 
     unlock(2)      // 			 
                    lock(3)		 
                    if !(--readcount)	 
                       unlock(2)		 
                    unlock(3)              

### Producer consumer problem (using semaphores)
     sem_init(resources, 0)
     sem_init(places, n)
        
     producer                  consumer
  
     sem_down(places)          sem_down(resources)
     enqueue_locked(item)      enqueue_locked(item)
     sem_up(resources)         sem_up(places)
    
### Producer consumer problem using condition variables
* a state variable is essential to prevent missed wakeups, state variable indicates if signalling happened before wait was called.
* primitives:
  * wait(condition, mutex) [mutex must be locked at the time of calling, it is released on sleep and retaken on wakeup] 
  * signal(condition) - [taking mutex is recommended during signal, eg. in the simple case, state variable checking might be interleaved in a way such that there is still missed signal.]
  * broadcast(condition) - like signal but wakes all process and not
  only head.
  
~~~~~~~~~~~~~~~~~
producer() 
lock(mutex)                                                               	
while (full)                        // while ensures state check 	  		
     wait(condition_nonfull, mutex)    after wakeup which is necessary    	
put()                                  if there are multiple signalees    	
signal(condition_nonempty) 
unlock(mutex)

consumer()
lock(mutex)			
	    while (empty)		
    wait(condition_nonempty, mutex) 
get()				 
signal(condition_nonfull)		 
unlock(mutex) 
~~~~~~~~~~~~~~~~~

### compare_exchange primitive

    compare_exchange(target, expected, desired) - if target = expected, target = desired else return value of target in expected.

### Bakery algorithm

     bakery algorithm - read writes are atomic, entering[i] = 0 & ticket[i] = 0 for i  = 1 to n
     entering[i] = true 
     ticket[i] = max (ticket[i] for all i)
     entering[i] = false
     for all j
         while (entering[j]);
         while((ticket[j] != 0) && ((ticket[i] > ticket[j]) || (ticket[i] == ticket[j] && i > j)  )

### RCU locks
- key idea: on all systems load and store to pointers are atomic. any_t *ptr = something;
- make a copy of shared data, update, swap pointers and free old copy. 
- additional work required (publish/subscribe) to ensure consistency wrt compiler and CPU reordering.
- publish - rcu_assign_pointer()
- subscribe - rcu_dereference()
- waiting for things to finish before cleaning up - synchronize_rcu()
  freeing the old structure - some readers might have references and hence cannot be freed
  however, after all processors undergo one schedule, old structure can be freed the free is done with this  function
  call_rcu(struct rcu_head *head, void (*func)(void *arg), void *arg);
  func is callback that gets called after all processors undergo schedule
- rcu_read_[un]lock = preempt disable. Also means no sleeping while
under lock.

### Seq locks
- has associated sequence number. Readers log sequence number before critical section and match afterwards else retry.
- reader
~~~~~~~~~~~~~~~~~~
    do {
        seq = read_seqbegin(&the_lock);
        /* Do what you need to do */
    } while read_seqretry(&the_lock, seq);
~~~~~~~~~~~~~~~~~~
- writer
~~~~~~~~~~~~~~~~~~
     void write_seq[un]lock(seqlock_t *lock);
~~~~~~~~~~~~~~~~~~
