/**
 * File:   01_led_blink.c
 * Author: ANANDHU KR (Mentor, SinroRobotics)
 * Description: Basic LED Blinking tutorial code for PIC18F4580.
 *              Toggles RD0 pin to blink a connected LED at regular intervals.
 */

// ==========================================
// 1. PIC18F4580 CONFIGURATION BITS
// ==========================================
// These configuration bits are essential settings written to the device Flash.
// They dictate fundamental hardware configurations before runtime execution begins.

// CONFIG1H: Configuration Register 1 High
#pragma config OSC = HS         // Oscillator Selection bits: HS (High-Speed Crystal) oscillator
#pragma config FCMEN = OFF      // Fail-Safe Clock Monitor: Disabled
#pragma config IESO = OFF       // Internal/External Oscillator Switchover: Disabled

// CONFIG2L: Configuration Register 2 Low
#pragma config PWRT = ON        // Power-up Timer: Enabled (prevents start-up voltage spikes from glitcing CPU)
#pragma config BOREN = OFF      // Brown-out Reset: Disabled

// CONFIG2H: Configuration Register 2 High
#pragma config WDT = OFF        // Watchdog Timer: Disabled (prevents automatic MCU resets in simple code)

// CONFIG3H: Configuration Register 3 High
#pragma config PBADEN = OFF     // PORTB A/D Enable: Set pins to digital I/O on reset
#pragma config LPT1OSC = OFF    // Low-Power Timer1 Oscillator: Configured for normal higher power operation
#pragma config MCLRE = ON       // MCLR Pin Enable: MCLR pin enabled (allows external reset)

// CONFIG4L: Configuration Register 4 Low
#pragma config LVP = OFF        // Single-Supply ICSP: Disabled (prevents high-voltage programmer pins conflicts)
#pragma config XINST = OFF      // Extended Instruction Set: Disabled

// Include file containing microchip peripheral register mappings
#include <xc.h>

// Define oscillator frequency (8MHz crystal) for XC8 delay routines if needed
#define _XTAL_FREQ 8000000 

/**
 * Basic software delay function
 * Creates a delay by making the microcontroller execute nested loops.
 * Note: Timing varies depending on the compiler optimization level.
 */
void delay() {
    int i, j;
    for (i = 0; i < 500; i++) {
        for (j = 0; j < 500; j++) {
            // Do nothing, just consume CPU cycles
        }
    }
}

/**
 * Main application entry point
 */
void main(void) {
    // ----------------------------------------------------
    // REGISTER CONFIGURATION
    // ----------------------------------------------------
    
    // 1. Configure all pins as Digital I/O
    // By default, many microcontroller pins are shared with Analog-to-Digital Converter (ADC) inputs.
    // Setting ADCON1 to 0x0F turns off analog capabilities, setting all pins to Digital.
    ADCON1 = 0x0F; 
    
    // 2. Set RD0 as Output
    // TRIS registers control direction: 0 = Output, 1 = Input
    // Setting TRISD0 (Bit 0 of TRISD) to 0 makes RD0 push/pull current.
    TRISDbits.TRISD0 = 0; 
    
    // 3. Clear the Output Latch for RD0 (Turn off LED initially)
    // LAT (Latch) registers write values to the output pin.
    LATDbits.LATD0 = 0;
    
    // ----------------------------------------------------
    // INFINITE RUNTIME LOOP
    // ----------------------------------------------------
    while (1) {
        LATDbits.LATD0 = 1; // Set RD0 to HIGH (5V) -> Turn LED On
        delay();
        
        LATDbits.LATD0 = 0; // Set RD0 to LOW (0V) -> Turn LED Off
        delay();
    }
}
