+++
title = 'GSoC 2025 Midterm Evaluation: Implementing POSIX Issue 8 Functions in RTEMS'
date = 2025-07-16T21:51:29+03:00
draft = false
tags = ['GSoC', 'RTEMS', 'POSIX', 'C', 'Programming', 'Open Source']
categories = ['Development']
+++

# GSoC 2025 Midterm Evaluation: Implementing POSIX Issue 8 Functions in RTEMS

As I reach the midpoint of my Google Summer of Code 2025 journey with RTEMS, I'm excited to share the progress I've made on implementing POSIX Issue 8 functions. This project aims to enhance RTEMS' POSIX compliance by adding support for new functions introduced in the latest POSIX standard.

## Project Overview

My GSoC project focuses on implementing and testing 8 key POSIX Issue 8 functions in RTEMS:

- `pthread_cond_clockwait()`
- `pthread_mutex_clocklock()`
- `pthread_rwlock_clockrdlock()`
- `pthread_rwlock_clockwrlock()`
- `sem_clockwait()`
- `timespec_get()`
- `posix_getdents()`
- `ppoll()`
- `dladdr()`
- `quick_exit()` and `at_quick_exit()`

## Completed Work

### 1. POSIX Header OK Tests (✅ Completed)

**Merge Request:** [#497](https://gitlab.rtems.org/rtems/rtos/rtems/-/merge_requests/497)

The first major milestone was creating API header OK tests for all the functions I planned to implement. These tests verify that the function prototypes are correctly declared in the appropriate headers.

```c
/* Example from testsuites/psxtests/psxhdrs/pthread/pthread_cond_clockwait.c */
#define _POSIX_C_SOURCE 202405L

#ifdef HAVE_CONFIG_H
#include "config.h"
#endif

#include <pthread.h>

#ifndef _POSIX_THREADS
#error "rtems is supposed to have pthread_cond_clockwait"
#endif

#if !defined(_POSIX_CLOCK_SELECTION) || _POSIX_CLOCK_SELECTION == -1
#error "rtems is supposed to have pthread_cond_clockwait"
#endif

int test(void);

int test(void)
{
  pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
  pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
  clockid_t clock_id = CLOCK_REALTIME;
  struct timespec abstime;
  int result;

  result = pthread_cond_clockwait(&cond, &mutex, clock_id, &abstime);

  return result;
}
```

This phase involved creating comprehensive tests for all target functions, ensuring they would compile correctly with the proper feature test macros and guards.

### 2. Newlib Header Updates (✅ Completed & Merged)

**Newlib Patch Series:** [Merged into newlib main](https://sourceware.org/pipermail/newlib/2025/021926.html)

A crucial part of the project was updating the newlib C library (which RTEMS uses) to include the proper function prototypes. My 5-patch series was successfully merged into newlib's main branch:

```c
/* Added to newlib/libc/include/time.h */
#if __POSIX_VISIBLE >= 200112 || __ISO_C_VISIBLE >= 2011
int timespec_get(struct timespec *, int);
#endif

/* Added to newlib/libc/sys/rtems/include/semaphore.h */
#if defined(__rtems__) && (_POSIX_C_SOURCE >= 202405L)
int sem_clockwait(sem_t *__sem, clockid_t __clock_id, 
                  const struct timespec *__abstime);
#endif

/* Updated guards for quick_exit() and at_quick_exit() in stdlib.h */
#if __ISO_C_VISIBLE >= 2011 || (_POSIX_C_SOURCE >= 202405L)
int at_quick_exit(void (*)(void));
void quick_exit(int);
#endif
```

The patch series included:
1. **timespec_get()** prototype addition to `time.h`
2. **pthread.h** guard updates for POSIX Issue 8 functions
3. **sem_clockwait()** prototype addition to `semaphore.h`
4. **ppoll()** prototype addition to `sys/poll.h`
5. **quick_exit()** and **at_quick_exit()** guard updates in `stdlib.h`

### 3. pthread Clock Functions Implementation (✅ Completed)

**Merge Request:** [#547](https://gitlab.rtems.org/rtems/rtos/rtems/-/merge_requests/547)

I successfully implemented four pthread clock functions:

#### pthread_cond_clockwait()
```c
int pthread_cond_clockwait(
  pthread_cond_t *cond,
  pthread_mutex_t *mutex,
  clockid_t clock_id,
  const struct timespec *abstime
)
{
  if (abstime == NULL) {
    return EINVAL;
  }

  if (clock_id != CLOCK_MONOTONIC && clock_id != CLOCK_REALTIME) {
    return EINVAL;
  }

  return _POSIX_Condition_variables_Wait_support(cond, mutex, abstime, clock_id);
}
```

#### pthread_mutex_clocklock()
```c
int pthread_mutex_clocklock(
  pthread_mutex_t *mutex,
  clockid_t clock_id,
  const struct timespec *abstime
)
{
  if (clock_id == CLOCK_REALTIME) {
    return _POSIX_Mutex_Lock_support(
      mutex,
      abstime,
      _Thread_queue_Add_timeout_realtime_timespec
    );
  }
  
  if (clock_id == CLOCK_MONOTONIC) {
    return _POSIX_Mutex_Lock_support(
      mutex,
      abstime,
      _Thread_queue_Add_timeout_monotonic_timespec
    );
  }
  
  return EINVAL;
}
```

The implementation also included comprehensive test suites that verify:
- Correct behavior with different clock types (CLOCK_REALTIME, CLOCK_MONOTONIC)
- Proper error handling for invalid parameters

### 4. quick_exit() and at_quick_exit() Tests (✅ Completed)

**Merge Request:** [#584](https://gitlab.rtems.org/rtems/rtos/rtems/-/merge_requests/584)

Implementing tests for the `quick_exit()` and `at_quick_exit()` functions. These functions provide a middle ground between `exit()` (which performs full cleanup) and `_Exit()` (which performs no cleanup).

```c
/* Test implementation showing LIFO handler execution */
static int counter;

static void at_quick_exit_0(void) {
  assert(counter == 0);
  ++counter;
}

static void at_quick_exit_1(void) {
  assert(counter == 1);
  ++counter;
}

static void at_quick_exit_2(void) {
  assert(counter == 2);
  ++counter;
}

static void *exit_thread(void *arg) {
  int eno;

  // Register handlers in order: 2, 1, 0
  eno = at_quick_exit(at_quick_exit_2);
  assert(eno == 0);
  
  eno = at_quick_exit(at_quick_exit_1);
  assert(eno == 0);
  
  eno = at_quick_exit(at_quick_exit_0);
  assert(eno == 0);

  // Handlers execute in LIFO order: 0, 1, 2
  quick_exit(EXIT_STATUS);
}
```

The test uses a custom fatal extension to validate that:
1. All `at_quick_exit()` handlers execute in LIFO order
2. The program terminates with the correct exit status
3. `quick_exit()` properly terminates the entire program (not just the thread)

## Remaining Work for Final Evaluation

The remaining functions to implement and test are:

1. **`sem_clockwait()`** - Semaphore wait with clock specification
2. **`timespec_get()`** - Get current time with specified time base
3. **`posix_getdents()`** - Get directory entries in POSIX format
4. **`ppoll()`** - Poll file descriptors with signal mask
5. **`dladdr()`** - Get symbol information from address

## Learning Outcomes

This project has significantly enhanced my understanding of:

- **POSIX Standards**: Deep dive into POSIX Issue 8 specifications
- **Real-time Systems**: Understanding RTEMS architecture and threading
- **C Library Development**: Working with newlib and system-level programming
- **Open Source Collaboration**: Working with maintainers and following review processes
- **Testing Methodologies**: Creating comprehensive test suites for system-level functions

## Looking Forward

The second half of GSoC will focus on completing the remaining functions and ensuring all tests pass. 

This project contributes to RTEMS' goal of providing comprehensive POSIX compliance, which is crucial for developers who need to port POSIX applications to real-time systems.

## Links

- **Project Repository**: [RTEMS GitLab](https://gitlab.rtems.org/rtems/rtos/rtems)
- **Merge Request #497**: [POSIX Header OK Tests](https://gitlab.rtems.org/rtems/rtos/rtems/-/merge_requests/497)
- **Merge Request #547**: [pthread Clock Functions](https://gitlab.rtems.org/rtems/rtos/rtems/-/merge_requests/547)
- **Merge Request #584**: [quick_exit Tests](https://gitlab.rtems.org/rtems/rtos/rtems/-/merge_requests/584)
- **Newlib Patches**: [Merged Patches](https://sourceware.org/pipermail/newlib/2025/021926.html)

---

*This blog post represents my midterm evaluation for Google Summer of Code 2025. The project continues to progress toward full POSIX Issue 8 compliance in RTEMS.* 