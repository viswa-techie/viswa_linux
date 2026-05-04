# Chapter 10: Inter-Task Communication (IPC)

## Learning Goals
- Master RTOS IPC mechanisms: queues, mailboxes, message buffers, stream buffers
- Understand implementation internals and memory layout
- Know when to use each IPC mechanism
- Learn ISR-safe IPC patterns
- Compare IPC across FreeRTOS, Zephyr, QNX, VxWorks

---

## 1. IPC Overview

```
  RTOS IPC Mechanisms
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Data-Passing IPC:                                       │
  │  ┌──────────────────────────────────────────────────┐    │
  │  │ Queue:          fixed-size items, FIFO            │    │
  │  │ Mailbox:        single-item overwrite (Zephyr)    │    │
  │  │ Message Buffer: variable-size messages             │    │
  │  │ Stream Buffer:  byte-stream (no message boundary) │    │
  │  │ Pipe:           byte-stream (QNX/VxWorks)         │    │
  │  └──────────────────────────────────────────────────┘    │
  │                                                           │
  │  Signaling IPC (no data, just wake-up):                  │
  │  ┌──────────────────────────────────────────────────┐    │
  │  │ Binary Semaphore:  signal event occurred          │    │
  │  │ Counting Semaphore: count available resources     │    │
  │  │ Task Notification:  lightweight direct signal     │    │
  │  │ Event Group:        multiple event flags          │    │
  │  └──────────────────────────────────────────────────┘    │
  │                                                           │
  │  Selection Guide:                                        │
  │  ┌─────────────────┬───────────────────────────────┐    │
  │  │ Use Case        │ Best IPC                       │    │
  │  ├─────────────────┼───────────────────────────────┤    │
  │  │ ISR → Task wake │ Task notification (fastest)    │    │
  │  │ ISR → Task data │ Queue or Stream buffer         │    │
  │  │ Task → Task msg │ Queue (fixed) or MsgBuf (var)  │    │
  │  │ Multiple events │ Event group                    │    │
  │  │ Resource count  │ Counting semaphore             │    │
  │  │ Byte stream     │ Stream buffer or pipe          │    │
  │  │ Producer-consumer│ Queue                         │    │
  │  │ Latest-value    │ Queue (len=1 overwrite)        │    │
  │  └─────────────────┴───────────────────────────────┘    │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Queues

```c
/* FreeRTOS Queue — primary data-passing IPC */

/*
 * Queue Memory Layout:
 *
 * ┌─ QueueHandle_t ─────────────────────────────────────┐
 * │  pcHead ───────────────► ┌────────┐ Item 0          │
 * │  pcWriteTo              │ N bytes │                  │
 * │  union { pcReadFrom }   ├────────┤ Item 1          │
 * │  xTasksWaitingToSend    │ N bytes │                  │
 * │  xTasksWaitingToReceive ├────────┤ Item 2          │
 * │  uxMessagesWaiting      │ N bytes │                  │
 * │  uxLength (max items)   ├────────┤ ...             │
 * │  uxItemSize (N bytes)   │ N bytes │                  │
 * │  cRxLock, cTxLock       └────────┘ Item (len-1)    │
 * │  ucQueueType            (circular buffer)           │
 * └─────────────────────────────────────────────────────┘
 *
 * Items are copied BY VALUE into queue storage (not by reference)
 * This is safe: sender's local variable can go out of scope
 */

/* Create a queue: 10 items, each 4 bytes (uint32_t) */
QueueHandle_t xSensorQueue = xQueueCreate(10, sizeof(uint32_t));

/* Send to queue (from Task) */
uint32_t sensorValue = adc_read();
if (xQueueSend(xSensorQueue, &sensorValue, pdMS_TO_TICKS(100)) != pdPASS) {
    /* Queue full after 100ms timeout — handle error */
    error_log("Sensor queue full");
}

/* Send to queue (from ISR) */
void ADC_IRQHandler(void) {
    BaseType_t xWoken = pdFALSE;
    uint32_t value = ADC1->DR;

    xQueueSendFromISR(xSensorQueue, &value, &xWoken);
    portYIELD_FROM_ISR(xWoken);
}

/* Receive from queue */
void vProcessingTask(void *pvParameters) {
    uint32_t rxValue;
    for (;;) {
        /* Block until data available */
        if (xQueueReceive(xSensorQueue, &rxValue, portMAX_DELAY) == pdPASS) {
            process_sensor_reading(rxValue);
        }
    }
}

/* Queue variants */
xQueueSendToFront(queue, &item, timeout);  /* LIFO push */
xQueueSendToBack(queue, &item, timeout);   /* FIFO push (same as xQueueSend) */
xQueuePeek(queue, &item, timeout);         /* Read without removing */
xQueueOverwrite(queue, &item);             /* Overwrite (queue length must be 1) */
uxQueueMessagesWaiting(queue);             /* Check count without blocking */
```

---

## 3. Queue Internals: Blocking and Unblocking

```
  Queue Blocking Mechanism
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  xQueueSend() when queue FULL:                           │
  │  1. Task added to xTasksWaitingToSend list               │
  │  2. Task removed from Ready list, added to Blocked list  │
  │  3. If timeout specified, added to delayed task list too  │
  │  4. Scheduler runs next highest-priority ready task       │
  │  5. When space available: blocked task moved to Ready     │
  │                                                           │
  │  xQueueReceive() when queue EMPTY:                       │
  │  1. Task added to xTasksWaitingToReceive list             │
  │  2. Task blocked (same mechanism as above)               │
  │  3. When item arrives: blocked task unblocked             │
  │  4. If multiple waiters: highest-priority one wakes first│
  │                                                           │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ Task A (prio 3) ──xQueueReceive──► [BLOCKED]  │      │
  │  │ Task B (prio 5) ──xQueueReceive──► [BLOCKED]  │      │
  │  │                                                 │      │
  │  │ Task C sends to queue:                          │      │
  │  │   → Task B wakes (higher priority = 5)          │      │
  │  │   → Task A remains blocked                      │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  Queue Set: wait on multiple queues simultaneously       │
  │  QueueSetHandle_t xSet = xQueueCreateSet(totalSize);     │
  │  xQueueAddToSet(queue1, xSet);                           │
  │  xQueueAddToSet(queue2, xSet);                           │
  │  QueueSetMemberHandle_t active = xQueueSelectFromSet(    │
  │      xSet, portMAX_DELAY);                               │
  │  /* 'active' is the queue/semaphore that has data */     │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. Message Buffers and Stream Buffers

```c
/* Stream Buffer — byte-stream, no message boundaries */
/* Good for: UART data, continuous sensor streams */

StreamBufferHandle_t xStream = xStreamBufferCreate(
    256,    /* Total buffer size in bytes */
    1       /* Trigger level: unblock receiver after 1 byte */
);

/* Send bytes (from ISR) */
void UART_IRQHandler(void) {
    BaseType_t xWoken = pdFALSE;
    uint8_t byte = UART1->RDR;

    xStreamBufferSendFromISR(xStream, &byte, 1, &xWoken);
    portYIELD_FROM_ISR(xWoken);
}

/* Receive bytes (in task) */
void vUartTask(void *pvParameters) {
    uint8_t buf[64];
    for (;;) {
        size_t rxBytes = xStreamBufferReceive(
            xStream, buf, sizeof(buf), pdMS_TO_TICKS(100));
        if (rxBytes > 0) {
            process_uart_data(buf, rxBytes);
        }
    }
}

/* Message Buffer — variable-size messages with boundaries */
/* Good for: command packets, protocol frames */

MessageBufferHandle_t xMsgBuf = xMessageBufferCreate(512);

/* Send a complete message */
typedef struct {
    uint8_t  cmd;
    uint16_t length;
    uint8_t  data[32];
} Command_t;

Command_t cmd = { .cmd = CMD_START, .length = 5, .data = {1,2,3,4,5} };
size_t sent = xMessageBufferSend(
    xMsgBuf, &cmd, sizeof(cmd), pdMS_TO_TICKS(100));

/* Receive a complete message (preserves boundaries) */
Command_t rxCmd;
size_t rxSize = xMessageBufferReceive(
    xMsgBuf, &rxCmd, sizeof(rxCmd), portMAX_DELAY);
/* rxSize = exact size of message that was sent */

/*
 * Message Buffer internals:
 * Each message stored as: [4-byte length header][message data]
 *
 * Buffer memory:
 * ┌──────┬───────────┬──────┬──────────┬──────┬─────┐
 * │len=5 │ 5 bytes   │len=12│ 12 bytes │len=3 │ ... │
 * └──────┴───────────┴──────┴──────────┴──────┴─────┘
 *
 * Limitation: single-reader, single-writer only!
 * For multiple readers/writers, use queues.
 */
```

---

## 5. Event Groups

```c
/* Event Groups — multiple event flags in a single 32-bit word */
/* Good for: synchronization on multiple conditions */

#define EVT_SENSOR_READY  (1 << 0)   /* Bit 0 */
#define EVT_CAN_RX        (1 << 1)   /* Bit 1 */
#define EVT_BUTTON_PRESS  (1 << 2)   /* Bit 2 */
#define EVT_TIMER_EXPIRED (1 << 3)   /* Bit 3 */

EventGroupHandle_t xEvents = xEventGroupCreate();

/* Set bits from tasks */
xEventGroupSetBits(xEvents, EVT_SENSOR_READY);

/* Set bits from ISR (uses timer daemon task internally) */
void Button_IRQHandler(void) {
    BaseType_t xWoken = pdFALSE;
    xEventGroupSetBitsFromISR(xEvents, EVT_BUTTON_PRESS, &xWoken);
    portYIELD_FROM_ISR(xWoken);
}

/* Wait for ANY of multiple events (OR) */
EventBits_t bits = xEventGroupWaitBits(
    xEvents,
    EVT_SENSOR_READY | EVT_CAN_RX,     /* Bits to wait for */
    pdTRUE,                              /* Clear bits on exit */
    pdFALSE,                             /* Wait for ANY (OR) */
    pdMS_TO_TICKS(500)                   /* Timeout */
);
if (bits & EVT_SENSOR_READY) { /* sensor event */ }
if (bits & EVT_CAN_RX) { /* CAN event */ }

/* Wait for ALL events (AND) — rendezvous point */
bits = xEventGroupWaitBits(
    xEvents,
    EVT_SENSOR_READY | EVT_CAN_RX | EVT_TIMER_EXPIRED,
    pdTRUE,                              /* Clear on exit */
    pdTRUE,                              /* Wait for ALL (AND) */
    portMAX_DELAY
);
/* Only returns when ALL three bits are set */

/* Synchronization barrier (rendezvous) */
EventBits_t syncBits = xEventGroupSync(
    xEvents,
    EVT_TASK_A_DONE,        /* Bits this task sets */
    EVT_TASK_A_DONE | EVT_TASK_B_DONE | EVT_TASK_C_DONE,  /* Wait for all */
    portMAX_DELAY
);
/* All three tasks reach this point before any proceeds */
```

---

## 6. IPC Across RTOS Platforms

```
  Zephyr IPC
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  k_msgq: Message Queue (fixed-size items, like FreeRTOS) │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ K_MSGQ_DEFINE(my_msgq, sizeof(DataItem), 10, 4);│     │
  │  │ k_msgq_put(&my_msgq, &item, K_MSEC(100));      │     │
  │  │ k_msgq_get(&my_msgq, &item, K_FOREVER);        │     │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  k_mbox: Mailbox (synchronous exchange, variable-size)   │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ Sender and receiver rendezvous                  │     │
  │  │ Receiver can accept/reject based on message info│     │
  │  │ Supports zero-copy if receiver has buffer       │     │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  k_pipe: Byte-stream pipe (like Stream Buffer)           │
  │  k_fifo / k_lifo: Linked-list based (zero-copy)         │
  └──────────────────────────────────────────────────────────┘

  QNX Neutrino IPC
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Message Passing (synchronous, kernel-mediated):         │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ Client:  MsgSend(coid, smsg, slen, rmsg, rlen) │      │
  │  │          (blocks until server replies)           │      │
  │  │                                                  │      │
  │  │ Server:  rcvid = MsgReceive(chid, msg, len, info)│     │
  │  │          MsgReply(rcvid, status, reply, rlen)    │      │
  │  │                                                  │      │
  │  │ Pulse:   MsgSendPulse(coid, prio, code, value)  │      │
  │  │          (non-blocking, 8-byte message)          │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  QNX message passing is the foundation of its            │
  │  microkernel — ALL services (filesystem, network,        │
  │  drivers) communicate via MsgSend/MsgReceive.            │
  │  Provides location transparency (works across network).  │
  └──────────────────────────────────────────────────────────┘
```

---

## 7. Zero-Copy IPC

```c
/* Standard queue: copies data IN and OUT (2 memcpy per message) */
/*
 * Send: memcpy(queue_storage ← &sensorData, sizeof)
 * Recv: memcpy(&rxData ← queue_storage, sizeof)
 *
 * Fine for small messages (4-64 bytes)
 * Expensive for large messages (DMA buffers, images)
 */

/* Zero-copy pattern: pass POINTERS through queue */
/* Queue item size = sizeof(void *), not sizeof(data) */

QueueHandle_t xPtrQueue = xQueueCreate(10, sizeof(uint8_t *));

/* Producer: allocate buffer, fill it, send pointer */
void vProducerTask(void *pvParameters) {
    for (;;) {
        uint8_t *buf = pvPortMalloc(DMA_BUF_SIZE);
        if (buf != NULL) {
            dma_receive(buf, DMA_BUF_SIZE);
            /* Wait for DMA complete... */
            xQueueSend(xPtrQueue, &buf, portMAX_DELAY);
            /* Do NOT free buf — consumer owns it now */
        }
    }
}

/* Consumer: receive pointer, process data, free buffer */
void vConsumerTask(void *pvParameters) {
    uint8_t *rxBuf;
    for (;;) {
        if (xQueueReceive(xPtrQueue, &rxBuf, portMAX_DELAY) == pdPASS) {
            process_data(rxBuf, DMA_BUF_SIZE);
            vPortFree(rxBuf);  /* Consumer frees */
        }
    }
}

/*
 * Zero-copy rules:
 * 1. Clear ownership: sender gives up, receiver owns
 * 2. Only one task accesses buffer at a time
 * 3. Must use dynamic memory (or static pool)
 * 4. Careful: memory leaks if consumer fails to free
 * 5. Consider memory pool (fixed blocks) to avoid fragmentation
 */

/* Zephyr k_fifo: native zero-copy */
/* Items contain an embedded linked-list node */
struct sensor_data {
    void *fifo_reserved;   /* Used by k_fifo internally */
    uint32_t timestamp;
    int16_t values[3];
};

struct sensor_data *item = k_malloc(sizeof(*item));
item->timestamp = k_uptime_get_32();
k_fifo_put(&sensor_fifo, item);

/* Consumer */
struct sensor_data *rx = k_fifo_get(&sensor_fifo, K_FOREVER);
process(rx);
k_free(rx);
```

---

## Interview Questions

**Q1: Why does FreeRTOS copy data into queues instead of using pointers?**
**A:** Copy-by-value is safer and simpler: (1) The sender's local variable can go out of scope — the data lives safely in the queue's own storage. No dangling pointer risk. (2) No ownership tracking needed — once copied in, the queue owns it. (3) Thread-safe by design — each queue operation is atomic with respect to the data. (4) No dynamic memory needed — queue storage is pre-allocated. (5) Simpler reasoning — no aliasing problems. The cost is two memcpy operations per message. For small items (up to ~64 bytes), the copy overhead is negligible. For large data (DMA buffers), pass pointers through the queue (queue item size = `sizeof(void*)`), but then the programmer must manage ownership explicitly.

**Q2: When would you use a Stream Buffer vs a Message Buffer vs a Queue?**
**A:** Queue: fixed-size items, multiple readers/writers, FIFO order, most general. Use for sensor readings, commands, events. Stream Buffer: continuous byte stream with no message boundaries, like a FIFO pipe. Single reader + single writer only. Use for UART data, audio streams, any continuous data flow. Message Buffer: variable-length messages with preserved boundaries, also single reader + single writer. Use for protocol packets of varying size, log messages of different lengths. Built on top of stream buffer with a 4-byte length header per message. Key: stream/message buffers are lighter weight than queues but limited to one reader + one writer.

**Q3: Explain QNX message passing. How does it differ from FreeRTOS queues?**
**A:** QNX uses synchronous, kernel-mediated message passing as its fundamental IPC. `MsgSend()` blocks the client until the server calls `MsgReply()`. This creates a natural request-response pattern. The kernel copies data between address spaces (since QNX processes have separate MMU-protected spaces). Key differences from FreeRTOS queues: (1) Synchronous — sender blocks until receiver responds (not just until message is in queue). (2) Cross-process — works between separate address spaces with MMU protection. (3) Location transparent — can work across a network (Transparent Distributed Processing). (4) Bidirectional — reply goes back via same `rcvid`. (5) Variable-length — message size specified per call. QNX also has pulses for lightweight, non-blocking, ISR-safe 8-byte notifications.

**Q4: What is an event group and when is it preferred over multiple semaphores?**
**A:** An event group holds up to 24 event flags (bits) in a single object. A task can wait for ANY combination (OR) or ALL (AND) of those flags atomically. Without event groups, waiting on multiple conditions requires: (a) multiple semaphores with polling, or (b) a Queue Set (FreeRTOS), or (c) complex state machines. Event groups are preferred when: (1) a task needs to synchronize on multiple independent events ("wait until sensor ready AND config loaded AND CAN bus up"), (2) implementing a rendezvous/barrier where N tasks must all reach a point before proceeding (`xEventGroupSync`), (3) simple flag-based signaling where the "data" is just which events occurred. Limitation: `xEventGroupSetBitsFromISR()` defers to the timer daemon task (not as fast as direct `xSemaphoreGiveFromISR()`).

---

## Summary

- Queues: most versatile IPC — fixed-size items, copy by value, multiple readers/writers
- Stream buffers: byte-stream, single reader/writer, no message boundaries
- Message buffers: variable-length messages with boundaries, single reader/writer
- Event groups: up to 24 event flags, wait for AND/OR combinations atomically
- Task notifications: fastest (embedded in TCB), but single-sender limitation
- Zero-copy: pass pointers through queue for large data, manage ownership carefully
- QNX uses synchronous message passing as its fundamental IPC model
- Zephyr k_fifo/k_lifo: zero-copy by embedding list node in data structure

---

[Previous Chapter: Interrupt Handling ←](Chapter_09_Interrupt_Handling.md) | [Next Chapter: Synchronization Primitives →](Chapter_11_Synchronization.md)
