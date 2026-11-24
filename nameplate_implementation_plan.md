# Implementation Plan: Smart Office NamePlate Display System

This implementation plan outlines the development of the **NamePlate Display System** for Raspberry Pi Pico using MicroPython and the PicoLibrary project architecture.

\---

## 1\. Architecture \& Design Overview

The system follows the **Model-View-Controller (MVC)** architectural pattern:

```mermaid
flowchart TD
    subgraph View \\\["Hardware Views"]
        LCD\\\["LCDDisplay (16x2)"]
        LED\\\["LightStrip (NeoPixel Status)"]
        BUZ\\\["PassiveBuzzer (Audio Feedback)"]
    end

    subgraph Controller \\\["Coordination"]
        NP\\\["NamePlate Controller"]
        BTN\\\["3x Buttons (Prev, Next, Avail)"]
    end

    subgraph Model \\\["Domain Models (Employee Hierarchy)"]
        EMP\\\["Employee (Abstract Base)"]
        FAC\\\["Faculty"]
        STF\\\["Staff"]
        ADM\\\["Administrator"]
        STW\\\["StudentWorker"]
    end

    BTN -->|Button Events| NP
    NP -->|Controls / Invokes| LCD
    NP -->|Sets Color Status| LED
    NP -->|Plays Beep / Chime| BUZ
    NP -->|Manages Collection| EMP
    EMP <|-- FAC
    EMP <|-- STF
    EMP <|-- ADM
    EMP <|-- STW
    EMP -->|Polymorphic Render| LCD
```

\---

## 2\. Hardware Pin Specifications (Raspberry Pi Pico)

|Hardware Component|PicoLibrary Class|Pico Pin / Bus|Description|
|-|-|-|-|
|**LCD Display (1602)**|`LCDDisplay`|GPIO 0 (SDA), GPIO 1 (SCL)|I2C LCD Display (I2C 0)|
|**Passive Buzzer**|`PassiveBuzzer`|GPIO 15|PWM audio tone generator|
|**NeoPixel Strip**|`LightStrip`|GPIO 2|8-LED WS2812B addressable RGB strip|
|**Previous Button**|`Button`|GPIO 10|Pull-up input: Navigate to previous employee|
|**Next Button**|`Button`|GPIO 11|Pull-up input: Navigate to next employee|
|**Availability Button**|`Button`|GPIO 12|Pull-up input: Toggle status (Available/Away/Busy)|

\---

## 3\. Implementation Phases

### Phase 1: Domain Model Hierarchy (`Employee.py`)

Build the core inheritance hierarchy ensuring strict adherence to the project standards (docstrings, headers, logging):

1. **`Employee` (Abstract Base Class)**:

   * Attributes: `\\\_name: str`, `\\\_phoneNumber: str`, `\\\_position: str`, `\\\_department: str`, `\\\_availability: str`.
   * Methods:

     * `getName()`, `getPhoneNumber()`, `getPosition()`, `getDepartment()`, `getAvailability()`.
     * `cycleAvailability()`: Rotates through `("Available", "Away", "Busy")`.
     * `displayInfo(display: LCDDisplay)`: Abstract method to be overridden by subclasses.
2. **`Faculty(Employee)`**:

   * Specific attributes: `\\\_courses: list\\\[str]`, `\\\_officeHours: str`.
   * Overrides `displayInfo()`:

     * Line 1: `f"{self.\\\_name} ({self.\\\_position})"` (truncated/formatted to 16 chars).
     * Line 2: `f"Hrs:{self.\\\_officeHours}"` or active course.
3. **`Staff(Employee)`**:

   * Specific attributes: `\\\_workHours: str`, `\\\_duties: str`.
   * Overrides `displayInfo()`:

     * Line 1: `f"{self.\\\_name} - Staff"`
     * Line 2: `f"Hours: {self.\\\_workHours}"`
4. **`Administrator(Employee)`**:

   * Specific attributes: `\\\_calendarSchedule: dict`.
   * Overrides `getAvailability()`: Dynamically queries schedule/time to determine status.
   * Overrides `cycleAvailability()`: Disabled / no-op (logs note that calendar automates availability).
   * Overrides `displayInfo()`:

     * Line 1: `f"Admin: {self.\\\_name}"`
     * Line 2: `f"{self.\\\_position} | Cal"`
5. **`StudentWorker(Employee)`**:

   * Specific attributes: `\\\_supervisor: str`, `\\\_shiftHours: str`.
   * Overrides `displayInfo()`:

     * Line 1: `f"{self.\\\_name} (Student)"`
     * Line 2: `f"Shift: {self.\\\_shiftHours}"`
6. **Self-Contained Verification**:

   * Include `if \\\_\\\_name\\\_\\\_ == "\\\_\\\_main\\\_\\\_":` block to instantiate all employee types, cycle statuses, and test polymorphic display outputs.

\---

### Phase 2: Controller \& Coordinator (`NamePlate.py`)

Implement the coordinator that connects user inputs to hardware outputs:

1. **State \& Attributes**:

   * `\\\_employees: list\\\[Employee]`
   * `\\\_currentIndex: int`
   * Hardware handles: `\\\_display`, `\\\_buzzer`, `\\\_lightstrip`, `\\\_btn\\\_prev`, `\\\_btn\\\_next`, `\\\_btn\\\_avail`.
2. **Navigation \& Audio Feedback**:

   * `nextEmployee()`: Increments `\\\_currentIndex` (with wrap-around), plays a high-pitch navigation chirp (e.g., 880 Hz, 50 ms), and updates view.
   * `previousEmployee()`: Decrements `\\\_currentIndex` (with wrap-around), plays a lower-pitch navigation chirp (e.g., 587 Hz, 50 ms), and updates view.
3. **Availability Management \& LightStrip Feedback**:

   * `toggleAvailability()`: Calls `cycleAvailability()` on the current employee, plays a double confirmation beep, and updates the status light.
   * Color mapping on `LightStrip`:

     * **Available**: Green `(0, 255, 0)`
     * **Away**: Amber / Yellow `(255, 180, 0)`
     * **Busy**: Red `(255, 0, 0)`
4. **View Refresh (`updateView()`)**:

   * Clears display.
   * Invokes polymorphic `current\\\_employee.displayInfo(self.\\\_display)`.
   * Sets NeoPixel LEDs to the current employee's availability color.
5. **Button Callback Integration**:

   * Implements `buttonPressed(name)` and `buttonReleased(name)` to route button events based on button name.
6. **Self-Contained Verification**:

   * Include `if \\\_\\\_name\\\_\\\_ == "\\\_\\\_main\\\_\\\_":` block with mock hardware to verify navigation, cycling, and edge cases.

\---

### Phase 3: Application Integration (`main.py`)

Develop the main entry point to run on the Raspberry Pi Pico:

1. **Hardware / Desktop Environment Detection**:

   * Safely import `machine`; fallback to simulation mocks if tested outside Pico.
2. **Component Instantiation**:

   * Set up `LCDDisplay`, `PassiveBuzzer`, `LightStrip`, and 3 `Button`s.
3. **Seed Employee Records**:

   * Faculty: Dr. Alan Turing (CS Dept, TR 2-4pm, courses: "CS101, CS402").
   * Staff: Grace Hopper (Systems Specialist, M-F 8am-4pm).
   * Administrator: Dean Ada Lovelace (Assoc Dean, automated calendar schedule).
   * Student Worker: Linus Torvalds (Lab Assistant, MWF 1-5pm).
4. **Execution Loop**:

   * Initialize `NamePlate` with seed employees.
   * Initial call to `updateView()`.
   * Keep-alive loop (`time.sleep(1)`) processing background interrupts.

\---

### Phase 4: Testing \& Verification Matrix

|Test ID|Test Description|Expected Result|Pass Criteria|
|-|-|-|-|
|**T1: Polymorphic Display**|Iterate through all 4 employee types|LCD shows distinct formatting per employee type|Subclass `displayInfo()` called without type checks|
|**T2: Navigation \& Audio**|Press Next/Prev buttons|Index moves cyclically; buzzer emits distinct navigation tones|Display \& LEDs update instantly; audible tones|
|**T3: Manual Availability**|Press Avail button on Faculty/Staff/Student|Status cycles Available $\\rightarrow$ Away $\\rightarrow$ Busy; LEDs change Green $\\rightarrow$ Yellow $\\rightarrow$ Red|LED color matches status|
|**T4: Admin Availability**|Press Avail button on Administrator|Manual cycle ignored; availability stays synced to calendar|Status remains calendar-controlled|
|**T5: Wrap-Around Bounds**|Press Prev at index 0; press Next at last index|Cleanly wraps around without `IndexError`|Circular navigation holds|

\---

## 4\. Key Questions \& Design Choices for the User

1. **LCD Display Model**: Are you using an I2C backpack (2 wires: SDA/SCL) or a direct 6-wire parallel connection (RS, E, D4-D7) for your LCD 1602?
2. **Calendar Automation Source**: For the Administrator, should calendar availability cycle based on simulated time intervals (e.g., changes every 30 seconds), or would you prefer a predefined hourly schedule lookup table?
3. **LightStrip Behavior**: Would you prefer the entire 8-LED strip to glow in the status color (Green/Yellow/Red), or have a single indicator pixel illuminated?

