# 01 – FreeRTOS Blinky + UART 🟢

### *Two tasks. One microcontroller. Zero chaos (thanks to the scheduler).*

> 🔥 **The cool part?** The LED blinks at 500ms and UART prints at 1000ms — two different frequencies running *simultaneously*. Try doing that cleanly in bare-metal without turning your `main()` into spaghetti.

---

## 🧠 What's Going On Here & Why?

In bare-metal embedded, you have **one loop that does everything** — and if you want two things happening at different rates, you're juggling timers, flags, and counters manually. It gets messy fast.

This project is the **entry point into FreeRTOS thinking** — the moment you stop writing *"do this, then do that"* code and start writing *"here's Task A, here's Task B, scheduler — you figure it out."*

The LED doesn't know the UART exists. The UART doesn't know the LED exists. They just run. That's the magic.

---

This is the **"Hello World" of FreeRTOS** — but make it interesting.

Instead of one main loop doing everything (the old bare-metal way), we let FreeRTOS run **two independent tasks** that each mind their own business:

| Task | What it does | Priority |
|------|-------------|----------|
| `StartBlink01` | Blinks the onboard LED every 500ms | `osPriorityBelowNormal` |
| `StartUartTask` | Sends `"UART Task Running"` every 1000ms | `osPriorityNormal` |

---

## 🍔 The Burger Analogy

Think of your MCU as a **single chef** in a kitchen.

In bare-metal, the chef does everything in one loop — flip patty, check fries, wipe counter, repeat. If one step takes too long, everything else waits.

With FreeRTOS, the chef has a **manager (the scheduler)**. The manager taps the chef on the shoulder and says:
> *"Hey, stop flipping patties for a microsecond — go check on the fries."*

The chef switches instantly. Both jobs get done. No one waits forever. That's **preemptive multitasking**.

---

## ⚙️ Setup

| Item | Details |
|------|---------|
| Board | Nucleo-F446RE |
| IDE | STM32CubeIDE |
| Config Tool | STM32CubeMX |
| OS | Windows |
| RTOS | FreeRTOS (CMSIS-RTOS v2) |
| UART | USART2 (via USB-to-Serial onboard) |

---

## 🔧 How It's Configured in CubeMX

<img width="1032" height="752" alt="freeRTOS_blinky_tasks" src="https://github.com/user-attachments/assets/f49ce885-6530-4e3e-9617-48f27df356db" />


Two tasks are created using `osThreadNew()`:

```c
uartTaskHandle = osThreadNew(StartUartTask, NULL, &uartTask_attributes);
blink01Handle  = osThreadNew(StartBlink01,  NULL, &blink01_attributes);
```

`osThreadNew()` is the CMSIS-RTOS v2 wrapper around FreeRTOS's `xTaskCreate()`. CubeMX generates the attribute structs for you — priorities, stack size, task name — all in one clean place.

---

## 💡 Why Two Different Priorities?

Great question. Here's the thinking:

- `uartTask` is `osPriorityNormal` — it's the **talker**, responsible for communication output. We want it to run reliably.
- `blink01` is `osPriorityBelowNormal` — it's just a **visual heartbeat**. If the scheduler is busy, the LED can wait a moment. No one dies.

When both tasks are blocked on `osDelay()`, the scheduler chills. The moment a delay expires, the **higher priority task runs first**. So if both timers fire at the same tick, UART wins.

<img width="831" height="567" alt="freeRTOS_blinky_prio" src="https://github.com/user-attachments/assets/a39d3195-9fdc-4dac-a1f0-dab1d695d34f" />


---

## 🔁 Task Code Walkthrough

### UART Task — The Talker
```c
void StartUartTask(void *argument)
{
    char msg[] = "UART Task Running\r\n";
    for(;;)
    {
        HAL_UART_Transmit(&huart2, (uint8_t*)msg, strlen(msg), HAL_MAX_DELAY);
        osDelay(1000);
    }
    osThreadTerminate(NULL);
}
```

Every second, it shouts `"UART Task Running"` over USART2. `osDelay(1000)` is key — it **yields control back to the scheduler** for 1000ms instead of spinning and hogging the CPU. Think of it as the task **voluntarily napping**.

> ⚠️ `osThreadTerminate(NULL)` after the loop is technically unreachable here — but it's good practice to have it. If the loop ever breaks, the task exits cleanly instead of crashing into the void.

---

### Blink Task — The Heartbeat
```c
void StartBlink01(void *argument)
{
    for(;;)
    {
        HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);
        osDelay(500);
    }
}
```

Toggles **PA5** (the onboard green LED on Nucleo-F446RE) every 500ms. Dead simple. But notice — it's also using `osDelay()` not `HAL_Delay()`.

**Why?** `HAL_Delay()` is a **blocking busy-wait** — the task sits and spins, hogging the CPU. `osDelay()` tells the scheduler *"I'm done for now, wake me up later"* — freeing the CPU for other tasks. This is the FreeRTOS way. 🙏

---

## 📟 TeraTerm Output

<img width="817" height="416" alt="freeRTOS_blinky_uart_teraterm" src="https://github.com/user-attachments/assets/d347ba24-bc10-4f44-9316-60e449a3886a" />


**Settings:** 115200 baud, 8N1, no flow control.

---

## 🏃 How to Run This

1. Clone the repo
2. Open the `.ioc` file in STM32CubeIDE (or CubeMX)
3. Generate code if needed
4. Build and flash to your Nucleo-F446RE
5. Open TeraTerm at **115200 baud** on the COM port for your board
6. Watch the LED blink and the terminal print — simultaneously 🎉

---

## 🤔 Key Takeaways

- `osDelay()` ≠ `HAL_Delay()` — always use `osDelay()` inside FreeRTOS tasks
- Priority matters — higher priority tasks preempt lower ones when both are ready
- FreeRTOS tasks are **infinite loops** — they never return, they delay and yield
- CubeMX handles the boilerplate so you can focus on the task logic

---

## 🌍 More Projects in This Repo

This is **Project 01** of my ongoing STM32 + FreeRTOS learning series. Each project builds on the last.

👉 **[Check out the full repo →](https://github.com/viswa08/Nucleo-F446RE-FreeRTOS-Projects)**

---

*Found this useful? A ⭐ helps others find this repo.*
