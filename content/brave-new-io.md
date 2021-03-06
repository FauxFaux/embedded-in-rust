+++
author = "Jorge Aparicio"
date = 2018-01-18T00:44:18+01:00
draft = false
tags = ["I/O", "microcontroller"]
title = "Brave new I/O"
+++

Hey there! It's been a while. I've been working on some cool stuff for you. Now that's in more or
less good shape I can blog about it!

This blog post introduces our new approach to I/O in embedded contexts.

# Overview: The register model

First some background information

In microcontrollers all external I/O requires interacting with *peripherals*. Peripherals are
additional pieces of electronics that sit in the same chip / package as the core processor.

Communication between the processor and peripherals occurs through a shared memory region. This
memory region is logically split in *registers*. Each register is usually machine word sized (e.g.
32 bits on ARMv7-M), to guarantee atomic (single instruction) memory accesses to it. All registers
associated to a single peripheral are usually located in a contiguous memory location; this group of
registers is referred to as register block.

Thus all I/O boils down to memory operations on these registers.

# The old approach

The old approach follows the C model of having peripherals' register blocks as global (read:
globally visible and accessible) resources.

In C, this model has the issue that two execution contexts (e.g. threads / interrupts) may perform
non atomic operations (e.g. a read-modify-write operation) on the same register leading to a *race
condition* (\*). See below:

> (\*) not a UB triggering data race. AFAIS UB (torn reads / write and misoptimization) can't
> really occur because operations on registers are marked as volatile and they are atomic (single
> instruction); microcontrollers are (usually) single core; and the memory model of the register
> memory region is non-weak.

``` c
// defined elsewhere
static volatile int* GPIOA_ODR = 0x48000014;

void main() {
  while (1) {
    *GPIOA_ODR += 1;
  }
}

// can preempt `main`
void interrupt_handler() {
  *GPIOA_ODR *= 2;
}
```

Here `main` and `interrupt_handler` race each other and this can cause the effects of
`interrupt_handler` to sometimes be lost.

In Rust, we improved this situation by making the register blocks not *directly* accessible. Instead
you had to use a critical section (or, in [Real Time For the Masses][RTFM] (RTFM), prove the
compiler that no preemption was possible) to access the register block. We also made register
manipulation fully type safe by generating an API from vendor provided [System View
Description][SVD] (SVD) files using a tool called [`svd2rust`].

[RTFM]: https://docs.rs/cortex-m-rtfm/0.3.0/cortex_m_rtfm/
[SVD]: https://www.keil.com/pack/doc/CMSIS/SVD/html/index.html
[`svd2rust`]: https://docs.rs/svd2rust/0.12.0/svd2rust/

Here's the previous C example ported to the old Rust model.

``` rust
// generated by svd2rust
const GPIOA: Peripheral<GPIOA> = unsafe { Peripheral::new(0x4800_0000) };
// also generated by svd2rust
#[repr(C)]
struct GPIOA {
    moder: MODER, otyper: OTYPER, ospeedr: OSPEEDR, pupdr: PUPDR, idr: IDR,
    odr: ODR, /* .. */
}

fn main() {
    loop {
        // critical section: `interrupt_handler` can't preempt this closure
        interrupt::free(|cs| {
            GPIOA.borrow(cs).odr.modify(|r, w| w.bits(r.bits() + 1));
        });
    }
}

// can preempt `main`
fn interrupt_handler() {
    interrupt::free(|cs| {
        GPIOA.borrow(cs).odr.modify(|r, w| w.bits(r.bits() * 2));
    });
}
```

The Rust version doesn't have the race condition that the C version had. That coupled with the type
safe API, which prevents you from misusing the registers (i.e. writing to the reserved portions of
it), was a great improvement but was not enough for creating solid higher level abstractions.

## The hole in the old model

The Achilles heel of this model, and the C model, is the *global access* property of peripherals.
That simply breaks all abstractions. Here's an example:

Correct clock configuration (there are several clocks running at different frequencies in a
microcontroller) is important for the correct operation of peripherals, like the USART, that deal
with asynchronous communication protocols, like serial communication. In asynchronous communication
protocols both sides of the communication channel have to agree on some baud rate (bit rate) before
the communication takes place. The communication only works if both sides operate at the agreed upon
baud rate.

Say you create a `Serial` abstraction that reads the RCC peripheral, which controls the clock
configuration, during initialization to configure the USART1 peripheral to operate at the right baud
rate:

``` rust
struct Serial<'a> {
    usart: &'a mut USART1,
}

impl<'a> Serial<'a> {
    pub fn new(usart1: &'a mut USART1) -> Self { /* .. */ }
    pub fn init(&mut self, rcc: &RCC, baud_rate: u32) { /* .. */ }
    pub fn write(&mut self, byte: u8) -> Result<(), Error> { /* .. */ }
}
```

You use the abstraction in some RTFM application where a task has exclusive access to the USART1
peripheral and you are able to use the `Serial` abstraction without locks.

``` rust
// rtfm: v0.2.0

app! { .. }

fn init(p: init::Peripherals, r: init::Resources) {
    Serial(p.USART1).init(p.RCC, 115_200);
    // ..
}

fn task(t: &mut Threshold, r: EXTI0::Resources) {
    let mut serial = Serial(r.USART1);
    // do stuff with `serial`
}
```

All seems good. Except that there's *no* guarantee that RCC will not be modified by some other task
thus there's no guarantee that the clock configuration will remain the same for all the executions
of `task`. The other task doesn't even have to name the RCC as one of its resources because the RCC
has global visibility so any task can always access it (this is actually [an old RTFM bug] that's
only occurs with the old I/O model):

[an old RTFM bug]: https://github.com/japaric/cortex-m-rtfm/issues/13

``` rust
fn foo() {
    interrupt::free(|cs| {
        let rcc: &RCC = RCC.borrow(cs);
        // modify `RCC` to tweak the clock configuration to, say, halve all the
        // clock frequencies
    });
}

fn some_other_task(t: &mut Threshold, r: EXTI1::Resources) {
    foo();
}
```

If this happens then the serial communication will fail: the hardware will raise some "framing"
error flag to signal the baud rate mismatch.

(Some of you may be thinking "but then you should store a reference to `RCC` in `Serial` to make
sure the clock configuration can't change". Unfortunately that's not enough because `Serial` has to
be *re-constructed* every time the task starts so RCC can still be modified by other task between
the end of `task` and the next time it's invoked)

Once you know about this hole then you conclude that no abstraction around the `svd2rust` API can
hold their invariants because *peripherals are always open to modification, even by crates created
by a third party*. Yikes!

# The new approach

The new I/O model (`svd2rust` v0.12.x) removes the root of the problem: it no longer provides global
access to peripherals. Instead peripherals need to be `take`n (read: moved) *into* the current
execution context. See below:

``` rust
// stm32f30x was generated using `svd2rust`
extern crate stm32f30x;

use stm32f30x::{GPIOA, Peripherals};

fn main() {
    let p = stm32f30x::Peripherals::take().unwrap();

    // difference: this is now an *owned* value, not a reference
    let gpioa: GPIOA = p.GPIOA;

    // same register manipulation API as before
    gpioa.odr.modify(|r, w| w.bits(r.bits() * 2));
}
```

The peripherals can only be `take`n once. `Peripherals::take` returns an `Option` that will only be
`Some` the first time the method is called; subsequent calls to `Peripheral::take` will return the
`None` variant.

``` rust
fn main() {
    let ok = Peripherals::take().unwrap();
    let panics = Peripherals::take().unwrap();
}
```

This effectively makes each peripheral a singleton because there can only ever be at most one
instance of them at any point in time. In the old model peripherals were also singletons but they
were *global* singletons: singletons accessible by anyone. In the new model peripherals are *scoped*
singletons that are *owned* by the execution context (task / thread) from where `Peripherals::take`
was called. A peripheral is not, forever, tied to a single execution context, though; it can be
*moved* into another execution context if desired because every peripheral is `Send`able.

How does this new ownership based model affect the way we create higher level abstractions? Let's
see.

## Freezing the clock configuration

Before I showed that changing the clock configuration can be problematic as it can affect the
operation of peripherals like the USART. Now that we have ownership over peripherals, making sure
that the clock configuration never changes is very easy: you simply `drop` the `RCC` peripheral.
Once `RCC` is gone the RCC registers can no longer be modified thus the clock configuration will
remain unchanged from that point and on.

RCC stands for Reset and Clock Control and controls not only the clock configuration but it's
responsible for enabling, resetting and disabling other peripherals. Dropping the whole thing would
mean that we can no longer enable other peripherals and most peripherals start disabled at boot time
to save power so maybe it's not the best of ideas.

What we can do instead is *split* RCC into parts in charge of different functionalities:

``` rust
struct Parts { ahb: AHB, apb1: APB1, apb2: APB2, cfgr: CFGR }

trait RccExt {
    /// Constrains RCC functionality by splitting it in parts
    fn constrain(self) -> Parts;
}

impl RccExt for RCC { .. }
```

`CFGR` controls the clock configuration and the other parts are in charge of enabling and disabling
peripherals. Note that `constrain` *consumes* the original `RCC` which granted *full access* to
every RCC register; this effectively *constrains* the operations that can be performed on RCC
registers to only the ones exposed by the members of `Parts`.

With a bit more of work we can achieve something like this:

``` rust
fn main() {
    let p = Peripherals::take().unwrap();
    let rcc = p.RCC.split();

    let clocks: Clocks = rcc.cfgr.sysclk(64.mhz()).pclk1(32.mhz()).freeze();

    assert_eq!(clocks.sysclk(), 64.mhz());
    assert_eq!(clocks.hclk(), 64.mhz());
    assert_eq!(clocks.pclk1(), 32.mhz());
    assert_eq!(clocks.pclk2(), 32.mhz());

    // can still use `rcc.{ahb,apb1,apb2}` over here
}
```

`CFGR` exposes a builder-like API to pick the frequency of each clock. The final `freeze` method
makes the configuration effective, consumes `CFGR` and returns a `Clocks` value. `Clocks` is a
`Copy`-able struct that contains the *frozen* clock configuration. `Clocks` can be used in the
initialization of abstractions like `Serial`; its very existence holds the invariant that the clock
configuration will not change.

## Typed configuration

Physical pins on a microcontroller can be configured as digital inputs, as digital outputs or as
peripheral pins (pins associated to peripherals like the USART). We can encode this configuration in
the *type* of a pin to enforce at compile time that a pin has a certain configuration. We can think
of the different configurations as the different states the pin can be in. This pattern of encoding
states in the type system is known as *type state* (some people also call it *session types*).

Let's see how this would apply in practice:

<!-- extern crate stm32f30x_hal as hal; -->

<!-- use hal::gpio::gpioa::PA9; -->
<!-- use hal::gpio::{Floating, Input, Output, PushPull}; -->
<!-- use hal::prelude::*; -->
<!-- use hal::stm32f30x::Peripherals; -->


``` rust
fn main() {
    let p = Peripherals::take().unwrap();

    let mut rcc = p.RCC.constrain();

    // splits the GPIOA peripheral into 16 independent pins + registers
    let mut gpioa = p.GPIOA.split(&mut rcc.ahb);
    // all pins start as floating inputs
    let pa9: PA9<Input<Floating>> = gpioa.pa9;

    // API available in the `Input<_>` state
    if pa9.is_low() {
        // ..
    } else if pa9.is_high() {
        // ..
    }

    // configure the pin PA9 as an output
    // this operation consumes the original `pa9` value
    let mut pa9: PA9<Output<PushPull>> =
        pa9.into_push_pull_output(&mut gpioa.moder, &mut gpioa.otyper);

    // API available in the `Output<_>` state
    pa9.set_low();
    pa9.set_high();
}
```

The `gpioa.moder` and `gpioa.otyper` values are proxies for the MODER and OTYPER registers. The API
requires explicitly passing them to avoid a race condition in the rare case that you want to
configure GPIOA pins from different execution contexts that can preempt each other; in that scenario
the API will force you to use a critical section to access the proxies.

It may seem redundant to be explicit about these registers when all the configuration is done in a
single execution context, but by having these explicit proxies you can be sure that once they are
destroyed the configuration of the GPIOA pins can't be changed anymore.

## No pin overlap

Modern microcontrollers pack lots of peripherals in a single package-- sometimes so many peripherals
that not all of them can be used at the same time. This is actually normal because most applications
won't use all of them; instead each application will use different kinds of peripherals and
different number of instances of each kind.

For maximum flexibility vendors don't hardwire peripherals to physical pins (mainly because there's
not enough physical pins in the first place) instead one can configure the function of a pin at
*runtime* -- this effectively associates the pin to a particular peripheral. But it's definitively
an error to associate a single pin to more than one peripheral. With move semantics we can reject
such misconfigurations at compile time:

<!-- extern crate stm32f30x_hal as hal; // v0.1.0 -->

<!-- use hal::i2c::I2c; -->
<!-- use hal::prelude::*; -->
<!-- use hal::serial::Serial; -->
<!-- use hal::stm32f30x::Peripherals; -->

``` rust
fn main() {
    let p = Peripherals::take().unwrap();

    let mut flash = p.FLASH.constrain();
    let mut rcc = p.RCC.constrain();

    let mut gpioa = p.GPIOA.split(&mut rcc.ahb);
    let clocks = rcc.cfgr.freeze(&mut flash.acr);

    // use pins PA9 and PA10 with USART1
    // the `Serial::usart1` constructor requires the pins to be configured with
    // the right Alternate Function (AF); it also consumes the pins
    let tx = gpioa.pa9.into_af7(&mut gpioa.moder, &mut gpioa.afrh);
    let rx = gpioa.pa10.into_af7(&mut gpioa.moder, &mut gpioa.afrh);
    let serial =
        Serial::usart1(p.USART1, (tx, rx), 9_600.bps(), clocks, &mut rcc.apb2);

    // try to use PA9 and PA10 with I2C2
    let scl = gpioa.pa9.into_af4(&mut gpioa.moder, &mut gpioa.afrh);
    //~^ error: use of move value: `gpioa.pa9`
    let sda = gpioa.pa10.into_af4(&mut gpioa.moder, &mut gpioa.afrh);
    //~^ error: use of move value: `gpioa.pa10`
    let i2c = I2c::i2c2(p.I2C2, (scl, sda), 400.khz(), clocks, &mut rcc.apb1);
}
```

---

And this is just the tip of iceberg of the things you can do with the new I/O model.

# Layered I/O

Let's now look at another aspect of the I/O story: code reuse.

The API that `svd2rust` generates serves as the lowest abstraction layer sitting very close to the
hardware and providing an API for directly manipulating registers. Most people will want to program
with higher level abstractions that map closer to actions like "read this sensor", "send data
through this interface", etc.

This section covers my plan for building those higher level abstractions with minimal code
duplication.

## Thou shalt not block

I/O can be performed in a blocking fashion or in an asynchronous fashion. And there's more than one
way to do asynchronous I/O: there's the callback model ("run this function when data is ready"), the
futures model ("a future represents an I/O action that will be completed some time in the future;
poll to see if the I/O has completed or not") and the generators model ("yield control, instead of
blocking, when no more progress can be made").

We can implement a blocking API on top of the `svd2rust` API; we can implement a futures based
version of the API on top of the `svd2rust` API; and we can implement a generator based version of
the API on top of the `svd2rust` API. The implementations of those three flavors of the same API
will look very similar. The question is: can we implement some intermediate API to avoid *writing
register manipulation* code in those three implementations? The answer is the [`nb`] crate.

[`nb`]: https://docs.rs/nb/0.1.1/nb/

I pulled a solution out of the Tokio book, or maybe it's actually from the *nix book. The solution I
chose was to implement the intermediate API as a non blocking API that returns a `WouldBlock` error
variant to signal that the operation can *not* be completed right now. This non fatal error variant
has a different meaning depending on the (a)synchronous model is used in:

- In a blocking API, `WouldBlock` means try again i.e. busy wait (or maybe do something more
  elaborated if you have a threading system).
- In a futures based API, `WouldBlock` means `Async::NotReady`
- In a generator based API, `WouldBlock` means `yield`

### Async, your way

Let's look at an example. Below is shown the `nb` API for reading a single byte from a serial
interface. `Serial::read` will return `Ok(byte)` if new data is available to be read; it will return
`Err(nb::Error::WouldBlock)` if no new data is available at the moment; and it will return e.g.
`Err(nb::Error::Other(Error::Overrun))` if some fatal error occurred.

``` rust
enum Error {
    Noise,
    Framing,
    Overrun,
    // ..
}

impl Serial {
    pub fn read(&mut self) -> nb::Result<u8, Error> { /* .. */ }
}
```

The [`nb`] crate provides macros to easily transform this intermediate API into the (a)synchronous
models I mentioned:

[`block!`] is used to implement blocking APIs by busy waiting.

``` rust
fn blocking_read(serial: &mut Serial) -> Result<u8, Error> {
    block!(serial.read())
}
```

[`block!`]: https://github.com/japaric/nb/blob/v0.1.1/src/lib.rs#L428-L456

[`try_nb!`] is used to implement futures based APIs.

``` rust
fn futures_read(serial: Serial) -> impl Future<(Serial, u8), Error> {
    let mut serial = Some(serial);
    futures::poll_fn(move || {
        let byte = try_nb!(serial.as_mut().unwrap().read());
        Ok(Async::Ready((serial.take().unwrap(), byte)))
    })
}
```

[`try_nb!`]: https://github.com/japaric/nb/blob/v0.1.1/src/lib.rs#L458-L495

[`await!`] is used to implement generator based APIs.

``` rust
fn generator_read(
    mut serial: Serial,
) -> impl Generator<Return = Result<(Serial, u8), Error>, Yield = ()> {
    let byte = await!(serial.read())?;
    Ok((serial, byte))
}
```

[`await!`]: https://github.com/japaric/nb/blob/v0.1.1/src/lib.rs#L389-L426

None of these higher level APIs required writing any register manipulation code and the functions
could even have been made generic if `Serial::read` was a trait method instead of an inherent
method. Which brings me to the next topic.

## HAL traits

The `nb` crate provides an easy transition from the svd2rust API to the different (a)synchronous
models but for code reuse we have to write generic code and this means that we have to establish a
set of traits to build that generic code upon.

Enter the [`embedded-hal`] crate.

[`embedded-hal`]: https://docs.rs/embedded-hal/0.1.0

This crate is basically a collection of traits that provides interfaces for the different
peripherals available on a embedded device. These traits form a Hardware Abstraction Layer, an
abstraction that hides device specific details like registers.

At the center of this crate we have `nb` based traits that make functions like the `Serial::read`
method we saw before more generic:

``` rust
pub mod serial {
    pub trait Read<Word> {
        type Error;

        /// Reads a single word from the serial interface
        fn read(&mut self) -> nb::Result<Word, Self::Error>;
    }
}
```

Then we have slightly higher level traits that actually pick an (a)synchronous model. Where possible
opt-in default implementations on top of the `nb` traits are provided.

``` rust
/// Blocking API
pub mod blocking {
    /// Blocking SPI API
    pub mod spi {
        /// Blocking transfer
        pub trait Transfer<Word> {
            type Error;

            /// Sends `words` to the slave. Returns the `words` received from
            /// the slave
            fn transfer<'w>(
                &mut self,
                words: &'w mut [Word],
            ) -> Result<&'w [Word], Self::Error>;
        }

        /// Blocking transfer
        pub mod transfer {
            /// Marker trait to opt into a default implementation of
            /// `blocking::spi::Transfer<W>`
            pub trait Default<W>: ::spi::FullDuplex<W> {}

            impl<W, S> blocking::spi::Transfer<W> for S
            where
                S: Default<W>,
                W: Clone,
            {
                type Error = S::Error;

                fn transfer(
                    &mut self,
                    words: &'w mut [Word],
                ) -> Result<&'w [Word], S::Error> {
                    for word in words.iter_mut() {
                        block!(self.send(word.clone()))?;
                        *word = block!(self.read())?;
                    }

                    Ok(words)
                }
            }
        }
    }
}
```

### Write once, run everywhere

The end goal of having these traits is writing *generic* drivers: library crates that let you
interface external components but that -- thanks to the HAL -- whose implementation is oblivious
to the hardware details of the platform they are running on.

SPI and I2C are two widely used communication protocols and there are several external components,
like digital sensors, out there that can be interfaced using one of these two communication
protocols. The `embedded-hal` crate contains blocking traits for these two communication protocols
and, as a proof of concept, I have developed a few *no_std* generic drivers on top of those traits:

- [`l3gd20`], gyroscope found on the STM32F3DISCOVERY board
- [`lsm303dlhc`], accelerometer + compass found on the STM32F3DISCOVERY board
- [`mfrc522`], RFID tag reader / writer
- [`mpu9250`], Nine-Axis (Gyro + Accelerometer + Compass) MEMS MotionTracking Device

[`l3gd20`]: https://docs.rs/l3gd20/0.1.0/l3gd20/
[`lsm303dlhc`]: https://docs.rs/lsm303dlhc/0.1.0/lsm303dlhc/
[`mfrc522`]: https://github.com/japaric/mfrc522
[`mpu9250`]: https://github.com/japaric/mpu9250

As an example, let's look at the `mfrc522` driver:

``` rust
#![no_std]

extern crate embedded_hal as hal;

/// MFRC522 driver
struct Mfrc522<SPI, NSS> where
    SPI: hal::blocking::spi::Transfer<u8> + hal::blocking::spi::Write<u8>,
    NSS: hal::digital::OutputPin,
{
    spi: SPI,
    nss: NSS,
}
```

The driver uses a SPI bus and a NSS (slave select) pin to interface with a MFRC522. The driver is
generic over both the SPI bus and the NSS pin; this means any abstraction that implements the
required traits can be used here. For instance, the [`linux-embedded-hal`] crate which implements
the `embedded-hal` traits for the [`spidev`] and [`sysfs_gpio`] crates can be used with this driver
to build a [program that targets a platform running Linux][rpi], like the Raspeberry Pi.

[`linux-embedded-hal`]: https://crates.io/crates/linux-embedded-hal
[`spidev`]: https://crates.io/crates/spidev
[`sysfs_gpio`]: https://crates.io/crates/sysfs_gpio
[rpi]: https://github.com/japaric/mfrc522/blob/v0.1.0/examples/rpi.rs#L68

<p align="center">
  <video controls>
    <source src="/brave-new-io/rpi.webm" type="video/webm">
  </video>
</p>

<p align="center">
  <em>The blue RFID tag turns the green LED (see bottom right corner) on while the RFID card turns
  it off.</em>
</p>

Since the crate is a `no_std` crate it can also be used to build a `no_std` program that targets an
ARM Cortex-M microcontroller. (The source for this second program will be in the [`blue-pill`]
repository once I finish cleaning it up.)

[`blue-pill`]: https://github.com/japaric/blue-pill

<p align="center">
  <video controls>
    <source src="/brave-new-io/cortex-m.webm" type="video/webm">
  </video>
</p>

So, by using the `embedded-hal` traits driver authors can effortlessly support a bunch of diverse
platforms (AVR, ARM Cortex-M, MSP430, RISC-V microcontrollers; ARM Cortex-A single board computers;
etc.) in a single crate. Whereas application developers, by implementing the `embedded-hal` traits
for their platform, can unlock a bunch of drivers at zero extra effort.

## Crate hierarchy for Cortex-M

Here's the whole I/O picture for the Cortex-M ecosystem.

At the lowest level we have the device crates generated by `svd2rust`. These expose type safe APIs
for directly manipulating registers. Depending on the source SVD file a device crate can support a
single device or several device families. For example, the `stm32f30x` crate supports device
families STM32F301xx, STM32F302xx and STM32F303xx.

At the next level you have *HAL implementation* crates like the [`stm32f30x-hal`] that implement the
`embedded-hal` traits for a single device crate. These crates expose higher level APIs for `Serial`
interfaces, `SPI` interfaces, clock configuration, etc. and it's what most people will be using.
These crates let you use generic drivers to interface external components.

[`stm32f30x-hal`]: https://crates.io/crates/stm32f30x-hal

There's one more level: *board support* crates.

HAL impl crates are highly generic: they let you configure peripherals to work with any valid pin
configuration. This is reflected in the types exposed by the crate:

``` rust
// crate: stm32f30x-hal

// `Serial` is generic over
// - the USART peripheral, which can be USART1, USART2, etc.
// - the TX and RX pins, which can be PA9 and PA10 or PB7 and PB8 for USART1, etc.
struct Serial<USART, TX, RX> where
    TX: TxPin<USART>,
    RX: RxPin<USART>,
    ..
{ .. }

impl<TX, RX> Serial<USART1, TX, RX> where .. {
    pub fn usart1(usart: USART1, (tx, rx): (TX, RX), ..) -> Self { .. }
}
```

But PCBs already have their microcontroller's pins routed to external components; this means that
some peripherals / pins *have* to be configured in a certain way. Thus board support crates should
narrow down the possible configurations of a peripheral to a single configuration to avoid
misconfigurations. Board support crates can achieve that by exposing type aliases that turn the
generic types from HAL impl crates into concrete types:

``` rust
// crate: f3 (board support crate for the STM32F3DISCOVERY)

// driver crate that exposes a generic `Lsm303dlhc<I2C> where I2C: ..` type
extern crate lsm303dlhc;

// Pins PA9 and PA10 have been routed to e.g. an UART <-> USB adapter so they
// can only be used for serial communication
// `Serial1` represents that serial interface
pub type Serial1 = stm32f30x_hal::Serial<USART1, PA9<AF7>, PA10<AF7>>;

// LSM303DLHC is connected to the microcontroller via the I2C1 bus lines: PB6
// and PB7
pub type Lsm303dlhc = lsm303dlhc::Lsm303dlhc<I2c<I2C1, (PB6<AF4>, PB7<AF4>)>>;
```

This means that we can *specify board connections at the type level* and have them enforced by the
compiler: if, for example, you try to construct a `f3::Lsm303dlhc` value using pins other than PB6
and PB7 you'll get a compiler error.

# Conclusion

The new I/O model brings move semantics to the table which lets us exploit the type system (mainly
using the type state pattern) to check configurations at compile time and have to compiler reject
wrong configurations.

The new model also serves as a solid foundation on top of which we can build generic drivers using
the `embedded-hal` traits.

The best part is that these two advantages are orthogonal. Perhaps you are not too crazy about the
type state stuff and think it's too much work for little gain. That's totally OK! You can still
benefit from the generic driver crates if you implement the `embedde-hal` traits; type state is not
a requirement for using generic driver crates.

# What's next

## Async HAL

All my proof of concept driver crates expose a blocking API. We want to explore writing
asynchronous driver crates using futures / generators. We are kind of blocked on that front though
because to write truly generic async driver code we would need HAL traits whose methods return
futures or generators and `impl Trait` doesn't work in that position yet:
``` rust
// I2C async write
trait Write {
    type Error;

    fn write(
        self,
        addr: u8,
        bytes: [u8; 16], // NOTE should be generic over the size
    ) -> impl Generator<Return = Result<(Self, [u8; 16]), Self::Error>, Yield = ()>;
    //~^ error: `impl Trait` not allowed outside of function and inherent method return types
}
```

Using boxed trait objects is not really an option as some applications will prefer to leave out
the memory allocator. It might be possible to walk down the futures route right now if we are OK
with using really long (lasagna) types that implement the `Future` trait as the return types ...

## Reduce implementation work

With the crate hierarchy I described before `svd2rust` does 90% of the work by generating the device
crates for us. Writing those by hand would be extremely tedious and error prone; it would also take
lots of human hours to completely write a single device crate.

The next level of the hierarchy, the HAL impl crates, need to be written by a human because
semantics need to be understood ("what does this register do?") and tools can't do those for us.
However, the way `svd2rust` currently works will let us to write a bunch of duplicated code. Let me
elaborate:

`svd2rust` produces a single device crate per input SVD file. The SVD file can be generic like the
STM32F30X.svd which covers *three* device families: STM32F301xx, STM32F302xx and STM32F303xx; or
less generic like the STM32F301X.svd, STM32F302X and STM32F303X.svd where each one targets *one* of
the device families mentioned before.

You may already see where this is going. If we generated *three* device crates and implemented
*three* HAL impl crates we would have ended with three very similar looking crates. By, instead,
implementing the HAL for the more generic device crate we reduce the amount of required work by
66%.

Vendors may not always provide these more generic SVD files but there are many similarities between
devices from the same vendor, even if they are from different product lines. If we can make
`svd2rust` find those similarities to produce *shared* crates (see below) for us we could make it do
99% of the implementation work.

Ideally `svd2rust -i *.svd` should generate crates like these:

``` rust
// crate: stm32_common
pub struct GPIOA { .. }
// ..
pub struct USART1 { .. }
pub struct USART2 { .. }
pub struct USART3 { .. }
// ..
```

``` rust
// crate: stm32f10x
extern crate stm32_common;

// NOTE no USART3 here
pub use stm32_common::{GPIOA, .., USART1, USART2, ..};
```

``` rust
// crate: stm32f20x
extern crate stm32_common;

pub use stm32_common::{GPIOA, .., USART1, USART2, USART3, ..};
```

Then the HAL implementation only needs to be done for the `stm32_common`:

``` rust
// crate: stm32_common_hal

extern crate stm32_common;

use stm32_common::USART1;

pub struct Serial<USART, ..> { .. }

// this implementation supports devices from the STM32F10x, STM32F20x, etc. families
impl hal::serial::Write for Serial<USART1, ..> { .. }
```

## Making more batteries

To encourage the use of the `embedded-hal` traits and to grow the no-std crates.io ecosystem I'm
starting [the weekly driver initiative][weekly-driver] :tada:.

[weekly-driver]: https://github.com/rust-embedded/rfcs/issues/39

The goal is to release one generic, `no_std`, `embedded-hal` based driver crate every one or maybe
two weeks. I can probably implement ~20 of these crates in a year on my own (I don't think I have
more than 30 external components with me in any case) so I hope others will join in so we can get
way more crates published. A short blog post will accompany each driver release describing its
functionality.

That's it for this post. Until next time.

---

__Thank you patrons! :heart:__

I want to wholeheartedly thank:

<p style="text-align:center">
  <a href="http://www.sharebrained.com/" style="border-bottom:0px">
    <img alt="ShareBrained Technology" src="/logo/sharebrained.png" width="200"/>
  </a>
</p>

[Iban Eguia], [Aaron Turon], [Geoff Cant], [Harrison Chin], [Brandon Edens], [whitequark],  [James
Munns], [Fredrik Lundström], [Kjetil Kjeka], Kor Nielsen and 34 more people for [supporting my work
on Patreon][Patreon].

[Iban Eguia]: https://github.com/Razican
[Aaron Turon]: https://github.com/aturon
[Geoff Cant]: https://github.com/archaelus
[Harrison Chin]: http://www.harrisonchin.com/
[Brandon Edens]: https://github.com/brandonedens
[whitequark]: https://github.com/whitequark
[James Munns]: https://jamesmunns.com/
[Fredrik Lundström]: https://github.com/flundstrom2
[Kjetil Kjeka]: https://github.com/kjetilkjeka
<!-- [Kor Nielsen]: -->

---

Let's discuss on [reddit].

[reddit]: https://www.reddit.com/r/rust/comments/7r5jqw/eir_brave_new_io/

Enjoyed this post? Like my work on embedded stuff? Consider supporting my work
on [Patreon]!

[Patreon]: https://goo.gl/DZtACV

Follow me on [twitter] for even more embedded stuff.

[twitter]: https://twitter.com/japaricious

The embedded Rust community gathers on the #rust-embedded IRC channel
(irc.mozilla.org). Join us!
