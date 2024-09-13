---
layout: single
classes: wide
author_profile: true
excerpt_separator: <!--more-->
title: "C vs. Ruby+YJIT: I2C Edition"
---
I recently added a bit-bang I2C implementation to the [lgpio gem](https://github.com/denko-rb/lgpio).

"Bit banging" means you don't use the I2C hardware built into the SBC. Instead, you manipulate a pair of regular GPIO, as fast as you can, to mimic what the hardware does.

This sounds like a bad idea. Why would you even do it? One use is if you need multiples of a device (eg. 4 of the same sensor) but they all have the same I2C address. Devices with the same address can't coexist on a bus, so you need more buses (or a multiplexer, but that's another post).

## How Bad Is It?

As a benchmark for I2C performance, I've been using a script that repeatedly fills and clears all the pixels of a 1" SSD1306 OLED, as fast as possible. It looks like this when running:

![SSD1306 OLED Benchmark](/images/oled_ssd1306.gif)

Performance is pretty good. On a Raspberry Pi 4, it does about 85% of what the hardware bus (at 400 kHz) can do, consuming a CPU core in the process. Hardware I2C doesn't use the CPU.

This is an extreme case, and it's moving upward of 300kbps here, which is a decent rate for I2C. Less demanding applications use proportionately less CPU, and work just fine.

## But Can We Make It Worse?

My implementation was written in C, as the gem is mostly Ruby bindings for the [lgpio C library](https://github.com/joan2937/lg).

After seeing [this talk](https://www.youtube.com/watch?v=qf5V02QNMnA) by Maxime Chevalier-Boisvert, where she shows that Ruby+YJIT can sometimes even outperform C extensions, I wondered if that might apply here.

We can't avoid calling C, **a lot**, to change GPIO levels , and YJIT can't optimze that. But bit-banging also involves arrays, loops, enumerators, bitwise operations, and other fun stuff. Surely it can optimize *something*.

## Ruby Implementation

To test this theory, I rewrote my original [C](https://github.com/denko-rb/lgpio/blob/e337e70bd4add3ead31e506407d1146a238a494c/ext/lgpio/lgpio.c#L683-L858) implementation as closely as I could in [Ruby](https://github.com/denko-rb/lgpio/blob/e337e70bd4add3ead31e506407d1146a238a494c/lib/lgpio/i2c_bitbang.rb). Both implementations are in the gem, and usable... for now.

The "C implementation" takes an array from Ruby, and handles all the "bit-bang logic" in C. On the other hand, the "Ruby implementation" does as much as possible in Ruby: enumerate through the array, get individual bits etc., only calling C to change GPIO levels.

##  Benchmarks

I created 2 benchmark scripts in the gem's examples folder:
  - [examples/i2c_bitbang_ssd1306_bench.rb](https://github.com/denko-rb/lgpio/blob/e337e70bd4add3ead31e506407d1146a238a494c/examples/i2c_bitbang_ssd1306_bench.rb) (C)
  - [examples/i2c_bitbang-rb_ssd1306_bench.rb](https://github.com/denko-rb/lgpio/blob/e337e70bd4add3ead31e506407d1146a238a494c/examples/i2c_bitbang-rb_ssd1306_bench.rb) (Ruby)
  
Apart from which bit-bang implementation they call, they both do the same thing: initialize the OLED, start timing, send 400 frames to it (alternating between filled and emtpy), stop timing. Finally, they calculate and print the number of frames per second (fps) achieved.

##  Test Setup

- Raspberry PI 4B 8GB, with enclosure, heatsink and fan
- CPU frequency fixed at 1800 MHz
- Raspberry Pi OS
- Kernel 6.6.47-v8+
- Rust 1.81.0
- Ruby+YJIT 3.3.5 (installed via rbenv + ruby-build)
- SCL on GPIO 5
- SDA on GPIO 6
- Only device connected to the I2C bus is the SSD 1306 OLED
- Each implementation was tested twice, with and without `--yjit`
- Each test consisted of 6 repeated runs, done consecutively
- For each test, only the 3 highest results were taken

##  Results

Results in frames per second (fps). Higher is better.

```bash
# C implementation
ruby examples/i2c_bitbang_ssd1306_bench.rb
# Results: 36.78, 35.78, 35.69 | Average: 36.08 fps
```

```bash
# C implementation + YJIT
ruby --yjit examples/i2c_bitbang_ssd1306_bench.rb
# Results: 36.77, 36.72, 36.21 | Average: 36.57 fps
```

```bash
# Ruby implementation
ruby examples/i2c_bitbang-rb_ssd1306_bench.rb
# Results: 25.99, 25.53, 24.99 | Average: 25.50 fps
```

```bash
# Ruby implementation + YJIT
ruby --yjit examples/i2c_bitbang-rb_ssd1306_bench.rb
# Results: 33.42, 32.76, 32.42 | Average: 32.87 fps
```

##  Conclusion

- As expected, YJIT only improves the C implementation by **about 1.0 %**
- Ruby+YJIT is **28.9 %** faster than plain Ruby
- Plain Ruby is only **70.7 %** as fast as C
- Ruby+YJIT is **89.9 %** as fast as the best C test

This is an impressive result, especially since we simply can't avoid a ton of C calls. No, the YJIT gain isn't multiples, like one of the examples in Maxime's talk, but who cares?
 
**The Ruby performance is getting close to C**.

Given how much easier the Ruby code is to read and work with, I'm considering removing the C implementation entirely. Maybe 3.4 will even bring the Ruby version up to par? Either way, bit-bang SPI will be Ruby-first when I get around to it.
