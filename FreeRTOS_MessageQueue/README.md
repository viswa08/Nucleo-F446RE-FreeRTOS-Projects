# 02 – FreeRTOS Message Queue 📬

### *Stop shouting across the room. Pass a note instead.*

> 🔥 **The cool part?** A button press and a periodic timer are both trying to talk to UART — at the same time, at different rates. Without a queue, they'd trip over each other. With a queue? Everyone waits their turn, no data lost, no chaos.

---

## 🧠 What's Going On Here & Why?

In the last project, tasks were lone wolves — they did their thing independently. But real embedded systems need tasks to **talk to each other**.

The problem: two tasks want to send data to one UART task. If they both just call `printf()` whenever they feel like it, you get **garbage output** — corrupted messages, race conditions, sadness.

The solution: a **Message Queue** — a proper mailbox where senders drop their messages and the receiver picks them up one by one, in order, at its own pace.

Think of it like a **Zomato order queue** 🍕 — multiple customers (tasks) place orders, the kitchen (UART task) processes them one at a time. No one jumps the counter. No order gets lost.

---

## 🗂️ The Three Tasks

| Task | Role | Priority |
|------|------|----------|
| `btnSenderTask` | Sends a message when the blue button is pressed | `osPriorityNormal` |
| `prdSenderTask` | Sends a message every 1000ms automatically | `osPriorityNormal` |
| `uartTask` | Reads from queue, prints via UART | `osPriorityNormal1` |

> `osPriorityNormal1` is one notch above Normal — UART task gets priority to drain the queue promptly.

---

## ⚙️ Setup

| Item | Details |
|------|---------|
| Board | Nucleo-F446RE |
| IDE | STM32CubeIDE |
| Config Tool | STM32CubeMX |
| OS | Windows |
| RTOS | FreeRTOS (CMSIS-RTOS v2) |
| Queue Size | 10 items |
| Item Type | `uint32_t` (via `messageQueue_t` struct) |

---

## ⏱️ Important: Why We Changed the Timer Source to TIM6

<img width="1752" height="805" alt="freeRtos_sys_mx" src="https://github.com/user-attachments/assets/aeaa3b8c-5c06-4383-93f5-3b2ce80038c0" />


By default, STM32 HAL uses **SysTick** as its timebase for `HAL_Delay()` and `HAL_GetTick()`.

FreeRTOS **also** wants SysTick — for its own scheduler tick.

Two drivers. One SysTick. That's a fight nobody wins.

The fix is simple: **move HAL's timebase to TIM6** (a basic timer, perfect for this job) and let FreeRTOS own SysTick entirely. CubeMX will even warn you to do this the moment you enable FreeRTOS. Always listen to that warning.

---

## ⏰ How FreeRTOS Timing Works (In 30 Seconds)

FreeRTOS runs on a **tick** — a periodic interrupt fired by SysTick, typically every 1ms.

Every tick, the scheduler wakes up and checks: *"Is anyone's delay expired? Any higher priority task ready?"*

`osDelay(1000)` doesn't spin. It tells the scheduler *"park me for 1000 ticks, wake me up later"* and hands the CPU to someone else. This is why `osDelay()` is a superpower compared to `HAL_Delay()`.

---

## 📦 The Message Queue

<img width="1867" height="850" alt="freertos_queue_mx" src="https://github.com/user-attachments/assets/5cac5690-52ed-4bc4-b246-b3ecfbcae04a" />


The queue is created in CubeMX with:
- **Queue depth:** 10 items — up to 10 messages can sit waiting before anyone panics
- **Item size:** `sizeof(messageQueue_t)` — a small struct carrying an event ID and a timestamp

```c
typedef struct {
    uint32_t event_id;
    uint32_t timestamp;
} messageQueue_t;
```

Two fields. That's it. Clean, simple, and tells the receiver *what happened* and *when*.

---

## 🔁 Task Code Walkthrough

### Button Sender — The Human Input
```c
if(HAL_GPIO_ReadPin(B1_GPIO_Port, B1_Pin) == GPIO_PIN_RESET)
{
    msg.event_id = 0x01;
    msg.timestamp = HAL_GetTick();
    osMessageQueuePut(messageQueueHandle, &msg, 0, 0);
    osDelay(200); // debounce
}
osDelay(20); // polling rate
```

Polls the blue button every 20ms. On press (active LOW — `GPIO_PIN_RESET`), it stamps the message with `event_id = 0x01` and the current tick count, then **drops it in the queue**. The `osDelay(200)` after a press is your debounce — stops one finger tap from becoming 10 messages.

---

### Periodic Sender — The Machine Input
```c
msg.event_id = 0x02;
msg.timestamp = HAL_GetTick();
osMessageQueuePut(messageQueueHandle, &msg, 0, 0);
osDelay(1000);
```

No conditions. Every second, it stamps `event_id = 0x02` and puts it in the queue. Clockwork. This simulates a sensor or a heartbeat signal — something that fires regardless of what the user is doing.

---

### UART Task — The Receiver
```c
if (osMessageQueueGet(messageQueueHandle, &msg, 0, osWaitForever) == osOK)
{
    printf("Event ID: %d, Timestamp: %lu\r\n", msg.event_id, msg.timestamp);
}
```

`osWaitForever` means this task **blocks indefinitely** until there's something in the queue — it doesn't spin, it sleeps. The moment a message lands, the scheduler wakes it up and it prints. Clean, efficient, no polling.

> ⚠️ For `printf()` to work over UART, `syscalls.c` needs to redirect `_write()` to `HAL_UART_Transmit()`. CubeMX doesn't do this automatically — it's a manual step.

<img width="815" height="417" alt="freertos_queue_teraterm" src="https://github.com/user-attachments/assets/a25418c8-6b14-447d-8f1e-58a8351dc6ae" />


---

## 🤔 Key Takeaways

- A **Message Queue** decouples producers from consumers — senders don't care when the receiver reads, receivers don't care when senders wrote
- `osMessageQueuePut()` with timeout `0` means *"drop it in or give up instantly"* — non-blocking send
- `osMessageQueueGet()` with `osWaitForever` means *"I'll wait as long as it takes"* — efficient blocking receive
- Always move HAL timebase off SysTick when using FreeRTOS — TIM6 is your friend
- Queue depth of 10 is your safety buffer — if UART falls behind, up to 10 messages queue up gracefully

---

## 🌍 More Projects in This Repo

This is **Project 02** of my STM32 + FreeRTOS series. Each project introduces one new RTOS concept.

👉 **[Check out the full repo →](https://github.com/viswa08/Nucleo-F446RE-FreeRTOS-Projects)**

---

Found this useful? A ⭐ helps others find this repo.
