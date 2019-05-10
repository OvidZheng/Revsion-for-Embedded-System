 *Chapter 2 Kernel of RTEOS*

# 1 Task Management
## 1.1 What does **Kernel** do?
 * Scheduling
 
## 1.2 What does Kernel schedule?
* Task

### What is Task?
* A task is **executable entity** specialized to perform a special action, which includes:
 * **Sequential Codes**
 * **Relative Data**
 * **Computer Resources**

* A task is an **infinite loop function**. It looks just like any other C function, containing return type and arguments, but it will **never return**. The return type of a task **must** always be declared to be **void**.

### Characteristics of tasks
* Concurrence
* Independence & Asychrony
* Dynamicw

## 1.3 How to describe a task?
### What's the difference between Task and process

```
int i = 1
test()
{
    sleep(10s); \\wait main() to plus 1 over i
    printf("%d", i); 
}

int main()
{
    create(test,......);
    i++
}
```
* If **create( )** function is to create a **task**, the output should be **1**, because each task has a **individual space** under the management of  **MPU (Memory Protection Unit)** and **MMU (Memory Management Unit)**
* In contrast, if it is a **Process**, the output will be **2**.

### The Structure of OS_TCB

```
typedef struct os_tcb {
	OS_STK        *OSTCBStkPtr; 
	   
#if OS_TASK_CREATE_EXT_EN > 0
	void           * OSTCBExtPtr; 
	OS_STK         * OSTCBStkBottom;     
	INT32U         OSTCBStkSize; 
	INT16U         OSTCBOpt; 
	INT16U         OSTCBId; 
#endif

	struct os_tcb *OSTCBNext; 
	struct os_tcb *OSTCBPrev; 
	
#if ((OS_Q_EN > 0) && (OS_MAX_QS > 0)) || (OS_MBOX_EN > 0) || (OS_SEM_EN > 0) || (OS_MUTEX_EN > 0)
	OS_EVENT * OSTCBEventPtr; 
#endif

#if ((OS_Q_EN > 0) && (OS_MAX_QS > 0)) || (OS_MBOX_EN > 0)
	void * OSTCBMsg;           
#endif
	……
	
INT16U  OSTCBDly;
INT8U   OSTCBStat; 
INT8U   OSTCBPrio;

INT8U   OSTCBX; 
INT8U   OSTCBY; 
INT8U   OSTCBBitX; 
INT8U   OSTCBBitY; 

#if OS_TASK_DEL_EN > 0
	BOOLEAN        OSTCBDelReq; 
#endif
} OS_TCB;	



```
### OSTCBList

```
OS_ENTER_CRITICAL();
OSTCBPrioTbl[prio] = ptcb;
ptcb->OSTCBNext = OSTCBList;                  
ptcb->OSTCBPrev = (OS_TCB *)0;
if (OSTCBList != (OS_TCB *)0)
{
	OSTCBList->OSTCBPrev = ptcb;
}
OSTCBList = ptcb;
OSRdyGrp |= ptcb->OSTCBBitY;        
OSRdyTbl[ptcb->OSTCBY] |= ptcb->OSTCBBitX;
OS_EXIT_CRITICAL();
return (OS_NO_ERR);
}
OS_EXIT_CRITICAL();
return (OS_NO_MORE_TCB);}

```

## How to create a task in us/OS II?

```
INT8U  OSTaskCreate (void (*task)(void *p_arg), void *p_arg, OS_STK *ptos,INT8U prio)
//task:指针函数，指向任务所开始的函数，当任务被第一次调度运行时，将会从函数处开始运行
//p_arg:传给task的参数指针
//ptos:堆栈指针
//prio:进程优先级
{
	OS_STK * psp;
	
	//为寄存器分配空间	
	#if OS_CRITICAL_METHOD == 3               
		OS_CPU_SR  cpu_sr = 0;
	#endif
	
	OS_ENTER_CRITICAL();//进入中断   
	
	if (OSIntNesting > 0) //判断中断嵌套层数
	{
		OS_EXIT_CRITICAL();
		return (OS_ERR_TASK_CREATE_ISR);
	}
	
	if (OSTCBPrioTbl[prio] == (OS_TCB *)0)
	//判断有没有任务占用该优先级
	{
		OSTCBPrioTbl[prio] = (OS_TCB *)1;   
		OS_EXIT_CRITICAL();
		
		psp = (OS_STK *)OSTaskStkInit(task, pdata, ptos, 0); //初始化任务栈
		err = OS_TCBInit(prio, psp, (OS_STK *)0, 0, 0, (void *)0, 0);//初始化TCB
		
		if (err == OS_NO_ERR) 
		{
			OS_ENTER_CRITICAL();
			OSTaskCtr++;                                      
			OS_EXIT_CRITICAL();
			if (OSRunning == TRUE) 
			{   
				OS_Sched();//调度任务
			}
		else 
		{
			OS_ENTER_CRITICAL();
			OSTCBPrioTbl[prio] = (OS_TCB *)0;
			OS_EXIT_CRITICAL();
		}
		return (err);
	}
	OS_EXIT_CRITICAL();
	return (OS_PRIO_EXIST);
}

```

### The initiate of stack

```
OS_STK *OSTaskStkInit (void (*task)(void *pd), void *p_arg, OS_STK *ptos, INT16U opt)
//task:任务执行函数入口
//p_arg:任务参数地址
//ptos:栈顶指针
//opt:预留参数
{
	OS_STK *stk;                                                         
	opt    = opt; 
	stk    = ptos;                                  
	*(stk)  = (OS_STK)task;       /* Entry Point*/                               
	*(--stk) = (INT32U)0;         	/*LR */                                     
	*(--stk) = (INT32U)0;         	/*R12*/                                     
	*(--stk) = (INT32U)0;         	/*R11*/
	*(--stk) = (INT32U)0;         	/*R10*/
	*(--stk) = (INT32U)0;         	/*R9*/
	*(--stk) = (INT32U)0;         	/*R8*/
	*(--stk) = (INT32U)0;         	/*R7*/
	*(--stk) = (INT32U)0;         	/*R6*/
	*(--stk) = (INT32U)0;         	/*R5*/
	*(--stk) = (INT32U)0;         	/*R4*/    
	*(--stk) = (INT32U)0;         	/*R3*/
	*(--stk) = (INT32U)0;         	/*R2*/
	*(--stk) = (INT32U)0;         	/*R1*/
	*(--stk) = (INT32U)p_arg;		//压入参数环境
	*(--stk) = (INT32U)0x 0x00000013L; //压入处理器状态寄存器CPSR                              
	return (stk);
}

```

### The initiation of TCB

```
INT8U  OS_TCBInit (INT8U prio, OS_STK *ptos, OS_STK *pbos, INT16U id, INT32U stk_size, void *pext, INT16U opt)
//prio进程优先级
//ptos栈顶指针
//pbos栈底指针
//id任务标识符
//stk_size栈容量
//opt选择项
{
#if OS_CRITICAL_METHOD == 3
	OS_CPU_SR  cpu_sr;
#endif
    
	OS_TCB    *ptcb;//申请一个指向TCB的指针
	
	OS_ENTER_CRITICAL();
	
	ptcb = OSTCBFreeList;//从空闲列获取一个TCB 
							   
	if (ptcb != (OS_TCB *)0) {
		OSTCBFreeList = ptcb->OSTCBNext;//修改空想列列首，使其指向下一项
		
		OS_EXIT_CRITICAL();
		
		ptcb->OSTCBStkPtr= ptos;
		ptcb->OSTCBPrio = (INT8U)prio;
		ptcb->OSTCBStat = OS_STAT_RDY;
		ptcb->OSTCBDly = 0;
		pext = pext;
		stk_size = stk_size;
		pbos = pbos;
		opt = opt;
		id = id;

		ptcb->OSTCBY = prio >> 3;
		ptcb->OSTCBBitY = OSMapTbl[ptcb->OSTCBY];
		ptcb->OSTCBX = prio & 0x07;
		ptcb->OSTCBBitX = OSMapTbl[ptcb->OSTCBX];
		
		OS_ENTER_CRITICAL();
		
		OSTCBPrioTbl[prio] = ptcb;
		ptcb->OSTCBNext = OSTCBList;
		ptcb->OSTCBPrev = (OS_TCB *)0;
		if (OSTCBList != (OS_TCB *)0) 
		{
			OSTCBList->OSTCBPrev = ptcb;
		}
		OSTCBList = ptcb;
		OSRdyGrp |= ptcb->OSTCBBitY;
		OSRdyTbl[ptcb->OSTCBY] |= ptcb->OSTCBBitX;
		
		OS_EXIT_CRITICAL();
		return (OS_NO_ERR);
	}
	OS_EXIT_CRITICAL();
	return (OS_NO_MORE_TCB);
}

```

## 1.4 How to schedule tasks in the ready queue?

```
void  OS_Sched (void)
{
	#if OS_CRITICAL_METHOD == 3
	OS_CPU_SR  cpu_sr = 0;
	#endif
	
	OS_ENTER_CRITICAL();
	
	if (OSIntNesting == 0)
	{
		if (OSLockNesting == 0)
		{
			OS_SchedNew();
			
			if (OSPrioHighRdy != OSPrioCur)
			{
				OSTCBHighRdy = OSTCBPrioTbl[OSPrioHighRdy];
				
				#if OS_TASK_PROFILE_EN > 0
				OSTCBHighRdy->OSTCBCtxSwCtr++;
				#endif
				
				OSCtxSwCtr++;
				OS_TASK_SW();
			}
		}
	}
	OS_EXIT_CRITICAL();
}

```

### OS_SchedNew

```
static  void  OS_SchedNew (void)
{
#if OS_LOWEST_PRIO <= 63  
	INT8U  y;
	
	y= OSUnMapTbl[OSRdyGrp];           
	OSPrioHighRdy = (INT8U)((y << 3) + OSUnMapTbl[OSRdyTbl[y]]);
	
#else  
	INT8U   y;
	INT16U *ptbl;
	if ((OSRdyGrp & 0xFF) != 0) 
	{
		y = OSUnMapTbl[OSRdyGrp & 0xFF];
	} 
	else 
	{
		y = OSUnMapTbl[(OSRdyGrp >> 8) & 0xFF] + 8;
	}
	ptbl = &OSRdyTbl[y];
	if ((*ptbl & 0xFF) != 0) 
	{
		OSPrioHighRdy = (INT8U)((y << 4) + OSUnMapTbl[(*ptbl & 0xFF)]);
	} else 
	{
		OSPrioHighRdy = (INT8U)((y << 4) + OSUnMapTbl[(*ptbl >> 8) & 0xFF] + 8);
	}
#endif
}

```

## 1.5 How to switch tasks
* When a kernel decides to run a different task, it simply saves the current task’s context (CPU registers) in the current task’s context storage area (Current task’s stack). Once this operation is performed, the new task’s context is restored from its storage area (New task’s stack) and then resumes execution of the new task’s code. This process is called a context switch.


```
		
OSCtxSw:
	STMFD	          SP!, {LR}                ;PC
	STMFD	          SP!, {R0-R12, LR}    ;R0-R12 LR                                     
	MRS	          R0,  CPSR               ;Push CPSR                           
	STMFD	          SP!, {R0}	          
		
	;----------------------------------------------------------------------------------
	; 		OSTCBCur->OSTCBStkPtr = SPa
	;----------------------------------------------------------------------------------		LDR		R0, =OSTCBCur
	LDR		R0, [R0]
	STR		SP, [R0]
	
	BL 		OSTaskSwHook                                                 
	;---------------------------------------------------------------------------------
	; OSTCBCur = OSTCBHighRdy;
	;----------------------------------------------------------------------------------
	LDR		R0, =OSTCBHighRdy
	LDR		R1, =OSTCBCur
	LDR		R0, [R0]
	STR		R0, [R1]
	//LDR将R0所指向寄存器中的值存放到R0中
	//STR将R0放入R1所指向的寄存器中
	
	;---------------------------------------------------------------------------------
	; OSPrioCur = OSPrioHighRdy;
	;---------------------------------------------------------------------------------
	LDR	          R0, =OSPrioHighRdy
	LDR	          R1, =OSPrioCur
	LDRB	          R0, [R0]
	STRB	          R0, [R1] 	
	;----------------------------------------------------------------------------------
	;  OSTCBHighRdy->OSTCBStkPtr;
	;----------------------------------------------------------------------------------
	LDR		R0, =OSTCBHighRdy                                            
	LDR		R0, [R0]
	LDR		SP, [R0]
	;----------------------------------------------------------------------------------
	;Restore New task context
	;----------------------------------------------------------------------------------
	LDMFD 	           SP!, {R0}	;POP CPSR       
	MSR 	           SPSR_cxsf, R0                                                    
	LDMFD 	           SP!, {R0-R12, LR, PC}^	 

```

## 1.6 With events trigger the context switch?
## 1.7 Critical section accessing during scheduling

```
Cli                                                                         
	MRS     R0,CPSR 
	ORR     R1,R0,#0x80
	//16进制的80在2进制下为1000 0000
	//此处与R1取或操作，把第8位置1，不再接受中断
	MSR     CPSR_c,R1
	MOV    PC,LR 

Sti
	MRS     R0,CPSR
	AND     R1,R0,#0xffffff7f 
	//16进制的7f在2进制下为0111 1111
	//与R1取与操作，将第8位置为0，接收中断
	MSR     CPSR_c,R1
	MOV     PC,LR
```

```
//改为使用栈来储存原有CPSR
//可以直接取出复原
PushAndCli
     MRS       R0,CPSR_c  
     STMFD   sp!,R0 
     ORR       R1,R0,#0x80 
     MSR      CPSR_c,R1 
     MOV      PC,LR

Pop
     LDMFD   sp!,R0
     MSR      CPSR_c,R0
```

```
//使用一个寄存器来保存原有CPSR
//可以直接取出复原
OSCPUSaveSR
	MRS     R0, CPSR
	ORR     R1, R0, #0x80 
	MSR     CPSR_c, R1
	MOV     PC, LR			     

OSCPURestoreSR
	MSR     CPSR_c, R0
	MOV     PC, LR
```

## 1.8 Interruption
### Timetick

```
void  OSTimeTick (void)
{
#if OS_CRITICAL_METHOD == 3 
	OS_CPU_SR  cpu_sr;
#endif    
	OS_TCB    *ptcb;
	if (OSRunning == TRUE) 
	{    
		ptcb = OSTCBList;  
		while (ptcb->OSTCBPrio != OS_IDLE_PRIO) 
		{ 
			OS_ENTER_CRITICAL();
			if (ptcb->OSTCBDly != 0) 
			{ 
				if (--ptcb->OSTCBDly == 0) 
				{ 
					if ((ptcb->OSTCBStat & OS_STAT_SUSPEND) == OS_STAT_RDY)
					{
						OSRdyGrp |= ptcb->OSTCBBitY; 
						OSRdyTbl[ptcb->OSTCBY] |= ptcb->OSTCBBitX;
					} 
					else 
					{ 
						ptcb->OSTCBDly = 1;
					}   
				}
			}
			ptcb = ptcb->OSTCBNext; 
			OS_EXIT_CRITICAL();
		}
	}
}

```
### Case 1: interruption on ARM 2440

```
IRQ
	sub			lr,lr,#4  
	stmfd		sp!,{r0.r12,lr}  
	mrs			r0,spsr
	stmfd		sp!,{r0}
		
	ldr			r0,=INTOFFSET              ;10:  0X28
	ldr			r0,[r0]
	ldr			r1,=HandlerEINT0        ;0x33FFFF20
	add			r1,r1,r0,lsl #2 
	ldr			r1,[r1]
	 
	mov 	  	lr,pc
	 
	mov			pc,r1
		
	ldmfd		sp!,{r0}
	msr			spsr_cxsf,r0
	ldmfd		sp!,{r0.r12,lr}
	movs		pc,lr

```

### Interruption with us/OS II running

```
IRQ
	stmdb	sp!,{r0-r2}                      ;sp_irq 
	mov		r0,sp 
	add		sp,sp,#12 
	sub		r1,lr,#4                             ;lr_irq 
	mrs		r2,spsr  

	msr		 cpsr_cxsf,#SVCMODE|NOINT  
	stmdb	 sp!,{r1}                         ;sp_svc 
	stmdb    sp!,{r3-r12,lr} 
	ldmia    r0!,{r3-r5}   
	stmdb	 sp!,{r2-r5}
	
	ldr		r0,=OSIntNesting 
	ldr		r1,[r0]
	add		r1,r1,#1 
	strb    r1,[r0]
	teq		r1,#1
	bne		%F1  

	ldr		r0,=OSTCBCur 
	ldr		r0,[r0]
	str		sp,[r0]
	
	msr	   psr_c,#IRQMODE|NOINT
	ldr		r0,= INTOFFSET                    ;10:    0x28
	ldr		r0,[r0]
	ldr		r1,=HandleEINT0      ;0x33FFFF20
			
	mov		lr,pc 
	ldr		pc,[r1,r0,lsl#2]
	msr		cpsr_c,#SVCMODE|NOINT 
	bl		OSIntExit

	ldr		r0,[sp],#4
	msr		spsr_cxsf,r0 
	ldmia   sp!,{r0-r12,lr,pc}^

```
### OSIntExit

```
void  OSIntExit (void)	
{
#if OS_CRITICAL_METHOD == 3
	OS_CPU_SR  cpu_sr;
#endif
	
	if (OSRunning == TRUE) 
	{
		OS_ENTER_CRITICAL();
		if (OSIntNesting > 0) 
		{		
			OSIntNesting--;
		}    
		If ((OSIntNesting == 0) && (OSLockNesting == 0)) 
		{
			OSIntExitY= OSUnMapTbl[OSRdyGrp];
			OSPrioHighRdy=(INT8U)((OSIntExitY<<3)+ OSUnMapTbl[OSRdyTbl[OSIntExitY]]);
			if (OSPrioHighRdy != OSPrioCur) 
			{
				OSTCBHighRdy= OSTCBPrioTbl[OSPrioHighRdy];
				OSCtxSwCtr++;
				OSIntCtxSw();	
			}    
		}    
		OS_EXIT_CRITICAL();
	}    
}

```

## 1.9 How to schedule task to ensure that their deadlines are met?
* Tasks and their special requirements 
 * Task number: n 
 * Task:                       Ti    ( i=1,2,…,n )
 * Period:                    Pi
 * Executing time:        ei
 * Relative deadline:     Di
 * Absolute deadline:    di
 * Release time:           ri 
 * Phasing:                  Ii

### Priority Inversion & Priority Inheritance
* Priotiry inheritance：If a higher-priority task TH is blocked by a lower-priority task TL, (Because TL is currently executing a critical section needed by TH), the lower-priority task TL temporarily inherits the priority of TH . When the block ceases, TL resumes its original priority.

### Priority Celling
* Priotiry ceiling: Priotiry celling is a priority defined for a shared resource si, the priority celling of si is the highest prority of any task that may access it.

### RM Scheduling Policy
* Any set of n periodic tasks that fully utilizes the processor under the RM scheduling policy must have, for each i ∈ {1,2,…n} 
* e1/P1+ e2/P2+…+ ei/Pi+ bi/Pi <= i (2(1/i)-1)

### EDF (Earliest Deadline First) Policy                             
* Schedule the task whose absolute deadline is the earliest
* No periodic assumption          
* Necessary and sufficient conditions for schedulability
* the sum of ei/di <= 1

## 2 Mutex, Synchronization, Communication
### 2.1 Semaphore for Mutex

```
OS_EVENT *OSSemCreate (INT16U cnt)
{
	OS_EVENT *pevent;
	pevent = OSEventFreeList;                             //Get next free event control block(ECB)
	if (OSEventFreeList != (OS_EVENT *)0) {     //See if pool of free ECB pool was empty
	OSEventFreeList = (OS_EVENT *)OSEventFreeList->OSEventPtr;
	}
	if (pevent != (OS_EVENT *)0) {                     //Get an ECB and initialize it
		pevent->OSEventType = OS_EVENT_TYPE_SEM; //Event type is Semaphore
		pevent->OSEventCnt = cnt;                 //Set semaphore value
		pevent->OSEventPtr = (void *)0;        // Unlink from ECB free list
		OS_EventWaitListInit(pevent);             //Initialize to 'nobody waiting' on Sem
	}
	return (pevent); 
}

```


```
void  OS_EventWaitListInit (OS_EVENT *pevent)
{
#if OS_LOWEST_PRIO <= 63
		INT8U  *ptbl;
#else
		INT16U *ptbl;
#endif
		INT8U   i;

		pevent->OSEventGrp = 0;                      
		ptbl  = &pevent->OSEventTbl[0];

	   for (i = 0; i < OS_EVENT_TBL_SIZE; i++) {
		   *ptbl++ = 0;
	}
}

```

### 1.2 Semaphore for counting

```
Producer

do 
{
	…
	Produce a new item
	…
	Apply empty
	Apply mutex
	…
	Add the new item to the queue
	…
	Free mutex
	Free full
} while (1);

```


```
Consumer

do 
{
	Apply full
	Apply mutex
	…
	Move a item from the queue
	…
	Free mutex
	Free empty
	…
	Consume the newly-moved item
	…
} while (1); 

```
## Memory Management

## Interruption & Time Management
### TimeDly

```
void  OSTimeDly (INT16U ticks)
{                                                         
		#if OS_CRITICAL_METHOD == 3
					OS_CPU_SR  cpu_sr;
		#endif    

		if (ticks > 0)
		{ OS_ENTER_CRITICAL();
							 if ((OSRdyTbl[OSTCBCur->OSTCBY] &= ~OSTCBCur->OSTCBBitX) == 0) 
									{   OSRdyGrp &= ~OSTCBCur->OSTCBBitY;    }
							 OSTCBCur->OSTCBDly = ticks; 
				OS_EXIT_CRITICAL();
				OS_Sched(); 
		}
}

```