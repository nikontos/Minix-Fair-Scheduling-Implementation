# **Operating Systems 2017** #

### **Lab Assignment #2** - *Implementation of Fair Scheduling in MINIX 3.2.0* ###

##### 1) Transfer of process group ID #####

  * In file **/usr/src/servers/pm/schedule.c**, in it's `return`, the function `sched_start_user( )` calls `sched_inherit( )`, which sends a message to *sched*.

  * In the function parameters of `sched_inherit( )`, we added `rmp->mp_procgrp`. In doing this, the message send from *pm* to *sched* includes the group ID of the process.
  * We had to change the function declaration of `sched_inherit( )` (file **/usr/include/minix/sched.h**) and it's definition (file **/usr/src/lib/libsys/sched_start.c**), and add the extra parameter/field.

  * *sched* receives this message in function `do_start_scheduling( )` of file **/servers/sched/schedule.c**. There, we assign it to field `procgrp` of `struct schedproc`.
  * In file **/servers/sched/schedproc.h**, we added the field `procgrp` for the aforementioned reasons.

##### 2) Add, initialize and update helper fields for calculating the user processes' priority. #####

  * In file **/servers/sched/schedproc.h**, we added the extra fields `proc_usage`, `grp_usage`, `fss_priority`.

  * In the same file, in function `do_start_scheduling( )`, we initialized the above fields.

  * After that, in function `do_noquantum( )` of the same file, we update the helper fields according to the fair scheduling algorithm.

##### 3) Reduce number of user queues and apply fair scheduling algorithm. #####

 * First, we reduced the number of user process queues by editing the file **/usr/include/minix/config.h** and to be more precice:

     `#define NR_SCHED_QUEUES 9 /* from 16 */`

     `#define MAX_USER_Q  	8 /* from 0 */`


  * In function `schedule_process( )` of file **/servers/sched/schedule.c**, we find the *user* process (checking for `priority == USER_Q`) with the minimum `fss_priority`, and schedule it by calling `sys_schedule( )`. By doing this, we make sure that after the next step we are going to describe, this process will be found on the head of the process queue in the kernel.

  * Having done the above, we repeatedly call `sys_schedule( )` for the remaining processes in the process table of *sched*. These processes enter the queue one after the other in the tail. If a process was already in the queue, it gets *dequeued* and *enqueued* again right after, entering the tail.

  * We took care only to include processes with `flags == IN_USE` to avoid errors.

##### 4) Tests #####

   * First, we ran the bash command `cd /usr/bin; sh sh;` in 4 terminals. File sh is a script that runs `echo` on an infinite loop, incrementing a counter. We verified by the rate of execution of `echo` that the groups get equal time shares.
   

   * We ran the Minix tests with command ` cd /usr/src/test; make all; ./run` and all but 48 (as expected) outputted **OK.**
   
   * We ran the bash command `top` and watched the output. It confirmed that the fair scheduling algorithm was correctly implemented.

### Debugging cases ###

1. In file **/servers/sched/schedule.c**, in function `do_start_scheduling( )`, we added a `printf( )` outputting the group ID.
2. In functions `schedule_process( )` and `do_noquantum( )` of file **/servers/sched/schedule.c** we used several `printf( )` calls during debugging.
3. By running the tests ` cd /usr/src/test; make all; ./run` we would see if everything is working properly.
4. For each error that was outputted, we searched the source code for it's `printf( )` and found it's parameters (error codes etc.). This way, we aquired clues of where the error occured and what triggered it.
5. Special case #1: Error `PM: SCHED denied taking over scheduling of (process name): (return value of sched_start( ))`, which occured because we hadn't included the extra field in the function definition (see (1) - second bullet).
6. Special case #2: Error `PM: An error occurred when trying to schedule (process endpoint): (error code)` appeared several times, and we solved it by including control statements (mostly for `flag IN_USE`) during the repeated call of function `sys_schedule( )`.