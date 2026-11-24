"""
# main.py - Smart Office NamePlate Display System for Wokwi Simulation
# Author: Gilbert Fevry
"""

import time
from machine import Pin, PWM, I2C
import neopixel

# ==============================================================================
# 1. LOGGING UTILITY
# ==============================================================================
class Log:
    @classmethod
    def i(cls, message): print(f"[INFO] {message}")
    @classmethod
    def d(cls, message): print(f"[DEBUG] {message}")
    @classmethod
    def w(cls, message): print(f"[WARN] {message}")
    @classmethod
    def e(cls, message): print(f"[ERROR] {message}")


# ==============================================================================
# 2. HD44780 I2C LCD DRIVER (PCF8574)
# ==============================================================================
class I2cLcd:
    MASK_RS = 0x01
    MASK_E  = 0x04
    SHIFT_DATA = 4
    LCD_FUNCTION_2LINES = 0x08
    LCD_FUNCTION = 0x20
    LCD_CLR = 0x01
    LCD_ENTRY_MODE = 0x04
    LCD_ENTRY_INC = 0x02
    LCD_ON_CTRL = 0x08
    LCD_ON_DISPLAY = 0x04

    def __init__(self, i2c, i2c_addr, num_lines=2, num_columns=16):
        self.i2c = i2c
        self.i2c_addr = i2c_addr
        self.num_lines = num_lines
        self.num_columns = num_columns
        time.sleep_ms(20)
        self.hal_write_init(0x30)
        time.sleep_ms(5)
        self.hal_write_init(0x30)
        time.sleep_ms(1)
        self.hal_write_init(0x30)
        time.sleep_ms(1)
        self.hal_write_init(0x20)
        time.sleep_ms(1)
        self.hal_write_cmd(self.LCD_FUNCTION | self.LCD_FUNCTION_2LINES)
        self.hal_write_cmd(self.LCD_ON_CTRL | self.LCD_ON_DISPLAY)
        self.hal_write_cmd(self.LCD_ENTRY_MODE | self.LCD_ENTRY_INC)
        self.clear()

    def hal_write_init(self, nibble):
        byte = ((nibble >> 4) & 0x0f) << self.SHIFT_DATA
        self.i2c.writeto(self.i2c_addr, bytes([byte | self.MASK_E | 0x08]))
        self.i2c.writeto(self.i2c_addr, bytes([byte | 0x08]))

    def hal_write_cmd(self, cmd):
        self._write_byte(cmd, 0)

    def hal_write_data(self, data):
        self._write_byte(data, self.MASK_RS)

    def _write_byte(self, val, mode):
        high = (val & 0xf0) | mode | 0x08
        low = ((val << 4) & 0xf0) | mode | 0x08
        self.i2c.writeto(self.i2c_addr, bytes([high | self.MASK_E, high, low | self.MASK_E, low]))

    def clear(self):
        self.hal_write_cmd(self.LCD_CLR)
        time.sleep_ms(2)

    def move_to(self, col, row):
        addr = col + (0x40 if row > 0 else 0x00)
        self.hal_write_cmd(0x80 | addr)

    def putstr(self, string):
        for char in string:
            self.hal_write_data(ord(char))


# ==============================================================================
# 3. HARDWARE HIERARCHIES (From UML Class Diagram)
# ==============================================================================
class Display:
    """ Base Display class """
    def clear(self): pass
    def showText(self, text, row=0, col=0): pass


class LCDDisplay(Display):
    """ LCD 1602 implementation inheriting from Display """
    def __init__(self, sda=0, scl=1):
        i2c = I2C(0, sda=Pin(sda), scl=Pin(scl), freq=400000)
        devices = i2c.scan()
        addr = devices[0] if devices else 0x27
        self._lcd = I2cLcd(i2c, addr, 2, 16)
        Log.i(f"LCDDisplay initialized at I2C address 0x{addr:02X}")

    def clear(self):
        self._lcd.clear()

    def showText(self, text, row=0, col=0):
        self._lcd.move_to(col, row)
        self._lcd.putstr(text[:16])


class Buzzer:
    """ Base Buzzer class """
    def play(self, tone): pass
    def stop(self): pass


class PassiveBuzzer(Buzzer):
    """ Passive PWM Buzzer inheriting from Buzzer """
    def __init__(self, pin):
        self._pwm = PWM(Pin(pin))
        self.stop()

    def play(self, tone):
        self._pwm.freq(tone)
        self._pwm.duty_u16(32767)

    def stop(self):
        self._pwm.duty_u16(0)


class Light:
    """ Base Light class """
    def on(self): pass
    def off(self): pass


class LightStrip(Light):
    """ NeoPixel WS2812B Light Strip inheriting from Light """
    def __init__(self, pin=2, numleds=8, brightness=0.5):
        self._numleds = numleds
        self._brightness = brightness
        self._np = neopixel.NeoPixel(Pin(pin), numleds)
        self.off()

    def setColor(self, color):
        scaled = tuple(int(c * self._brightness) for c in color)
        for i in range(self._numleds):
            self._np[i] = scaled
        self._np.write()

    def off(self):
        for i in range(self._numleds):
            self._np[i] = (0, 0, 0)
        self._np.write()


class Button:
    """ Debounced push button with interrupt callbacks """
    def __init__(self, pin, name, handler=None):
        self._pin = Pin(pin, Pin.IN, Pin.PULL_UP)
        self._name = name
        self._handler = handler
        self._last_time = 0
        self._pin.irq(trigger=Pin.IRQ_FALLING, handler=self._callback)

    def setHandler(self, handler):
        self._handler = handler

    def _callback(self, pin):
        now = time.ticks_ms()
        if time.ticks_diff(now, self._last_time) > 200:  # 200ms debounce
            self._last_time = now
            if self._handler:
                self._handler.buttonPressed(self._name)


# ==============================================================================
# 4. EMPLOYEE DOMAIN HIERARCHY (Inheritance & Polymorphism)
# ==============================================================================
class Employee:
    """ Abstract base employee class """
    AVAILABLE = "Available"
    AWAY = "Away"
    BUSY = "Busy"
    STATUS_CYCLE = [AVAILABLE, AWAY, BUSY]

    def __init__(self, name, phone, position, department, availability="Available"):
        self._name = name
        self._phoneNumber = phone
        self._position = position
        self._department = department
        self._availability = availability

    def getName(self): return self._name
    def getPosition(self): return self._position
    def getPhoneNumber(self): return self._phoneNumber
    def getAvailability(self): return self._availability

    def cycleAvailability(self):
        curr = self.getAvailability()
        nxt = (self.STATUS_CYCLE.index(curr) + 1) % len(self.STATUS_CYCLE)
        self._availability = self.STATUS_CYCLE[nxt]
        Log.i(f"{self._name} status cycled -> {self._availability}")

    def displayInfo(self, display):
        """ Polymorphic display contract """
        display.clear()
        display.showText(self._name[:16], row=0, col=0)
        display.showText(self._position[:16], row=1, col=0)


class Faculty(Employee):
    """ Faculty: Displays name and office hours """
    def __init__(self, name, phone, position, department, courses=None, officeHours="TR 2-4 PM"):
        super().__init__(name, phone, position, department)
        self._courses = courses or []
        self._officeHours = officeHours

    def displayInfo(self, display):
        display.clear()
        display.showText(self._name[:16], row=0, col=0)
        display.showText(f"OH: {self._officeHours[:12]}", row=1, col=0)


class Staff(Employee):
    """ Staff: Displays name and working hours """
    def __init__(self, name, phone, position, department, workHours="M-F 8am-4pm"):
        super().__init__(name, phone, position, department)
        self._workHours = workHours

    def displayInfo(self, display):
        display.clear()
        display.showText(self._name[:16], row=0, col=0)
        display.showText(f"Hrs: {self._workHours[:11]}", row=1, col=0)


class Administrator(Employee):
    """ Administrator: Automated calendar-driven availability """
    def __init__(self, name, phone, position, department):
        super().__init__(name, phone, position, department)
        self._startTime = time.time()

    def getAvailability(self):
        # Automated simulated calendar cycle: 0-25s Available, 25-45s Busy, 45-60s Away
        elapsed = int(time.time() - self._startTime) % 60
        if elapsed < 25:
            return self.AVAILABLE
        elif elapsed < 45:
            return self.BUSY
        else:
            return self.AWAY

    def cycleAvailability(self):
        Log.i(f"Admin {self._name}: Manual toggle ignored (availability tied to calendar).")

    def displayInfo(self, display):
        display.clear()
        display.showText(self._name[:16], row=0, col=0)
        display.showText(f"Cal: {self.getAvailability()[:11]}", row=1, col=0)


class StudentWorker(Employee):
    """ Student Worker: Displays name and shift schedule """
    def __init__(self, name, phone, position, department, supervisor="Staff", shiftHours="MWF 1-5 PM"):
        super().__init__(name, phone, position, department)
        self._supervisor = supervisor
        self._shiftHours = shiftHours

    def displayInfo(self, display):
        display.clear()
        display.showText(self._name[:16], row=0, col=0)
        display.showText(f"Shift:{self._shiftHours[:10]}", row=1, col=0)


# ==============================================================================
# 5. CONTROLLER (NamePlate)
# ==============================================================================
class NamePlate:
    """ Coordinates hardware and employee views """
    COLOR_AVAILABLE = (0, 255, 0)      # Green
    COLOR_AWAY = (255, 180, 0)         # Amber / Yellow
    COLOR_BUSY = (255, 0, 0)           # Red

    def __init__(self, display, buzzer, lightstrip, btnPrev, btnNext, btnAvail, employees):
        self._display = display
        self._buzzer = buzzer
        self._lightstrip = lightstrip
        self._employees = employees
        self._currentIndex = 0

        btnPrev.setHandler(self)
        btnNext.setHandler(self)
        btnAvail.setHandler(self)

    def getCurrentEmployee(self):
        return self._employees[self._currentIndex]

    def nextEmployee(self):
        self._currentIndex = (self._currentIndex + 1) % len(self._employees)
        self._chirp(880, 60)
        self.updateView()

    def previousEmployee(self):
        self._currentIndex = (self._currentIndex - 1) % len(self._employees)
        self._chirp(587, 60)
        self.updateView()

    def toggleAvailability(self):
        emp = self.getCurrentEmployee()
        emp.cycleAvailability()
        # Confirmation chime
        self._buzzer.play(700)
        time.sleep_ms(40)
        self._buzzer.stop()
        time.sleep_ms(30)
        self._buzzer.play(900)
        time.sleep_ms(50)
        self._buzzer.stop()
        self.updateView()

    def updateView(self):
        emp = self.getCurrentEmployee()
        # 1. Polymorphic display render
        emp.displayInfo(self._display)

        # 2. LightStrip status color update
        status = emp.getAvailability()
        color = self.COLOR_AVAILABLE
        if status == Employee.AWAY:
            color = self.COLOR_AWAY
        elif status == Employee.BUSY:
            color = self.COLOR_BUSY
        self._lightstrip.setColor(color)

    def buttonPressed(self, name):
        Log.i(f"Button Pressed: {name}")
        if "next" in name:
            self.nextEmployee()
        elif "prev" in name:
            self.previousEmployee()
        elif "avail" in name:
            self.toggleAvailability()

    def _chirp(self, tone, ms):
        self._buzzer.play(tone)
        time.sleep_ms(ms)
        self._buzzer.stop()


# ==============================================================================
# 6. APPLICATION ENTRY POINT
# ==============================================================================
def main():
    Log.i("Starting Wokwi Smart Office NamePlate System...")

    # Hardware instantiation
    lcd = LCDDisplay(sda=0, scl=1)
    buzzer = PassiveBuzzer(pin=15)
    strip = LightStrip(pin=2, numleds=8, brightness=0.5)

    btn_prev = Button(pin=10, name="btn_prev")
    btn_next = Button(pin=11, name="btn_next")
    btn_avail = Button(pin=12, name="btn_avail")

    # Seed Employees (Faculty, Staff, Administrator, Student Worker)
    employees = [
        Faculty("Dr. Alan Turing", "305-348-1001", "Professor", "SCIS", officeHours="TR 2-4 PM"),
        Staff("Grace Hopper", "305-348-1002", "IT Director", "CS", workHours="M-F 8am-4pm"),
        Administrator("Dean Lovelace", "305-348-1003", "Associate Dean", "Engineering"),
        StudentWorker("Linus Torvalds", "305-348-1004", "Lab Assistant", "SCIS", shiftHours="MWF 1-5 PM")
    ]

    plate = NamePlate(lcd, buzzer, strip, btn_prev, btn_next, btn_avail, employees)
    plate.updateView()

    Log.i("System active! Click the Blue (Prev), Green (Next), or Orange (Avail) buttons.")

    # Periodic background refresh for Administrator calendar updates
    while True:
        emp = plate.getCurrentEmployee()
        if isinstance(emp, Administrator):
            plate.updateView()
        time.sleep(1)


if __name__ == "__main__":
    main()