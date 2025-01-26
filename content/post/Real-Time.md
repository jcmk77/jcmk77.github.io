---
title: "Real-Time Programming by C++"
date: 2024-01-01T23:42:29-08:00
Description: ""
Tags: []
Categories: []
DisableComments: false
draft: false
---



** Implementation: YMODEM **
Through the implementation of YMODEM, we got hands-on experience with the challenges and solutions required for real-time communication systems. These multiple projects involved creating a protocol that allows for the transfer of files over serial communication, ensuring data integrity and reliability in real-time environments. 
(I felt quite chanllenged by these projects, since I had to deal with the timing and synchronization issues that are inherent in real-time systems.)
<br><br>

**Important topics Explored Through YMODEM** :
<br>
**multithreading** 
YMODEM implementations often involve threads for sending, receiving, and managing errors concurrently.<br>
**synchronization mechanisms**
Synchronizing access to shared resources like buffers or sockets is critical. Mutexes (mutual exclusions) ensure that only one thread accesses a resource at a time.<br>
**atomic operations**
Atomic variables are introduced to manage counters or flags that multiple threads use without risking inconsistent states.<br>
**Condition Variables**
YMODEM implementations often use condition variables to coordinate threads, such as waiting for a packet to be sent before attempting to read the next one.<br>
<br>
**Addition**
Since we didn't use too much of the real-time programming on the hardware (only Z-board), I decided do some additional projects to learn more about it(by STM32)
<br><br>
We have 6 projects in total.(which quite a lot for a semester)<br>
Part1: 
we used C++ to implement a YMODEM protocol for file transfer over serial communication. However, it was not able to handle extra situation like error or interruption.<br><br>
Part2: 
We added more functionalities to the YMODEM protocol, for example, the medium. there are receiver and sender side, which used interfaces to satisfie the requirement. the special WRITE and READ function were implemented to handle the specific error or noise. <br><br>
Part3: 
The one that is most confused. We are supposed to build a real time schedule that manage the task and the priroty, using something like mutax and unique lock, wait() notify() etc. they are so confusing at first because in real tasks everything was working not that straightforward, espcially the Drain function. for satisfying the requirement, we had to build our own MyDrain() function. it was a painful experience but I learned something from it. it inspired me to learn more about the real time programming that is so important if I want to hand on the my own project.<br><br>
Part4: 
we drew the State chart which is a very useful tool to visualize the state of the system.(a good habbit for design a software system I believe). here we were considering the cancellation (from the user) and time out situation. prevent a fatal loss of synchronization or an excessive sequence of errors (including excessive timeouts and an excessively repeated block), it should request the cancelation of the YMODEM session. The request for cancellation should be made when the YMODEM sender should actually be able to receive and process it. 
![Alt text](/images/pics/ENSC351/s1.jpg "Optional Title")
<br><br>
Part5:
it was chill part because we used Z-board to implement a simple task like two terminals communication included UART. (but I like this part, as it was not pure software programming) <br><br>
Part6: 
we basically put everything together and make a final project. this time the whole system was more making sense to me. I think I need to learn more about the real time coding and embedded system because 3 or 4 months is not enough for me to learn all of them. therefore, I will keep learning and start my own project if possible. <br><br>
