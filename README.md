# Industrial Crane Load & Collision Safety Guard

## 💡 Overview

This project implements a comprehensive safety and monitoring system for industrial crane hoisting installations. Using a **PIC18F4580 microcontroller**, the system handles real-time load weight monitoring via ADC and emergency crash preemption using hardware interrupts.

The firmware prevents catastrophic damage from structural cable overloading or carriage rail overruns by continuously scanning sensor inputs, updating an LCD dashboard, driving multi-state LED warnings, and actively overriding motor controls during critical faults.

## 🛠️ Hardware Requirements

* **Microcontroller:** PIC18F4580
* **Display:** 16x2 Character LCD
* **Sensors:** * Potentiometer (simulating the load cell/weight sensor)
* Limit Switch / Push Button (simulating the carriage bumper)


* **Actuators:** DC Motor with L293D driver (or Relay) for hoist mechanism
* **Indicators:** Green, Yellow, and Red LEDs; Piezo Buzzer
* **Controls:** Dedicated Reset Push Button

## 📌 Pin Configuration

| Component | Functionality | PIC18F4580 Pin | Notes |
| --- | --- | --- | --- |
| **Load Potentiometer** | Analog Load Input | `RA0 (AN0)` | Scaled to 0-100% via ADC |
| **Bumper Limit Switch** | Collision Interrupt | `RB0 (INT0)` | Triggers External Interrupt 0 on falling edge |
| **Reset Push Button** | System Reset | `RB1` | Digital GPIO with external pull-up resistor |
| **Hoist Motor Enable** | Motor Control | `RC0` | Commands the L293D driver or relay |
| **Green LED** | Safe Indicator | `RC1` | Active High |
| **Yellow LED** | Warning Indicator | `RC2` | Active High |
| **Red LED** | Overload Indicator | `RC3` | Active High |
| **Buzzer** | Audio Alert | `RC4` | Piezo element control |
| **LCD Data (D0-D7)** | 8-bit Parallel Bus | `PORTD (RD0-RD7)` | LCD Data lines |
| **LCD RS & EN** | Control Lines | `RE0`, `RE1` | Register Select and Enable strobe |

## ⚙️ System Logic & Operational States

The system operates in a continuous polling loop for weight management, but relies on absolute hardware interrupts (`INT0`) for collision preemption.

| System State | Trigger Condition | Output Actions | LCD Dashboard |
| --- | --- | --- | --- |
| **Normal Load** | ADC Load: 0% to 70% | • Green LED is ON.<br>

<br>• Motor is ENABLED. | **R1:** `LOAD: xx%`<br>

<br>**R2:** `STATUS: SAFE` |
| **Heavy Load Warning** | ADC Load: 71% to 90% | • Yellow LED flashes.<br>

<br>• Buzzer outputs a slow chirping signal. | **R1:** `LOAD: xx%`<br>

<br>**R2:** `STATUS: WARNING` |
| **Structural Overload** | ADC Load: > 90% | • Red LED is ON.<br>

<br>• Buzzer sounds continuously.<br>

<br>• Motor is DISABLED (forced low). | **R1:** `LOAD: xx%`<br>

<br>**R2:** `OVERLOAD - STOP!` |
| **Collision Stop** | Bumper switch triggers `INT0` | • Motor DISABLED instantly via ISR.<br>

<br>• Red & Yellow LEDs flash an alternating strobe (0xCC/0x33).<br>

<br>• System halts. | **R1:** `CRITICAL COLLISION`<br>

<br>**R2:** `SYSTEM HALTED!` |

### 🔄 Collision Reset Mechanism

Once the crane triggers the **Collision Stop** state, the system locks up for safety. The software continuously polls the dedicated Reset input switch (`RB1`). The crane remains totally unresponsive to load changes until the operator manually presses the Reset switch, which clears the alert flags and reinitializes the main monitoring loop.

## 🧮 ADC Scaling Logic

The PIC18F4580's ADC is configured to read an 8-bit result (0 to 255). To map this raw data to a user-friendly percentage for the LCD, the firmware utilizes the following mathematical scaling:

```c
// Convert 8-bit ADC resolution to a 0-100% scale
Percentage = (adc_value * 100) / 255;

```

## 🧑‍💻 Firmware Architecture

The code was written from scratch in **MPLAB X IDE** using the **XC8 Compiler**. Key implementation details include:

* **Configuration Bits:** Watchdog Timer disabled (`WDT = OFF`), Low-Voltage Programming disabled (`LVP = OFF`), and High-Speed Oscillator configured.
* **ADC Module:** `ADCON1` configured to set `RA0` as analog while keeping all other required output pins digital. `ADCON0` manages the channel selection and sampling.
* **Interrupt Service Routine (ISR):** Global (`GIE`) and Peripheral (`PEIE`) interrupts are enabled. The `__interrupt()` routine strictly listens for the `INT0IF` flag. Upon triggering, it immediately cuts power to the motor pins before handling the visual UI updates.
* **LCD Drivers:** Custom functions for `lcd_cmd()`, `lcd_data()`, and `lcd_init()` to handle the 8-bit initialization and data writing sequences.

## ▶️ Usage & Simulation

1. Open the project in **MPLAB X IDE**.
2. Compile the code using the XC8 toolchain to generate the `.hex` file.
3. Load the `.hex` file into your Proteus simulation or flash it directly to your PIC18F4580 development board.
4. **Testing:**
* Adjust the potentiometer to observe the state transitions from *Safe* -> *Warning* -> *Overload*.
* Trigger the bumper switch at any time to verify the hardware interrupt instantly overrides the active motor state.
* Press the Reset button to recover from a halted collision state.

## Circuit Diagram
![image alt](https://github.com/Gokulxu/PIC_PROJECTS/blob/25432d0c4f82eebd7129ab3573b24f573ef2de04/Screenshot%202026-06-16%20195905.png)



## Code 
```c
// ==============================================================================
// Project: Industrial Crane Load & Collision Safety Guard
// Target: PIC18F4580
// Compiler: XC8
// ==============================================================================

// 1. Configuration Bits
#pragma config OSC = HS    // High-Speed Oscillator 
#pragma config WDT = OFF   // Disable Watchdog Timer
#pragma config LVP = OFF   // Disable Low-Voltage Programming

#include <xc.h>
#include <stdio.h>         // Required for sprintf()

#define _XTAL_FREQ 20000000 // 20 MHz Crystal Frequency

// ------------------------------------------------------------------------------
// Hardware Pin Definitions
// ------------------------------------------------------------------------------
// LCD Control & Data (8-bit mode)
#define RS LATAbits.LATA2
#define EN LATAbits.LATA3
#define LCD_PORT LATD

// Motor Controls (L293D)
#define MOTOR_EN  LATCbits.LATC0
#define MOTOR_IN1 LATCbits.LATC5
#define MOTOR_IN2 LATCbits.LATC6

// Warning Indicators
#define GREEN_LED LATCbits.LATC1
#define YELLOW_LED LATCbits.LATC2
#define RED_LED   LATCbits.LATC3
#define BUZZER    LATCbits.LATC4

// Inputs
// RA0 is ADC Load Potentiometer
// RB0 is INT0 Limit Switch
#define RESET_BTN PORTBbits.RB1

// ------------------------------------------------------------------------------
// Function Prototypes
// ------------------------------------------------------------------------------
void system_init(void);
void lcd_cmd(unsigned char cmd);
void lcd_char(unsigned char data);
void lcd_string(const char *str);
void lcd_init(void);
void lcd_set_cursor(unsigned char row, unsigned char col);
unsigned char read_adc(void);

// ------------------------------------------------------------------------------
// Interrupt Service Routine (Collision Preemption)
// ------------------------------------------------------------------------------
void __interrupt() ISR(void) {
    // Check if INT0 (Bumper Limit Switch) triggered the interrupt
    if (INTCONbits.INT0IF == 1) {
        
        // 1. IMMEDIATE PREEMPTION: Cut hoist motor power
        MOTOR_EN = 0; 
        
        // 2. Update LCD Alert
        lcd_cmd(0x01); // Clear display
        __delay_ms(2);
        lcd_set_cursor(1, 1);
        lcd_string("CRITICAL CRASH!!");
        lcd_set_cursor(2, 1);
        lcd_string("SYSTEM HALTED!  ");
        
        // 3. Lockout Loop: Trap system until Reset button is pressed
        // Assuming the reset button is wired with a pull-up resistor (Active LOW)
        while (RESET_BTN == 1) {
            // Alternating Strobe pattern for Red & Yellow LEDs
            RED_LED = 1; YELLOW_LED = 0; BUZZER = 1;
            __delay_ms(200);
            RED_LED = 0; YELLOW_LED = 1; BUZZER = 0;
            __delay_ms(200);
        }
        
        // 4. Operator has pressed Reset: Clear warnings and flag
        RED_LED = 0; 
        YELLOW_LED = 0; 
        BUZZER = 0;
        
        // Clear display before returning to main loop
        lcd_cmd(0x01);
        __delay_ms(2);
        
        // Clear the Interrupt Flag to prevent an infinite interrupt loop
        INTCONbits.INT0IF = 0; 
    }
}

// ------------------------------------------------------------------------------
// Main Program Loop
// ------------------------------------------------------------------------------
void main(void) {
    system_init();
    lcd_init();
    
    char lcd_buffer[16];
    unsigned char adc_raw = 0;
    unsigned char load_percentage = 0;

    while (1) {
        // Scan Analog Load Input
        adc_raw = read_adc();
        
        // Scale 8-bit result (0-255) to a Percentage (0-100%)
        load_percentage = (adc_raw * 100) / 255;
        
        // Update top row of LCD
        lcd_set_cursor(1, 1);
        sprintf(lcd_buffer, "LOAD: %u%%    ", load_percentage);
        lcd_string(lcd_buffer);
        
        // Evaluate Safety Thresholds
        lcd_set_cursor(2, 1);
        
        // STATE 1: Normal Load (0% to 70%)
        if (load_percentage <= 70) {
            GREEN_LED = 1; 
            YELLOW_LED = 0; 
            RED_LED = 0; 
            BUZZER = 0;
            
            // Ensure motor is active and hoisting
            MOTOR_IN1 = 1; MOTOR_IN2 = 0; MOTOR_EN = 1; 
            
            lcd_string("STATUS: SAFE    ");
        }
        // STATE 2: Heavy Load Warning (71% to 90%)
        else if (load_percentage <= 90) {
            GREEN_LED = 0; 
            RED_LED = 0; 
            
            // Motor remains active, but warnings begin
            MOTOR_IN1 = 1; MOTOR_IN2 = 0; MOTOR_EN = 1; 
            lcd_string("STATUS: WARNING ");
            
            // Flashing Yellow LED & Chirping Buzzer using software delay
            YELLOW_LED = 1; BUZZER = 1;
            __delay_ms(250);
            YELLOW_LED = 0; BUZZER = 0;
            __delay_ms(250);
        }
        // STATE 3: Structural Overload (>90%)
        else {
            GREEN_LED = 0; 
            YELLOW_LED = 0; 
            RED_LED = 1;       // Solid Red
            BUZZER = 1;        // Continuous Buzzer
            
            MOTOR_EN = 0;      // Immediately Disable Motor
            
            lcd_string("OVERLOAD - STOP!");
            __delay_ms(100);
        }
    }
}

// ------------------------------------------------------------------------------
// Hardware Initialization
// ------------------------------------------------------------------------------
void system_init(void) {
    // 1. ADC Configuration
    ADCON1 = 0x0E;      // Set RA0/AN0 as Analog, all other ANx pins Digital
    ADCON2 = 0x0A;      // Left Justified output (we only need the top 8 bits), Fosc/32
    ADCON0bits.ADON = 1;// Turn on A/D module
    
    // 2. Port Directions (1 = Input, 0 = Output)
    TRISA = 0x01;       // RA0 = Input (Pot), RA2, RA3 = Output (LCD)
    TRISB = 0x03;       // RB0 = Input (INT0 Bumper), RB1 = Input (Reset Btn)
    TRISC = 0x80;       // RC0-RC6 = Outputs (Motor, LEDs, Buzzer)
    TRISD = 0x00;       // RD0-RD7 = Outputs (LCD Data)
    
    // Set initial states to OFF
    LATC = 0x00;        
    
    // 3. Interrupt Configuration
    INTCON2bits.INTEDG0 = 0; // Set INT0 to trigger on falling edge (button press)
    INTCONbits.INT0IF = 0;   // Clear the flag initially
    INTCONbits.INT0IE = 1;   // Enable INT0 external interrupt
    INTCONbits.PEIE = 1;     // Enable Peripheral Interrupts
    INTCONbits.GIE = 1;      // Enable Global Interrupts
}

// ------------------------------------------------------------------------------
// ADC Driver
// ------------------------------------------------------------------------------
unsigned char read_adc(void) {
    ADCON0bits.GO = 1;       // Start A/D Conversion
    while(ADCON0bits.GO);    // Wait for conversion to finish
    return ADRESH;           // Return only the top 8-bits (Left Justified)
}

// ------------------------------------------------------------------------------
// LCD Drivers
// ------------------------------------------------------------------------------
void lcd_cmd(unsigned char cmd) {
    LCD_PORT = cmd;
    RS = 0;
    EN = 1;
    __delay_ms(2);
    EN = 0;
}

void lcd_char(unsigned char data) {
    LCD_PORT = data;
    RS = 1;
    EN = 1;
    __delay_ms(2);
    EN = 0;
}

void lcd_string(const char *str) {
    while (*str != '\0') {
        lcd_char(*str++);
    }
}

void lcd_set_cursor(unsigned char row, unsigned char col) {
    unsigned char pos;
    if (row == 1) {
        pos = 0x80 + (col - 1);
    } else {
        pos = 0xC0 + (col - 1);
    }
    lcd_cmd(pos);
}

void lcd_init(void) {
    __delay_ms(20);         // Wait for LCD power up
    lcd_cmd(0x38);          // 8-bit mode, 2 lines, 5x7 dots
    lcd_cmd(0x0C);          // Display ON, Cursor OFF
    lcd_cmd(0x06);          // Entry mode: auto-increment
    lcd_cmd(0x01);          // Clear display
    __delay_ms(5);
}
