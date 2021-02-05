---
title: "System Calls"
layout: page
---

Argument Passing
====================================


**1. Data structures and functions:**


```
* Array of addresses to the strings in stack -- used to find argc, we malloc this array before hand and then push each argument to the stack after we push the actual strings to the stack
            strtok_r() /* used to parse the arguments passed in. */
        memset() /* used to overwrite memory and modify the values stored. */
        memcpy() /* used to overwrite a memory location with new data. */
        malloc() /* allocates a page with the specified number of bytes without nulling the data there, to store values we do not want to be overwritten. */
        calloc() /* allocates a page in heap with the specified number of bytes, setting the space given to zeros. */
```


**2. Algorithms:**
        The algorithm we are planning to use is:
* In process_create(), parse name from fncopy and give it to thread_create() which gives it to the load()
* Parse arguments in setup stack using for loop with strtok_r(), push each argument onto stack and save the addresses in order to create argv
* Doing that we also get the argc from that for loop
* Stack align with null bytes using memset
* Push argc and return address
        In between each step we should verify if the address is still within the stack space assigned which is between PHYS_BASE and 0x08048000.


**3. Synchronization:**
            No synchronization needed because each process gets its own stack containing the  arguments passed into it.


**4. Rationale:**
*Why is it better than the alternative?
* The alternatives are that we create a helper function but then that creates another stack frame and that takes so much time. This option would also require more code but is easier to conceptualize. 
* There are some advantages to using strtok_r() in our implementation rather than strtok() because in the implementation of strtok_r() we are saving a non-static pointer while the strtok() saves a static pointer which limits you from being able to parse multiple strings at a time.
* Shortcomings?
* The alternatives are that we create a helper function but then that creates another stack frame and that takes so much time. This option will be faster and require less code but will have more complex algorithms and might be harder to add features later on.
* We are splitting the name in kernel mode instead of user mode. Might be a target for insertion attacks.


Process Control Syscalls
====================================
* Our implementation uses a one to one tid to pid mapping. This should simplify the code and we won’t have to use up repetitive data since there is only one process per thread. 
* For all syscalls, we must perform a validation check on the arguments that are being passed into the respective function. Assuming each of the three different arguments we can get are not null, we can simply check that the address space to which they refer to is valid before proceeding with the execution of the function.
* For an executable, if we are writing to the executable we should just call file_deny_write() in order to cancel writes to executables currently running. We get the executable names from the load function in userprog/process.c.
 * Validation Algorithm:
* Check if the stack pointer is in user space using the function is_user_vaddr() in threads/vaddr.h, if not return -1. Assuming we are not extending the stack for a thread, we would just check if the pointer is within PHYS_BASE and 0x0804800 as said in the spec.
* Start dereferencing the stack pointer for the arguments being passed in
* Check if each argument is in userspace
* Use pagedir_get_page() in pagedir.c on the user address with the thread’s page directory, if it returns null that means that the address is not mapped and return -1. Else continue with the system call
* Then for each pointer to a pointer we check their address with the same algorithm until every address is validated


## Wait Syscall
**1. Data structures and functions:**
```C
struct wait_status {
tid_t tid; /* stores the child thread id */
int exit_code; /* stores the exit status of the thread with tid */
struct list_elem elem; /* Used to append to the parent thread’s children list */
struct lock ref_lock; /* Makes sure concurrency is not an issue between parent reading and child writing the ref_count */
int ref_count; /* stores 2 if the child and parent are alive, 1 if one has died and 0 if both have died */
struct semaphore dead; /* stores 0 if the child is still alive and 1 if the child has died */
}
```
To struct thread we are going to add:
```C
struct list children; /* List of wait_statuses */
struct wait_status * wait_status; /* stores the process’s status */
```
Description: The wait_status structure book keeps children for the parent thread and can be used by the kernel to freespace if both parent and its children have died. 


**2. Algorithms:**
* Call process wait on a child tid.
* Process wait will iterate through the child process list stored in the struct.
* While iterating we will check if the tid is the same as the tid passed in and if it matches we call sema down on the one stored in the child’s struct.
* Once the child exits we call sema up and on the same semaphore, waking up the parent to continue executing.
* After the child exits we set the child’s exit code to dead and decrease the child’s reference count and finally return our exit code.
* We need to decrease the child’s reference count, so that we know that the memory used can be freed.


**3. Synchronization:**
Assume that we have a parent process P and a child process C. There are multiple ways that the wait can ensure proper synchronization:
* When P calls wait(C) before C exits then the parent will try to down semaphore dead, but will be blocked if C is not exited. Once C exits, the semaphore is then upped thus allowing the parent to retrieve the exit status.
* After C exits before the parent calls wait, the semaphore will be upped before P can sema down and thus the parent can retrieve the exit_code through the wait_status struct
* We decrease the child reference count in order to check if the wait_statuses will be freed.
* If P terminates without waiting before/after C exits then, the child will not be blocked because it just signals the dead parent, but the parent is not there. Then
* The special cases might be external interrupts and having the child process error out before it could exit.


**3. Rationale:**
* Why is it better than the alternatives?
* If you store the exit code in the thread struct, we need struct thread pointers but we would not really know when the parent or child dies if one or the other is freed. Thus having a separate struct for the status is ideal because then we can tell if either of the parent or child has died and needs to be freed.
* Both the alternatives mentioned here takes O(n) time where n is the number of children the process creates.
* Shortcoming?
* The shortcoming is that the communication between the parent and child is more complex due to the separation of the data. It also uses more space to have two separate structs for parent and child. We will also have to free this and will take up more time than just a struct thread pointer to the data.  


## Exec Syscall
**1. Data structures and functions:**
We will add this to struct thread:
```C
struct semaphore load_sema; /* Locks the system after we exceed the maximum threads */
bool load_status;     /* stores the success of the child */
```


**2. Algorithms:**
* First the parent calls process_execute() with the command line arguments.
* Then process_execute calls thread_create() which creates a new thread with start_process().
* The child process then calls load which then returns the success, which the child then updates to the parent and then calls sema_up to signal it’s status.
* The parent concurrently downs the load_sema, and ten checks the load status.
* Return -1 if failed, return child_tid if the child did not fail.


**3. Synchronization:**
First the parent calls down on the semaphore, this blocks the parent until the child ups the semaphore with the load_status updated. The load status is passed back to the parent thread and put in load_status as a -1 or the tid of the child. 


**4. Rationale:**
* Why is it better than the alternative?
* An alternative would be a conditional lock, but it takes some CPU resources because we could have to loop and keep checking if the child has finished the load(). Semaphores are better in this regard because it just blocks until the child is done. The semaphore is obviously simpler.
* The runtime of this algorithm depends on how much time the child loading takes.
* Shortcoming?
* It might be hard to implement with all the synchronization.


## Exit Syscall
**1. Data structures and functions:**
List containing children so that we would call sema functions on matching children.
Sema_up and Sema_down to terminate children. 
Wait_status to change the ref_cnt of children.


*2. Algorithms:**
* Save the exit code in the shared data. Up the semaphore in the data shared with our parent process (if any). In some kind of race-free way, mark the shared data as unused by us and free it if the parent is also dead. Iterate the list of children and, as in the previous step, mark them as no longer used by us and free them if the child is also dead. Terminate the thread. Therefore, call’s process exit, shares the exit code with the parent, dies


*3. Synchronization:*
All shared data between children and parent process. To safely free shared data, locks and a reference count or pair of boolean variables in the shared data area are used to make sure that the data to be freed is only used by one process.Therefore, concurrency issues would be avoided by freeing data that is used by only one process. The cost of this synchronization is O(N) in both time and space where N is the number of child processes.  Data in the form of locks, reference count, and boolean variables are needed for each child process. 


**4. Rationale:**
* Why is it better than the alternative?
* There are no alternatives.
* Shortcoming?
* If there are lots of data that are shared, there could be a lot of locks or variables for each child process. 
* In order to exit a process, all data used by the process and dead child processes must be freed to prevent memory leaks. After freeing all the data, it is safe to terminate the process. 


## Halt Syscall
**1. Data structures and functions:**
We just use the shutdown_power_off ()
process_exit() to free each process.


**2. Algorithms:**
* Free all memory that is being used by the processes
* Shutdown the OS using hardware


**3. Synchronization:**
The syscall frees all memory shared by threads, so there inherently will not be any synchronization issues that arise from this syscall.


**4. Rationale:**
* Why is it better than the alternative?
* There are no alternatives, and this is virtually the only possible implementation of the functionality we are trying to achieve.
* Shortcoming?
* Does not apply.


## Practice Syscall
**1. Data structures and functions:**
```C
Addition() /* used to add two values together. */
```


**2. Algorithms:**
* f->eax += 1


**3. Synchronization: **
No synchronization should be needed.


**4. Rationale:**
* Why is it better than the alternative?
* There are no alternatives.
* Shortcoming?
* Does not apply.


File Operation Syscalls
====================================
**1. Data structures and functions:**
```C
struct file_status {
        int fd;           /* The file descriptor, basically a FID for the file */
        struct file * fp;   /* The file pointer, pointing to the file */
        char * file_name;    /* The name of the file */
        
        struct list_elem elem; /* list elem */
}
struct lock file_lock; /* a global lock on the file system */
        struct list file_list; /* list of open files, will use pintos linked list implementation (constant list size) */
        static int next_fd; /* the next file descriptor assigned */
```


In src/filesys/filesys.c:
```C
void filesys_init(); /* to initialize the file_lock*/
void filesys_create(); /* for create syscall */
void filesys_open(); /*for open syscall */
void filesys_remove(); /*for remove syscall */
```


In src/filesys/file.c:
```C
void file_close(); /* for close syscall */
void file_read(); /* for read syscall */
off_t file_write(); /* for write syscall */
off_t file_length(); /*for file size syscall */ 
void file_seek(); /* for seek syscall */
void file_tell(); /* for tell syscall */
```


**3. Synchronization:**
We do not want any interruptions when creating a new file, or our new file could potentially be filled with junk. Acquiring a file_lock should ensure the atomicity of this operation. 


**4. Rationale:**
* Why is it better than the alternative?
* We use the Pintos linked list implementation because our array is of fixed size for the sake of this project, and this particular implementation of the linked list allows for flexibility in the maximum number of files while also providing for relatively fast and clean times of access.
* Our process has separate file descriptors that are only unique to itself but not the whole operating system. One advantage to our file descriptor design might be that no other processes can use the same file descriptor that is being used in another process (i.e. one process can use a file descriptor of four and another can also use a file descriptor of four). That way it separates the files opened by the child and parent thread and should increase security.
* There is a unique file descriptor per file that is opened.
* Shortcoming?
* We do not really see any shortcomings with any of these syscall implementations at all, as the bulk of the functionality for all of these syscalls is more or less already written in src/filesys/filesys.c and src/filesys/file.c.


##Create Syscall
**2. Algorithms:**
* Acquire file_lock
* Use filesys_create() to create file
* Release file_lock


##Remove Syscall
**2. Algorithms:**
* Get file->file_name
* Acquire file_lock
* Use filesys_remove(file_name)
* Release file_lock


##Open Syscall
**2. Algorithms: **
* Acquire file_lock
* Use filesys_open(name)
* Create a file_status struct and set to be the return value of the call to filesys_open()
* Get next_fd and set that to the struct file_status fd
* Increment next_fd by 1
* Release file_lock
* Return the file descriptor from filesys_open()


##Filesize Syscall
**2. Algorithms:**
* Acquire file_lock
* Search the open files list in struct thread for the file_status struct with the file descriptor argument. 
* Get the file pointer from that file_status struct
* Execute file_length() on the file pointer.
* Release file_lock


##Read Syscall
**2. Algorithms:**
* Acquire file_lock
* Search thread’s open files for the file_status struct, if not found return -1.
* Get file pointer from that file_status struct
* Execute file_read() on the file pointer, the buffer from the argument, and the size to read from the argument.
* Release file_lock.
* Return the value returned from file_read().


##Write Syscall
**2. Algorithms:**
* Obtain the lock on the file system.
* If the file descriptor is the stdout file descriptor we use putbuf() on buffer and size.
* Else we get the file_struct from the process and check if the file descriptor exists, if not return -1.
* Get the file pointer from the file_status struct.
* Call file_write() on the file pointer we got, the buffer, and the size from the arguments passed in.
* Release the lock on the file system


##Seek Syscall
**2. Algorithms: **
* Obtain the lock on the file system
* Calculate file size
*If new_pos > filesize:
* file_seek() to EOF
* Else:
* file_seek() to new_pos
* Release file_lock


##Tell Syscall
**2. Algorithms:**
* Obtain file_lock
* Get file pointer from going through open files list in struct threads and searching for file descriptor
* Execute file_tell() on the file pointer
* Release the file_lock
* Return result from file_tell()


##Close Syscall
**2. Algorithms: **
* Obtain file_lock
* Get the struct file_status with the file descriptor
* From that get the file pointer
* Call file_close() on the pointer (file_close() frees the file itself)
* Free the struct file_status
* Release the file_lock


Additional Questions:
====================================
1. Take a look at the Project 1 test suite in pintos/src/tests/userprog. Some of the test cases will intentionally provide invalid pointers as syscall arguments, in order to test whether your implementation safely handles the reading and writing of user process memory. Please identify a test case that uses aninvalidstack pointer (%esp) when making a syscall. Provide the name of the test and explain how the test works. (Your explanation should be very specific: use line numbers and the actual names of variables when explaining the test case.)


Name of the test is: tests/userprog/exec-bad-ptr.c
```C
#include <syscall.h>
#include "tests/main.h"


void
test_main (void)
{
  exec ((char *) 0x20101234);
}
```
Explanation: First, in the line after the test_main function we have some address that for sure is not valid in the userspace. We call that invalid pointer to the exec system call which should detect if that address is going to page fault or not.  It should return -1.


2. Please identify a test case that uses a valid stack pointer when making a syscall, but the stack pointer is too close to a page boundary, so some of the syscall arguments are located in invalid memory. (Your implementation should kill the user process in this case.) Provide the name of the test and explain how the test works. (Your explanation should be very specific: use line numbers and the actual names of variables when explaining the test case.)


Name of the test is: tests/userprog/exec-bound.c
```C
/* Exec a child with an exec string that spans a page boundary. */


#include <syscall.h>
#include "tests/userprog/boundary.h"
#include "tests/lib.h"
#include "tests/main.h"


void
test_main (void)
{
  wait (exec (copy_string_across_boundary("child-args arg1 arg2")));
}
```
Explanation: Essentially the static string is being copied across a page that should not be allocated, through the copy_string_across_boundary function. The execute function is then called on it and should not be able to access the string because the pointer is in invalid memory. Execute should return -1 which is then passed into wait. Since -1 is not a valid tid, wait should also fail thus giving us no page faults.


3. Identify one part of the project requirements which is not fully tested by the existing testsuite. Explain what kind of test needs to be added to the test suite, in order to provide coverage for that part of the project. (There are multiple good answers for this question.)
* Seek appears to not be covered in the test suite.
* Create a known string and write it to a file. Call file_seek on the file on the 0th byte and call file_read on the file. Compare the output with the known string and check if they are the same
