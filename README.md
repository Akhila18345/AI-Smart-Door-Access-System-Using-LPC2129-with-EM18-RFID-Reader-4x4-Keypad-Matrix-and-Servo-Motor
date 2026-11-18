
[README.md](https://github.com/user-attachments/files/35010395/README.md)
# Smart Door Lock System (LPC2129)

An RFID + keypad door lock built on the NXP **LPC2129** (ARM7TDMI) and
developed with **Keil uVision 5**.

The user scans an RFID card, the controller checks its ID, then asks for a
4-digit password on a 4x4 keypad. Only when both match does a servo motor
unlock the door for 30 seconds. A 16x2 LCD shows every step and a tri-colour
LED shows the result.

1. The LCD shows "scan your card".
2. The RFID reader sends 12 ASCII characters over UART0.
3. The ID is compared with the stored card number.
   - Wrong card: red LED, "invalid card", 20-second pause, back to step 1.
   - Correct card: green LED, "card accepted", then "enter password".
4. Four key presses are collected from the keypad.
   - Wrong password: red LED, "wrong password", 20-second pause, back to step 1.
   - Correct password: blue LED, "access granted", the servo opens the door,
     waits 30 seconds, then closes it and the system starts again.

## Hardware connections

| LPC2129 pin | Connected to | Notes |
|-------------|--------------|-------|
| P0.0 / P0.1 | UART0 TXD0 / RXD0 | RFID reader serial output |
| P0.8 - P0.11 | Keypad rows | Configured as outputs |
| P0.12 - P0.15 | Keypad columns | Configured as inputs |
| P0.17 | Green LED | Active low |
| P0.18 | Red LED | Active low |
| P0.19 | Blue LED | Active low |
| P0.7 | Servo signal | PWM channel 2 |
| P1.17 | LCD RS | 4-bit mode |
| P1.18 | LCD RW | 4-bit mode |
| P1.19 | LCD EN | 4-bit mode |
| P1.20 - P1.23 | LCD D4 - D7 | 4-bit mode |

Clock: 12 MHz crystal, PLL to 60 MHz CPU clock, 15 MHz peripheral clock
(VPBDIV = 0). The timer-based delays assume this 15 MHz PCLK.

Servo: 20 ms PWM period; 1.0 ms pulse opens the door, 2.0 ms closes it.

Keypad layout used by the scanner:

```text
1 2 3 A
4 5 6 B
7 8 9 C
* 0 # D
```

## Project files

| File | Purpose |
|------|---------|
| `armpro.uvproj` | Keil uVision project (target LPC2129) |
| `Startup.s` | Startup code, stack, PLL and VPBDIV configuration |
| `smartdoor_main.c` | Application flow: card check, password check, door control |
| `gpio.c` / `gpio.h` | Pin directions, active-low LED setup, pin number macros |
| `keypad.c` / `keypad.h` | 4x4 matrix keypad scan and de-bounce wait |
| `rfid.c` / `rfid.h` | Reads a 12-character card ID from UART0 |
| `servo.c` / `servo.h` | PWM setup, `servo_open()` and `servo_close()` |
| `4bit lcd_driver.c` | HD44780-compatible LCD in 4-bit mode |
| `uart0_driver.c` | Polled UART0 init, transmit and receive |
| `delay.c` | Timer0 based `delay_ms()` and `delay_sec()` |
| `header.h` | Prototypes for the LCD, UART and delay drivers |

## Build and flash

1. Open `armpro.uvproj` in Keil uVision 5.
2. Confirm the device is **NXP LPC2129** and the target options are unchanged.
3. Press **F7** (Project > Build Target). The output is `Objects\armpro.hex`
   (hex generation is enabled in the project).
4. Program the board with Flash Magic (ISP over UART) or a JTAG adapter.

There are no external libraries or include paths; every source file sits in
this folder.

## Configuration

All the values you are likely to change are in `smartdoor_main.c`:

| What | Where |
|------|-------|
| Valid card ID | `strcmp(card, "123456789012")` (line 25) |
| Valid password | `strcmp(password, "1234")` (line 40) |
| Door open time | `delay_ms(30000)` = 30 s (line 48) |
| Error message time | `delay_ms(20000)` = 20 s (lines 57 and 66) |

UART parameters are set in `uart0_driver.c`; the baud rate must match the RFID
reader.

## Testing

1. Power the board. The LCD asks for a card.
2. Scan the card programmed as `123456789012`. The LCD shows
   "card accepted" and the green LED turns on.
3. Type `1234` on the keypad. The LCD shows "access granted", the blue LED
   turns on, and the servo opens for 30 seconds.
4. Repeat with a different card, or a wrong password, and confirm the red LED
   and error message appear.

## Known limitations

- The card ID and password are compiled into the program; there is no way to
  add or change users at runtime.
- `main()` never calls `lcd_init()` or `uart0_init()`. If the LCD stays blank
  or the reader is not detected, call them once at startup, after
  `gpio_init()`, with the reader's baud rate, for example:
  `lcd_init(); uart0_init(9600);`
- The delays are busy-wait loops, so the controller does nothing else while
  the door is open or an error is displayed.
- Only the first 12 characters of the card data are read, and keypad input is
  not echoed on the LCD, so the user cannot see how many digits were entered.

## Author

Akhila Ganesh - Smart Door Lock project, LPC2129 / Keil uVision.
