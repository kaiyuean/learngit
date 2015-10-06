## CSCI 5828 – Fall 2015
## Homework 3
## Kaiyue An & Yun Zhou

<hr>

### Question 1

**Using the material that we have covered in Lectures 8, 9, and 10, explain why the broken program doesn't work. What concurrency-related problems are occuring in this program? If you see the program end in livelock, then describe what is happening with the threads. Why can't they make progress? If you see the program end in another way, such as getting to the point where it prints out the product ids but doesn't include all of them, explain why you think that happened. Note: If you start to add println statements to the Producers and Consumers, you may actually altar the behavior of the program! If you observe this, then also include a discussion on why that happens as well. In your answer, if you want to include snippets of code and/or output to explain what you are seeing, then do so. Use all of Markdown's capabilities to display what you need to explain the concurrency-related problems that you are observing.**

The code describes twenty processes, ten producers and ten consumers, who share a common, fixed-size buffer used as a queue. The producers' role is to generate a piece of data, put it into the queue and start again. At the same time, the consumers are consuming the data (removing it from the queue) one piece at a time. The producers and consumers execute concurrently. The problem is that we have to make sure that the producers won't try to add data into the queue if it's full and that the consumers won't try to remove data from an empty queue. 

However, several concurrency-related problems occurring here: **race condition, memory visibility, starvation and livelock**. 

**Race condition**: 
* We may have a race condition if two or more threads are accessing the same variables. For example, consider producer A and producer B are trying to increase the same product id at the same time. Producer A thread has higher priority, so it gets the id=0 and and adds 1 to the id. Then producer B wakes up and gets the id(which is still 0) and adds 1 to the it. 
* We may also have a race condition when two(or more) producers want to add products to the queue at the same time and when two(or more) consumers want to remove products from the queue at the same time. 

**Livelock**: 
* Livelock occurs in the broken code when a thread continuously attempts an action that fails. A thread often acts in response to the action of another thread. If the other thread's action is also a response to the action of another thread, then livelock may result. As with deadlock, livelocked threads are unable to make further progress. However, the threads are not blocked — they are simply too busy responding to each other to resume work.
* In this program livelock occurs in several situations. For example, it occurs when the message says"Too many items in the queue". The reason is that if the queue size is 9, producer A thread adds a new product. At the same time, producer B thread wakes up and adds a new product to make the queue size to be 10. The previous thread still takes the queue size as 9 and adds product into it, which makes the queue size to be 11. **This is also memory visibility problem between producers**. Moreover, as long as the consumer has a priority lower than that of the producers, they may never be scheduled by the JVM to run and therefore may never be able to consume an item and free up buffer space for the producer. **In this situation, the producers are livelocked waiting for the consumer to free buffer space.** This situation will have outcome as below: 
```
  ...
  Too many items in the queue: 12!
  Too many items in the queue: 12!
  Too many items in the queue: 12!
  ...
```
The vise-verse situation which illustrates the **starvation** : 
```
  ...
  Queue empty!
  Queue empty!
  Queue empty!
  ...
```
For the last question, I did not observe this situation. But I found this problem in the fixing process. I guess the reason is because of the **race condition** between producers. Multiple producers are trying to produce a new product at the same time and they share the same ID. 

<hr>

### Question 2
**Now switch your attention to the broken2 program. The only difference between the two programs are the synchronized keywords on the methods contained in ProductionLine.java. For this question, explain why this approach to fixing the program failed. Why is it that synchronizing these methods is not enough. What interactions between the threads are still occuring that cause the program to not be able to produce the correct output? Again, you may use snippets of code and/or output to illustrate your points. The only requirement for this question is that you focus exclusively on issues related to why this particular approach fails to solve the problem. In other words, your answer to this question should be different than your answer to the question above where you are discussing the program and its concurrency problems in general.**

Broken2 program adds synchronized to the methods(size(), append(), retrieve())contained in the ProductionLine.java. Thus when two producers want to add product to the queue at the same time, use synchronized can make sure only one producer add product to the queue each time. When two consumers want to remove product from the queue at the same time, use synchronized can make sure only one cunsomer remove product from the queue each time. Adding synchronized to method size() can make sure when two(or more) producers or two(or more) consumers want to get the size of the queue, they can be synchronized thus they won't get the same size at the same time. 

This program version resolves these problems stated in Question 1: 
* Race condition between producers and producers when two(or more) producers want to add products to the queue at the same time.
* Race condition between consumers and consumers when two(or more) consumers want to remove products from the queue at the same time. 

However, there still exist two problems. 
* The first problem is when a producer want to add a product to the queue, if the queue if full, the producer will always wait and can't recover from the wait status. When a consumer want to retrive a product from the queue, if the queue is empty, the consumer will always wait and can't recover from the wait status. The reason it fails is that using keyword "synchronized" can only achieve *intrinsic lock*. However, a thread that's blocked on an intrinsic lock is not interruptible, we have no way to recover. When the queue is full, the producer is still trying to append products into it and consumer is not notified to retrieve product from it; and when the queue is empty, the consumer is still trying to remove products from it and producer is not notified to add product into it. There is no way to interrupt them. 
* The second problem is that there still exists race condition between two different producers when they want to make new products and they maybe get the same id for different products. Thus we maybe have two or more products have the same id.

<hr>

### Question 3
**Now turn your attention to creating an implementation of the program that functions correctly in the fixed directory. In your answer to this question, you should discuss the approach you took to fix the problem and get your version of the program to generate output that is similar to the example_output.txt file that is included with the repo.**

To solve the first problem of the broken2 program, we use wait() and notifyAll() methods in the append() and retrieve() methods in ProductionLine.java file. When the queue size is full, producer thread will run into wait status and release the lock. But when the queue is not full, it can add product to the queue, **and notify all other waiting threads to wake up and execute**. When the queue size equals to 0, consumer thread will run into wait status and release the lock . But If the queue is not empty, then they can retrieve products from the queue, **and notify all other waiting threads to wake up and execute**. 

The following are the changed code:
```
	public synchronized void append(Product p) {
       while(products.size() == 10 ) {
           try {
               wait();
           }
           catch (InterruptedException e) {
               e.printStackTrace();
           }
       }
       products.add(p);
       this.notifyAll();
   }

   public synchronized Product retrieve() {
       while(products.size() == 0 ) {
           try {
               wait();
           }
           catch (InterruptedException e) {
               e.printStackTrace();
           }
       }
       this.notifyAll();
       return products.remove(0);
    }
```
To solve the second problem of race condition between two different producers, we add synchronized(queue){...} before making a new production. In this case when two or more producers want to make new products and they won't get the same id at the same time. One producer will be allowed to make product and other producers will wait until this producer come out. The following are the changed code:
```
	while (count < 20) {
      if (queue.size() < 10) {
          synchronized(queue){
              Product p = new Product();
              String msg = "Producer %d Produced: %s on iteration %d";
              System.out.println(String.format(msg, id, p, count));
              queue.append(p);
          }
          count++;
        }
    }
```









