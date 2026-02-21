### Atomic Assignments
```java
void someMethod(int passedInt) {
   counter = passedInt; // counter is an instance variable
}
```
Executing compound actions, .e.g. Decrementing a counter involves fetching the counter value, decrementing it and then writing the updated value for a total of three steps.
Compare and Swap

When the value of a variable depends on another or the new value of a variable depends on its older value.
**option1** : volatile : why not used, : If there's a single thread that writes to the volatile variable and other threads only read the volatile variable then just using volatile is enough, however,
if there's a possibility of multiple threads writing to the volatile variable then "synchronized" would be required to ensure atomic writes to the variable.

If you have a variable say a counter that is being worked on by a thread, it is possible the thread keeps a copy of the counter variable in the CPU cache and manipulates it rather than writing to the main memory

**option2**: locks or synchronized.

```java
synchronized void someMethod(int passedInt) {
   counter = passedInt; // counter is an instance variable
}
```

### Synchronized

Each object in Java has an entity associated with it called the "monitor lock" or just monitor. Think of it as an exclusive lock. Once a thread gets hold of the monitor of an object, it has exclusive access to all the methods marked as synchronized. 
No other thread will be allowed to invoke a method on the object that is marked as synchronized and will block, till the first thread releases the monitor which is equivalent of the first thread exiting the synchronized method.

```java
obj = new A()
```

IllegalMonitorState

## Wait & Notify
Understand the Java wait, notify, and notifyAll methods to control thread synchronization within synchronized blocks. Learn when and how to release or acquire object monitors to coordinate thread execution effectively.

### Reentrant Lock
replaces synchronized blocks or methods, a condition replaces the object monitor methods

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

class Demonstration {

    public static void main(String args[]) throws InterruptedException {
        MissedSignalExample.example();
    }
}

class MissedSignalExample {

    public static void example() throws InterruptedException {

        final ReentrantLock lock = new ReentrantLock();
        final Condition condition = lock.newCondition();

        Thread signaller = new Thread(new Runnable() {

            public void run() {
                lock.lock();
                condition.signal();
                System.out.println("Sent signal");
                lock.unlock();
            }
        });

        Thread waiter = new Thread(new Runnable() {

            public void run() {

                lock.lock();

                try {
                    condition.await();
                    System.out.println("Received signal");
                } catch (InterruptedException ie) {
                    // handle interruption
                }

                lock.unlock();

            }
        });

        signaller.start();
        signaller.join();

        waiter.start();
        waiter.join();

        System.out.println("Program Exiting.");
    }
}
```
he above code when run would time out and show that one of the threads threw an exception. The code is never able to release the semaphore causing the other thread to block forever. Whenever using locks or semaphores, 
remember to unlock or release the semaphore in a finally block. The corrected version appears below.

