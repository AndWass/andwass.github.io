---
title: "Embedded async"
date: 2020-09-02T09:07:49+02:00
draft: true
---
Rust is positioning itself to become one of, if not the only, languages
that can enable true allocation-free async programming. A mature
async eco-system would in my opinion be a game-changer for embedded Rust;
something that can alter how we write embedded software completely.

## Whats the big deal?

Pretty much all embedded software can be boiled down to the following sequence of events:

  1. Initialize hardware
  2. Poll hardware state and wait for interrupts
  3. React to hardware state and interrupts
  4. Go to `2`

Now this can be done in multiple ways, for instance the Rust embedded eco-system has a neat project called
[RTIC](https://rtic.rs/0.5/book/en/) that can help you, or you write it all on your own using
more traditional code.

Consider the following piece of code, the goal is to increment `COUNTER` every time
we detect a low-to-high transition of some digial signal and turn on a LED for 100ms if the LED isn't
already turned on. The `COUNTER` will always be reset to 0 every second.

```rust
use core::sync::atomic::{AtomicUsize, Ordering};

static COUNTER: AtomicUsize = AtomicUsize::new(0);

const LED_DEALY_MS: u32 = 100;

#[entry]
fn main() -> ! {
    set_timer_1hz();
    let mut last_state = false;
    let mut led_timer_started = false;

    loop {
        let state = read_signal_level();

        if state && !last_state {
            COUNTER.fetch_add(1, Ordering::Relaxed);
            if !led_timer_started {
                led_timer_started = true;
                start_led_timer(LED_DEALY_MS);
                set_led_on();
            }
        }

        if led_timer_started && led_timer_timeout() {
            led_timer_started = false;
            set_led_off();
        }
        last_state = state;
    }
}

#[interrupt]
fn timer() {
    COUNTER.store(0, Ordering::Relaxed)
}
```

Compare the code above with the code below, which is utilizing not-yet-existing `async` functionality.

```rust
use core::sync::atomic::{AtomicUsize, Ordering};

static COUNTER: AtomicUsize = AtomicUsize::new(0);

const LED_DEALY_MS: u32 = 100;

async fn led_blinker(mut trigger: TriggerReceiver, mut timer: AsyncTimer) {
    loop {
        trigger.triggered().await;
        set_led_on();
        timer.delay(LED_DELAY_MS).await; // 100ms
        set_led_off();
        trigger.clear(); // Make the trigger available to be triggered again.
    }
}

async fn signal_processor() {
    let (mut trigger_sender, trigger_receiver) = Trigger::new();
    executor::spawn(led_blinker(trigger_receiver, AsyncTimer::new()));
    loop {
        positive_flank().await;
        COUNTER.fetch_add(1, Ordering::Relaxed);
        trigger_sender.trigger();
    }
}

#[entry]
fn main() -> ! {
    set_timer_1hz();
    executor::spawn(signal_processor());
    executor::run();
}

#[interrupt]
fn timer() {
    COUNTER.store(0, Ordering::Relaxed)
}
```

I don't know about you but to me the last code fragment is clear as to what is happening,
the one before is a complete mess in comparison!

## Where we are today

As far as I know there is no "standard" embedded executor, and [embedded-hal](https://github.com/rust-embedded/embedded-hal)
so far contains no async traits (which is understandable, this is **hard**, but that is a subject for another post).

In terms of executors, I know of two executors (both in POC/WIP state) that are specifically targetting embedded
no-alloc environments:

  * [uio](https://github.com/AndWass/uio) <- *Selfplug* this is mine
  * [static-executor](https://github.com/Dirbaio/static-executor)

These take different approaches to how tasks are stored; `static-executor` statically allocates tasks, which uses features
not available in stable Rust. The static allocation provides better compile-time safety than `uio` tasks which are stack-allocated.
`uio` on the other hand relies only on stable Rust, but panics if for instance a task is deallocated while still running.

A complete no-alloc async ecosystem needs more than tasks and executors though; traits to enable async peripheral drivers
and traits/tools to enable async middle-ware also has to be ironed out. This is the really interesting stuff, but also tricky.

