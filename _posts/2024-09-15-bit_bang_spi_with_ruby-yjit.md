---
layout: single
classes: wide
author_profile: true
excerpt_separator: <!--more-->
title: "Bit Bang SPI with Ruby+YJIT"
---
In my [previous post]({% post_url 2024-09-13-c_vs_ruby-yjit_i2c_edition %}), I tested Ruby+YJIT against C, doing bit-bang I2C in the [lgpio gem](https://github.com/denko-rb/lgpio). The results convinced me to just do [bit-bang SPI in Ruby](https://github.com/denko-rb/lgpio/blob/master/lib/lgpio/spi_bitbang.rb).

Testing YJIT vs. plain Ruby for this is trivial, but can I compare against C, without *actually writing* a C implementation? SPI is almost the same as I2C for the purpose of my OLED benchmark. Could I use the I2C results somehow?  We'll get to that, but first...

## Improving I2C

Since the last post, I made an optimization to I2C: save the SDA pin state, and only call the C API when it needs to change, so fewer calls overall.

For fairness, I did this for both implementations, and changed the [C](https://github.com/denko-rb/lgpio/blob/f5df7486ae5cda145ed812b7fa32604fd7f12556/examples/i2c_bitbang-rb_ssd1306_bench.rb) and [Ruby](https://github.com/denko-rb/lgpio/blob/f5df7486ae5cda145ed812b7fa32604fd7f12556/examples/i2c_bitbang-rb_ssd1306_bench.rb) benchmarks to use a pattern of lines. This makes it so approximately half the time SDA needs to change, and the other half, it stays. It looks like this now:

![SSD1306 OLED Lines Benchmark](/images/2024-09-15-bit_bang_spi_with_ruby-yjit/oled_ssd1306_lines.gif)

Of course, I had to redo all the benchmarks. These are the results in short:
```
Frames per second. Best 3 of 10 runs averaged. Higher is better.

C implementation           : 45.00 fps | 1070k C calls/s
C implementation + YJIT    : 45.24 fps | 1076k C calls/s
Ruby implementation        : 29.01 fps |  690k C calls/s
Ruby implementation + YJIT : 40.71 fps |  968k C calls/scc

Ruby+YJIT vs Best C : 90.0%
% Gain From YJIT    : 40.3%
```
Everything is faster. Ruby+YJIT as a percentage of C is about the same. YJIT is doing more vs. plain Ruby, maybe because it's constantly using the saved SDA state. I put this data pin optimization into bit-bang SPI from the start too, so it's an even playing field.

You'll notice I calculated the number of C calls (`lgGpioWrite` or `lgGpioRead`) per second.

I'll spare you the [explanation](https://github.com/denko-rb/lgpio/blob/f5df7486ae5cda145ed812b7fa32604fd7f12556/examples/spi_bitbang_ssd1306_sim_bench.rb#L30-L44), but with this pattern, I2C now makes 23 calls for 99% of bytes, and up to 28 for 1%. SPI does 20 and 24 respectively. 1% is negligible, so I2C is 23, SPI is 20.

## Why Do We Care?

[This example](https://github.com/joan2937/lg/blob/master/EXAMPLES/lgpio/bench.c) in the LGPIO C repo benchmarks how many times `lgGpioWrite` can be called per second, doing *nothing* else. It prints "toggles per second", so double the result to get calls. I modified it to call `lgGpioRead` instead, and results were basically the same.

My Pi 4 setup averages `1111k C calls per second`. This is the new high-water mark.

We can compare the different I2C implementations against this, to estimate how much time is spent on things that aren't GPIO calls:
```
GPIO throughput loss (overhead) vs. pure C toggling. Lower is better.

C implementation           :  3.7 %
C implementation + YJIT    :  3.2 %
Ruby implementation        : 37.9 %
Ruby implementation + YJIT : 12.9 %
```
YJIT looks even better now. The Ruby it *can* optimize runs at almost **3x** speed!

##  Back to SPI

I wrote this [benchmark](https://github.com/denko-rb/lgpio/blob/master/examples/spi_bitbang_ssd1306_sim_bench.rb), where the SPI interface sends out a pattern of bytes almost exactly like it's driving an I2C OLED, except for `1179` pixel bytes instead of `1024`. Ignoring the 1% of bytes with slight variance, `1179` makes it the same number of C calls per "frame" as the real OLED.

**Note:** This won't drive the SPI version of the SSD1306. It's just a simulation for comparison.

Before the results, also note that SPI has to do more than I2C:
  - Every transfer first checks the mode to set the right idle clock level
  - Every byte can branch to be either MSBFIRST or LSBFIRST
  - Every bit uses a `case` for SPI mode, so GPIO levels change in the right order
  - Every bit has two more conditionals, since reading and writing can both be optional

All of this happens in Ruby, so YJIT should help *more* than it does with I2C.

##  Results

Higher is better for fps & calls/s. Lower is better for overhead.

```bash
ruby examples/spi_bitbang_ssd1306_sim_bench.rb
# Results: 26.86, 26.39, 26.22 | Average: 26.49 fps |  630k C calls/s | 43.3% overhead
```

```bash
ruby --yjit examples/spi_bitbang_ssd1306_sim_bench.rb
# Results: 42.87, 42.80, 42.07 | Average: 42.58 fps | 1013k C calls/s |  8.8% overhead 
```

##  Conclusion

- YJIT improves overall performance by **60.7 %** over plain Ruby
  - Compare to a **40.3%** gain for I2C
- Without YJIT, **43.3%** of time is spent in Ruby, instead of writing to GPIOs
- YJIT speeds up the Ruby parts by a factor of **4.92x**!
- Ruby+YJIT SPI reaches **91.2%** of max theoretical GPIO throughput
  - Compare to **87.1%** for I2C
  - Perhaps YJIT handles the SPI control flow or keyword args better?
- Ruby+YJIT's throughput is **94.2%** of a hypothetical C SPI with equal overhead to I2C 
  - Real C overhead may be a bit more, as mentioned above, favoring YJIT
  
I don't feel the need to write it for real and find out. Maintaining Ruby instead of C is worth the small performance hit, especially when these are likely to be used at far lower data rates than tested.
