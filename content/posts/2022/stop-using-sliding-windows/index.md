+++
date = '2022-07-11T10:00:00-05:00'
draft = false
title = 'Stop Using Sliding Windows When You Just Need a Smooth Signal'
+++

I review a lot of code, and I keep seeing the same mistake over and over.
Someone needs to smooth out a noisy signal (maybe sensor data, maybe user
metrics, whatever) and they immediately reach for a sliding window moving
average, a.k.a. the SMA filter.

Here's an example you've probably seen:
```c
#define WINDOW_SIZE 10

typedef struct {
    float buffer[WINDOW_SIZE];
    int index;
    int count;
    float sum;
} MovingAverage;

// Initialize the moving average structure
void average_init(MovingAverage *ma) {
    ma->index = 0;
    ma->count = 0;
    ma->sum = 0.0f;
    for (int i = 0; i < WINDOW_SIZE; i++) {
        ma->buffer[i] = 0.0f;
    }
}

// Add new value and return current average
float average(MovingAverage *ma, float new_value) {
    // Subtract the value being replaced from sum
    ma->sum -= ma->buffer[ma->index];
    
    // Add new value to buffer and sum
    ma->buffer[ma->index] = new_value;
    ma->sum += new_value;
    
    // Update circular index
    ma->index = (ma->index + 1) % WINDOW_SIZE;
    
    // Track how many values we've added (up to WINDOW_SIZE)
    if (ma->count < WINDOW_SIZE) {
        ma->count++;
    }
    
    // Return average
    return ma->sum / ma->count;
}

// Example usage
int main() {
    MovingAverage ma;
    average_init(&ma);
    
    printf("Adding values and calculating moving average:\n");
    printf("Value: %.1f, Average: %.2f\n", 10.0f, average(&ma, 10.0f));
    printf("Value: %.1f, Average: %.2f\n", 20.0f, average(&ma, 20.0f));
    printf("Value: %.1f, Average: %.2f\n", 30.0f, average(&ma, 30.0f));
    printf("Value: %.1f, Average: %.2f\n", 40.0f, average(&ma, 40.0f));
    printf("Value: %.1f, Average: %.2f\n", 50.0f, average(&ma, 50.0f));
    
    return 0;
}
```

There's a lot of unnecessary complexity here to maintain the circular buffer
(~35 lines of code), not to mention the amount of memory required grows
linearly with the window size (see [FIR
filters](https://en.wikipedia.org/wiki/Finite_impulse_response)) . Don't be
fooled into thinking this is a "C problem" either. Higher level languages with
better semantics for buffers can reduce the lines of code, but the compiler
still must implements all of this under the hood and cannot avoid
performance/memory/program-size costs.

Additionally, the SMA filter has a 'startup period'. It must fill the buffer,
before it can output meaningful results. This startup time could be significant
if you require a large window size. This startup period is especially painful
if you want to change your filter parameters at run-time -- e.g, if you want to
change the window size or if your ADC scale factor changes, you have to discard
the filter output until the buffer has been completely replaced by new values.

Lastly, there are some situations where an SMA will fail to smooth the signal.
If your input stream occasionally has very large samples (larger than the mean
× window size), then the SMA will output a rectangular impulse response -- e.g.
if one very large sample is input, the SMA will give a sudden large step up
when that sample enters the window and a large step down when it leaves the
window. I'll refer to this as the [black
noise](https://en.wikipedia.org/wiki/Colors_of_noise) problem, because black
noise would exacerbate this most, but this has nothing to do with signal vs.
noise.

SMA filters have their time and place, but I see them used all too often in
situations when an EWMA would perform better.

## Avoid the headaches, use an EWMA filter if you can

An exponentially weighted moving average (EWMA) filter is superior in
_almost_ every way:
* less memory: only stores 1 value, regardless of "window size"
* smaller program: no buffer management overhead
* no startup delay: EWMA filter output is useful after only 1 sample
* easily tuned in real-time
* not vulnerable to the black noise problem of SMA filters

Unless your application specifically needs to remember the values in the
sliding window, consider using an exponential moving average (EWMA) instead.
Being an [IIR filter](https://en.wikipedia.org/wiki/Infinite_impulse_response)
it is able to offer similar (often better) results, with a fraction of the
resources. The EWMA has just one state variable and just one simple
calculation. Take a look at this [example from
OpenWRT's](https://git.openwrt.org/?p=project/qosify.git;a=blob_plain;f=qosify-bpf.c;hb=refs/heads/master)
QoS implementation:
```c
static __always_inline __u32 ewma(__u32 *avg, __u32 val)
{
	if (*avg)
		*avg = (*avg * 3) / 4 + (val << EWMA_SHIFT) / 4;
	else
		*avg = val << EWMA_SHIFT;

	return *avg >> EWMA_SHIFT;
}
```
*Pulled from OpenWRT's qosify implementation:
https://git.openwrt.org/project/qosify.git*

This is an implementation of the prototypical EWMA, with `alpha = 0.75` (and
scaled by `EWMA_SHIFT`):
```
ewma[n] = α*sample + (1-α)*ewma[n-1]
```

Compared to the SMA filter before, the EWMA stores only one value and has zero
overhead for the circular buffer. Changing α is comparable to changing the
window size of an SMA filter. Higher α gives a higher bias to new values (e.g.
small window size), while small α gives more bias for historical values (e.g.
large window size).

Below is an example of some noisy sensor data, and two EWMA filters used to
"smooth out" the data:

{{< img src="ewma-demo.png" alt="EWMA Demo" caption="Two EWMA filters (red & yellow) applied to a noisy sensor data stream. The red EWMA has higher alpha than the yellow, and thus tracks the sensor output more closely." >}}

## Summing it all up

Take a look at your application's requirements and check whether you need a
literal average (in the mathematical sense) or if you need access to historical
values. In these cases you might want an SMA. For example, you need an SMA if:
- You're also computing the median (you need all the values)
- Regulatory requirements specify "average of last N samples"
- You're doing frequency domain analysis and specifically need an FIR filter
- You already have to store the samples for some other reason

Otherwise, consider if an EWMA would be a simpler and more performant solution.
