# Chapter 1: Introduction
## What are Embedded System
* Any device include a **computer**

## What are Real Time System(RTS)?
* Any system where a **timely** response by computer to the external event is **vital** is **Real-Time System**

### Hard RTS
* Systems in which something really **bad** will happen if the System does not deliver its output before the **deadline**.

### Soft RTS
* Systems in which **nothing** Catastrophic will happen if the deadlines are **missed**. But the performance will be **degraded** under an acceptable scope.

 
## What are RTES special in?
* Application Specify
 * **specialize** and **optimize** the design for specific application.
* Have to worry about both **Hardware** and **Software**
* Have to worry about **non-functional constrains**
 * Real-time
 * Memory
 * Cost
 * power
 * Reliability

### Popular Architecture of RTES
#### Polling Loop

```
while(true)
{
    if(A) { F1() };
    if(B) { F2() };
    if(C) { F3() };
}
```

#### Foreground/Back(Interurpt)
#### Multitask
