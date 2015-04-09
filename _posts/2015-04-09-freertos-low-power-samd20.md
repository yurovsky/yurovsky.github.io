---
layout: post
title: Low power FreeRTOS tickless mode with Atmel SAMD20 family
---

This post discusses a proposed tickless idle implementation for the Atmel
SAMD20 Cortex M0+ MCU family.  This same code should work on the SAMD21 and
other related SAMD parts and it should also apply to the new power-optimized
SAML21 parts.  Please read [this post about FreeRTOS tick suppression and Cortex M0 first](http://yurovsky.github.io/2015/04/03/freertos-tickless-low-power-m0/) -- I describe a minor patch that you will need to apply there.

This is by no means the only way to implement tickless idle on these MCUs and
your application may require slightly different implementation choices.

## Timer setup

In this example we will use Channel 0 on the timer-counter 4 (TC4) block of the
MCU.  I believe that all SAMD2x and related parts have this piece of hardware
and our example code will therefore be generic enough.  Adjust the TC block and
channel as needed (for instance you may have routed some of the counters to
actual IO pins).

We must implement the FreeRTOS port method `vPortSetupTimerInterrupt`, this
needs to set up our timer for use as a reasonably slow OS tick (replacing the
standard SysTick that FreeRTOS would use otherwise). To keep things simple in
this example we will use the internal ultra low power 32kHz oscillator as our
clock source.  We then call the ASF timer-counter API to set things up:

    static struct tc_module tc;

    void vPortSetupTimerInterrupt(void)
    {
        struct tc_config config;

        tc_get_config_defaults(&config);
        config.clock_source     = GCLK_GENERATOR_5;
        config.counter_size     = TC_COUNTER_SIZE_32BIT;
        config.run_in_standby   = true;
        config.clock_prescaler  = TC_CLOCK_PRESCALER_DIV1;
        config.wave_generation  = TC_WAVE_GENERATION_MATCH_FREQ;

        configASSERT(tc_init(&tc, TC4, &config) == STATUS_OK);

        /* Connect to FreeRTOS tick handler */
        tc_register_callback(&tc, (tc_callback_t)xPortSysTickHandler,
            TC_CALLBACK_CC_CHANNEL0);
        tc_enable_callback(&tc, TC_CALLBACK_CC_CHANNEL0);

        /* Set up the counter */
        tc_set_count_value(&tc, 0);
        tc_set_top_value(&tc, TIMER_RELOAD_VALUE_ONE_TICK);

        /* Start */
        tc_enable(&tc);
    }

At this point we should have a working FreeRTOS tick.  Make sure that you
consistently use milliseconds throughout your code rather than raw ticks, for
example when calling `vTaskDelay` -- that is, pass in a number of ticks needed
for a certain number of milliseconds.

## Tick Suppression

We need to supply FreeRTOS with a `vPortSuppressTicksAndSleep` method.  This is
called when the system is going to sleep and takes as an argument an expected
idle (sleep) time in number of ticks.  This method must:

1. pause the system tick counter
2. make sure we can really go to sleep (if we cannot, resume the tick counter and give up)
3. arm the sleep timer by changing its settings and connecting a dummy callback (not the FreeRTOS tick callback).
4. go to sleep
5. on wakeup, determine how long we actually slept (it may be less than or equal to the requested idle time since we may have woken up due to some interrupt) and inform FreeRTOS by adjusting the internal tick count with `vTaskStepTick`.

We should perform these steps under a (partial) critical section -- we want to
suspend interrupts but not the scheduler itself.  For convenience, we can
handle resuming the "system tick" timer in a helper method, for example:

    static inline void resume_system_tick(void)
    {
        /* Disconnect callback */
        tc_disable_callback(&tc, TC_CALLBACK_CC_CHANNEL0);
        tc_unregister_callback(&tc, TC_CALLBACK_CC_CHANNEL0);

        /* Connect the FreeRTOS system tick callback */
        tc_register_callback(&tc, (tc_callback_t)xPortSysTickHandler,
            TC_CALLBACK_CC_CHANNEL0);
        tc_enable_callback(&tc, TC_CALLBACK_CC_CHANNEL0);

        /* Resume system tick, the starting count is taken from whatever
           is in the counter right now.  This lets us literally resume
           or otherwise pre-load the counter. */
        tc_set_top_value(&tc, TIMER_RELOAD_VALUE_ONE_TICK);
        tc_start_counter(&tc);
    }

Due to the ASF TC design, we also need a dummy callback that takes the place of
`xPortSysTickHandler` when we are in the tick suppression (sleep) state:

    static void empty_cb(struct tc_module *const module_inst) { }

We also need to account for a few things:

    /* Number of timer counts that make up one RTOS tick. */
    #define TIMER_COUNTS_ONE_TICK   ((system_gclk_gen_get_hz(GCLK_GENERATOR_5)) / configTICK_RATE_HZ)

    /* The maximum number of ticks we can suppress: that is, the number
       of ticks that fit into our 32-bit counter. */
    #define MAX_SUPPRESSED_TICKS     (0xFFFFFFFFUL / (unsigned long)TIMER_COUNTS_ONE_TICK)

We now implement `vPortSuppressTicksAndSleep`:

    void vPortSuppressTicksAndSleep(TickType_t xExpectedIdleTime)
    {
        /* Make sure the expected idle time is in range */
        if (xExpectedIdleTime > MAX_SUPPRESSED_TICKS)
            xExpectedIdleTime = MAX_SUPPRESSED_TICKS;

        /* Pause the system tick timer while we reconfigure it */
        tc_stop_counter(&tc);
        /* Save the counter value at this time so that we can use it
           for tick accounting on wakeup. */
        uint32_t last_count = tc_get_count_value(&tc);
        /* Make sure that the overflow interrupt flag is cleared to avoid a
           spurious event */
        tc.hw->COUNT32.INTFLAG.bit.OVF = 1;

        /* Enter critical section */
        portDISABLE_INTERRUPTS();
        __asm volatile("dsb");
        __asm volatile("isb");

        switch (eTaskConfirmSleepModeStatus()) {
            case eAbortSleep: /* Never mind, back to system tick */
                resume_system_tick();
                break;

            /* We are going to sleep indefinitely, an interrupt will
               wake us up.  In this implementation we do not have a
               way to count how long we slept if we were to actually
               do that, so we will treat this like eStandardSleep with
               maximum sleep time instead. */
            case eNoTasksWaitingTimeout:
                xExpectedIdleTime = MAX_SUPPRESSED_TICKS;
                /* fall through... */
            /* We are going to sleep for the specified amount of time
               (via wakeup alarm) or until another interrupt wakes us
               up. */
            case eStandardSleep:
                /* Configure desired low-power state.  This is up to
                   you and you may want to pick a state based on resume
                   time versus the expected sleep time or other factors.
                   See the ASF documentation for details about
                   SYSTEM_SLEEPMODE_IDLE_2 for instance */
                system_set_sleepmode(SYSTEM_SLEEPMODE_STANDBY);

                /* Disconnect the system tick */
                tc_disable_callback(&tc, TC_CALLBACK_CC_CHANNEL0);
                tc_unregister_callback(&tc, TC_CALLBACK_CC_CHANNEL0);

                /* Connect the dummy callback */
                tc_register_callback(&tc, empty_cb,
                    TC_CALLBACK_CC_CHANNEL0);
                tc_enable_callback(&tc, TC_CALLBACK_CC_CHANNEL0);

                /* Set our wakeup alarm */
                tc_set_count_value(&tc, 0);
                tc_set_top_value(&tc,
                    (xExpectedIdleTime * TIMER_COUNTS_ONE_TICK) - 1));
                
                /* Start timer */
                tc_start_counter(&tc);

                /* Enter low power sleep mode (ie: wfi) */
                system_sleep();

                /* We just woke up.  How long did we sleep for? */

                /* A counter overflow means that we slept for at
                   least the expected time, so we can go ahead
                   and adjust the RTOS tick count by that amount
                   right now. */
                if (tc.hw->COUNT32.INTFLAG.bit.OVF)
                    vTaskStepTick(xExpectedIdleTime);

                /* We must also adjust for time left over (if any) */
                uint32_t cur_count = (last_count +
                    tc_get_count_value(&tc));
                vTaskStepTick(cur_count / TIMER_COUNTS_ONE_TICK);
                    TIMER_COUNTS_ONE_TICK);
                
                /* Resume the system tick, starting at approximately the
                   remainder of time until the next tick (that is, what
                   we could not account for above) */
                tc_set_count_value(&tc,
                    cur_count % TIMER_COUNTS_ONE_TICK);
                resume_system_tick();
                break;
        }

        /* Leave critical section */
        portENABLE_INTERRUPTS();
        __asm volatile("dsb");
        __asm volatile("isb");
    }

Please note that I am handling `eNoTasksWaitingTimeout` as if it is
`eStandardSleep` (with maximum sleep time) in the above implementation.  This
ensures that we account for sleep time with just our simple timer-counter block.
 A different MCU (or wakeup source) could allow us to implement `eNoTasksWaitingTimeout` as an indefinite sleep like you see in the example in the FreeRTOS documentation.  With a slow enough tick I feel that this is not a bad design decision -- the MCU will sleep for quite a long time before waking up and returning to sleep in the `eNoTasksWaitingTimeout` case.
