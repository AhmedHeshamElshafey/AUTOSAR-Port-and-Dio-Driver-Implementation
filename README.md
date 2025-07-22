# AUTOSAR Port and Dio Driver Implementation on Tiva C

This project demonstrates a **modular and AUTOSAR-compliant implementation** of the **Port** and **Dio drivers** on the **Tiva C (TM4C123GH6PM)** microcontroller, along with **Button** and **LED** modules and a simple demo application. To validate the functionality, a **cooperative time-triggered task scheduler** (mini OS simulation) is implemented to execute tasks at predefined intervals based on timer ticks.

---

## 📌 Overview

### ✅ What This Project Includes

- **AUTOSAR MCAL Layer**
  - `Port Driver`: Configures GPIO pins (direction, pull-up/down, mode,...etc).
  - `Dio Driver`: Provides digital input/output functions for channels and ports.
- **ECUAL Layer**
  - `Button Module`: Abstracts button handling using Dio.
  - `LED Module`: Controls LEDs using Dio.
- **Application Layer**
  - Demo app to toggle LED when button is pressed.
  - Uses simple scheduler (tick-based) to simulate real-time behavior.

---

## ⚙️ Hardware Requirements

- **Tiva C LaunchPad (TM4C123GH6PM)**
- 1 Push Button (digital input)
- 1 LED (digital output)
- USB Debugger (ICDI or other)
- IDE:  Code Composer Studio

---
## 📂 Project Structure
```
├── MCAL/
│ ├── Dio/
│ │ ├── Dio.c/.h
│ │ ├── Dio_Cfg.h
│ │ └── Dio_Regs.h
│ └── Port/
│ ├── Port.c/.h
│ ├── Port_Cfg.h
│ └── Port_Regs.h
│
├── ECUAL/
│ ├── Button/
│ │ ├── Button.c/.h
│ └── LED/
│ ├── LED.c/.h
│
├── APP/
│ └── main.c → Includes task scheduler logic
│
├── Config/
│ ├── Dio_Cfg.c/.h
│ └── Port_Cfg.c/.h
│
└── README.md
```
---
## 🔧 MCAL Layer:

### 1️⃣ Port Driver

**File**: `Port.c`, `Port.h`, `Port_Cfg.h`, `Port_Regs.h`

#### 🔹 Responsibilities:
- Initialize GPIO ports and pins.
- Set direction (input/output), pull-up/down, digital/analog mode.
- Follows AUTOSAR Port driver service APIs (e.g., `Port_Init()`).
- You define each pin’s config in Port_Cfg.c, making the driver reusable and decoupled.

#### 🧠 Code Behavior:

```c
void Port_Init(const Port_ConfigType* ConfigPtr)
{
    for (int i = 0; i < PORT_CONFIGURED_PINS; i++)
    {
        // 1. Enable clock to the corresponding port using RCGCGPIO
        // 2. Unlock pin if it's PC0-PC3 (special function unlock)
        // 3. Set direction via GPIODIR
        // 4. Configure pull-up or pull-down via GPIOPUR or GPIOPDR
        // 5. Enable digital function via GPIODEN
        // 6. Configure alternative function, if needed
    }
}
```
### 2️⃣ Dio Driver
**File**: `Dio.c`, `Dio.h`, `Dio_Cfg.h`, `Dio_Regs.h`

#### 🔹 Responsibilities:
- Abstract direct register access for reading/writing digital pins.
- Provides channel-level (pin), port-level, and group-level access.
- Used internally by Button and LED modules to interact with hardware.

#### 🧠 Code Behavior:
```c
Dio_LevelType Dio_ReadChannel(Dio_ChannelType ChannelId)
{
    // Get port base address and channel offset from ChannelId
    // Return HIGH or LOW based on the bit status
}

void Dio_WriteChannel(Dio_ChannelType ChannelId, Dio_LevelType Level)
{
    // Set or clear specific bit using GPIO DATA register
}
```
## 🧩 ECUAL Layer
### 3️⃣ LED Module
**File**: `LED.c`, `LED.h`

#### 🔹 Responsibilities:
- Toggle, set, or clear an LED.
- Interfaces with Dio_WriteChannel().

#### 🧠 Example:
```c
void LED_Toggle(void)
{
    Dio_LevelType current = Dio_ReadChannel(LED_CHANNEL);
    Dio_WriteChannel(LED_CHANNEL, !current);
}
```
### 4️⃣ Button Module
**File**: `Button.c`, `Button.h`

#### 🔹 Responsibilities:
- Read button press state.
_ Abstracts debouncing if extended.

#### 🧠 Example:
```c
ButtonState Button_GetState(void)
{
    return Dio_ReadChannel(BUTTON_CHANNEL);
}
```
## 🧠 Application Layer (main.c)
### ⏱ Mini Scheduler (Tick-Based OS Simulation)
- Implements a static schedule based on timer ticks. Each tick triggers certain tasks at defined intervals.

#### 🔹 Structure:
```c
if (g_New_Time_Tick_Flag == 1)
{
    switch (g_Time_Tick_Count)
    {
        case 20: case 100:
            Button_Task();
            break;

        case 40: case 80:
            Button_Task();
            Led_Task();
            break;

        case 60:
            Button_Task();
            App_Task();
            break;

        case 120:
            Button_Task();
            App_Task();
            Led_Task();
            g_Time_Tick_Count = 0;
            break;
    }
    g_New_Time_Tick_Flag = 0;
}
```
#### 🧠 High-Level Purpose:
- This code implements a static time-based scheduler, where:
    - A timer (e.g., SysTick) runs in the background and periodically sets a flag every 10 milliseconds.
    - This flag `(g_New_Time_Tick_Flag)` tells the main loop:**“Hey, a new time slice has arrived — check if any task should run.”**
    - A counter `(g_Time_Tick_Count)` keeps track of how many ticks have passed.
    - Based on this counter, specific tasks are executed at specific intervals.
#### 🔍 Code Breakdown (Line-by-Line):

**(1)** `if (g_New_Time_Tick_Flag == 1)` :  
 - Check if a new timer tick has occurred
 - This flag is likely set inside a Timer ISR (Interrupt Service Routine), like:
   ```c
          void SysTick_Handler(void)
          {
            g_New_Time_Tick_Flag = 1;
            g_Time_Tick_Count++;
          }
   ```

**(2)** `switch (g_Time_Tick_Count)`:
 - Check which time slot we're in
 - This counter increases with every timer tick.
 - You’re using a switch statement to map certain tick values to specific actions (task execution).
      For example, if your system tick is every 10ms, then:
      20 → 200ms ,
      40 → 400ms ,
      60 → 600ms ,
      ... ,
      120 → 1.2 seconds

**(2)** `Task Cases` : <br>
✅ Every 200ms and 1 second, only the button is checked.
- Button_Task() likely reads the button state and stores it (pressed/released).
- Useful for debouncing or periodic sampling.

✅ At 400ms and 800ms, the system checks:
- `Button_Task()`→ reads the input state.
- `Led_Task()` → maybe toggles the LED based on app state.
- You’re updating outputs based on past inputs here.

✅ At 600ms:
- Read the button again.
- Then run App_Task(), which is probably the main decision logic (e.g., "If button was pressed, toggle LED").
- This case links input reading with application decision-making.

✅ At 1.2 seconds:

- Run all 3 tasks: `Read input` , `Update app logic` , `Update LED`
- Then reset the counter, so the cycle repeats from zero.
- This case ensures that all tasks are re-run periodically to keep the system up to date and cyclic.
#### 🧩 Why Use This?
✅ No RTOS required <br>
✅ Deterministic behavior<br>
✅ CPU-friendly (no busy waiting)<br>
✅ Helps in AUTOSAR OS concepts like:<br>
    - Cyclic Tasks<br>
    - Schedule Tables<br>
    - Alarms<br>
✅ Easy to debug and extend<br>

## 🔁 Functional Flow Summary:<br>
**(1)** `Port_Init()` configures pins (LED output, Button input).<br>
**(2)** `Dio_Init()` maps logical channels to physical ports.<br>
**(3)** `Timer ISR` sets g_New_Time_Tick_Flag every X ms.<br>
**(4)** Application checks flag, increments g_Time_Tick_Count.<br>
**(5)** Based on tick count, Button_Task, Led_Task, and App_Task execute.<br>
**(6)** `Button_Task()` reads button, stores state.<br>
**(7)** `App_Task()` checks state and triggers logic.<br>
**(8)** `Led_Task()` updates LED.<br>

## 🔬 Configuration Files
### Port_Cfg.h / .c
- Define pins' initial settings

### Dio_Cfg.h / .c
- Maps channels to IDs used by Dio


