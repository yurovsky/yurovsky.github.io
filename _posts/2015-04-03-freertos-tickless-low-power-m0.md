---
layout: post
title: Low power FreeRTOS tickless mode on Cortex M0 and M0+
---

FreeRTOS provides optional tick suppression (tickless idle), a common and very
effective low power technique.  The low power examples that ship with FreeRTOS
focus on power-optimized Cortex M3 and M4 MCUs (such as the STM32L1 and SAM4L)
and there are (at this time) no in-tree examples for M0 or M0+ parts.  The
approach, however, is the same but just a little bit more work is required
because the `ARM_CM0` port in FreeRTOS does not export the "timer setup"
symbol needed as part of tickless idle initialization.

Intead, it implements a private method `prvSetupTimerInterrupt` which only
deals with SysTick.  The fix is really simple, go ahead and make that symbol
public (and rename it to `vPortSetupTimerInterrupt` per FreeRTOS standard) and
mark the internal (default) implementation as being a weak define so that we
can replace it with an MCU-specific one.

Keep in mind that while FreeRTOS for ARM Cortex-M is very generic (relying on
standard things like SysTick and the NVIC), the tick suppression is one of those
areas where you must do MCU-specific things since you're effectively setting an
alarm (via a timer or similar hardware block), sleeping until woken by an
interrupt, and then fixing up the RTOS tick accounting on wakeup.  Another area
that is MCU-specific is your NVIC configuration (number of bits for priority
and groups) but that's covered in FreeRTOSConfig.h

## FreeRTOS patch

    --- a/FreeRTOS/Source/portable/GCC/ARM_CM0/port.c
    +++ b/FreeRTOS/Source/portable/GCC/ARM_CM0/port.c
    @@ -107,7 +107,7 @@
     /*
      * Setup the timer to generate the tick interrupts.
      */
    -static void prvSetupTimerInterrupt( void );
    +void vPortSetupTimerInterrupt( void );
     
     /*
      * Exception handlers.
    @@ -205,7 +205,7 @@
     
        /* Start the timer that generates the tick ISR.  Interrupts are disabled
        here already. */
    -	prvSetupTimerInterrupt();
    +	vPortSetupTimerInterrupt();
     
        /* Initialise the critical nesting count ready for the first task. */
        uxCriticalNesting = 0;
    @@ -358,7 +358,7 @@
      * Setup the systick timer to generate the tick interrupts at the required
      * frequency.
      */
    -void prvSetupTimerInterrupt( void )
    +__attribute__(( weak )) void vPortSetupTimerInterrupt( void )
     {
        /* Configure SysTick to interrupt at the requested rate. */
        *(portNVIC_SYSTICK_LOAD) = ( configCPU_CLOCK_HZ / configTICK_RATE_HZ ) - 1UL;

## Implementation

Per the FreeRTOS manual, you'll define `configUSE_TICKLESS_IDLE` as `2` in your
FreeRTOSConfig.h file in order to enable tick supression.  You now need to
implement the following somewhere in your code:

- `vPortSetupTimerInterrupt` should set up a timer of your choosing to act as the RTOS tick.  This takes the place of the standard SysTick in the CPU core.
- `vPortSuppressTicksAndSleep` will be called by FreeRTOS to go ahead and activate tick suppression.  This needs to set up the wakeup alarm timer, decide what low power mode (if any) to activate, activate that mode and, when execution resumes, fix up the RTOS tick accounting by calling `vTaskStepTick`.  See the [documentation](http://www.freertos.org/low-power-tickless-rtos.html) for more details and an example implementation.

The implementation will need the `vPortSuppressTicksAndSleep` which is actually inserted with the macro `portSUPPRESS_TICKS_AND_SLEEP` so you will need something like:

    #include <portmacro.h>
    
    void vPortSuppressTicksAndSleep(TickType_t xExpectedIdleTime);
    #define portSUPPRESS_TICKS_AND_SLEEP    vPortSuppressTicksAndSleep

and this can be placed right in to FreeRTOSConfig.h so that `vPortSuppressTicksAndSleep` gets called.
