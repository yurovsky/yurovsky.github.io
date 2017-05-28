---
layout: post
title: Button (or other GPIO pin) debouncing
---

(this is mostly a repost from [back in 2011 on my old blog](http://yurovsky.blogspot.com/2011/02/button-or-other-gpio-pin-debouncing.html) but I keep referring to it so I wanted it in an easier place to find)

GPIO pin de-bouncing is a fairly common task and there are many good ways to implement it.  Here's how I handle it on most projects, I think that it's fairly
clean and easy to adapt to small microcontrollers or even some larger systems.

Each pin that needs to be sampled and debounced can be represented with a state machine comprised of a state, value, and counter.  A pin is either idle (whether pressed or released) or in transition to being pressed or released.  To transition to the idle state, the pin must maintain the same level for a number of counts.

A pin can therefore be represented something like:

    enum pin_state {
            PIN_IDLE       = 0,
            PIN_PRESSING   = 1,
            PIN_RELEASING  = 2,
    };
    
    struct pin {
            enum pin_state state;
            char pressed;
            unsigned char debounce;
            unsigned char debounce_threshold;
    };

I use three states but with the combination of 'state' and 'pressed' and 'debounce' we really have four real states (idle-pressed, idle-released, pressing, and releasing).

At initialization time, the pin structure(s) should be set to zero.  The 'pin' structure should also contain information about the pin to enable a routine to check its value (for example the GPIO port and pin number).  We then poll the pin or pins in a thread or main loop.  For example, to check just one pin:

    static struct pin;

    void init(void)
    {
            /* (if needed) */
            memset(&pin, 0, sizeof(pin));
            
            /* pick some reasonable threshold, this is a
               factor of your circuit and polling
               frequency */
            pin.debounce_threshold = 10;
    }

    void check_pins(void)
    {
            /* invert this if the pin is active-low, as is common
               for buttons, we treat a '1' as 'active' */
            char cur = gpio_get_pin_value();

            switch (pin.state) {
                  case PIN_IDLE:
                         if (cur != pin.pressed) {
                                pin.state = cur ?
                                        PIN_PRESSING : PIN_RELEASING;
                         }
                         break;
                  case PIN_PRESSING:
                         if (cur) {
                                pin.debounce++;
                         } else {
                                pin.debounce = 0;
                                pin.state = PIN_IDLE;
                         }
                         break;
                  case PIN_RELEASING:
                         if (cur) {
                                pin.debounce = 0;
                                pin.state = PIN_IDLE;
                         } else {
                                pin.debounce++;
                         }
                         break;
           }

           if (pin.state > PIN_IDLE &&
                          pin.debounce > pin.debounce_threshold) {
                  /* report the pin press or release */
                  report_pin(cur);
                  /* and now the pin is idle */
                  pin.state = PIN_IDLE;
                  pin.debounce = 0;
                  pin.pressed = cur;
           }
    }

If there are multiple pins to check then I would replace the single
`struct pin` with an array and loop over them.
In that case `struct pin` should contain pin port and pin number information
for your implementation of  `gpio_get_pin_value()`.

When debouncing a physical button we generally shoot for around 50ms (that is,
a level change below 50ms is filtered out). We usually wind up with 47ms when
debouncing with an RC circuit (typical values are a 4.7k Ohm resistor and a 100nF capacitor making RC equal to 47ms) so either one seems like a good target.

The very generic state machine described above just uses a counter and this can
be calculated or experimentally calibrated to the system based on the behavior
of that system's GPIOs. If there is a time source available, for example a timer block or an RTC with reasonable precision, this counter be
replaced with a time stamp and it's much easier to set the time threshold for
debouncing.
