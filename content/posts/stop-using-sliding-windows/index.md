+++
date = '2022-07-11T10:00:00-05:00'
draft = false
title = 'Stop Using Sliding Windows When You Just Need a Smooth Signal'
categories = ['rants', 'software']
tags = ['rant', 'math', 'software']
+++

I review a lot of code, and I keep seeing the same mistake over and over.
Someone needs to smooth out a noisy signal (maybe sensor data, maybe user
metrics, whatever) and they immediately reach for a sliding window moving
average.

Here's an example; something I've seen all too often:

```go
// The over-engineered way
type SlidingAverage struct {
    window     []float64
    windowSize int
}

func NewSlidingAverage(windowSize int) *SlidingAverage {
    return &SlidingAverage{
        window:     make([]float64, 0, windowSize),
        windowSize: windowSize,
    }
}

func (s *SlidingAverage) Update(value float64) float64 {
    s.window = append(s.window, value)
    if len(s.window) > s.windowSize {
        s.window = s.window[1:]
    }
    
    sum := 0.0
    for _, v := range s.window {
        sum += v
    }
    return sum / float64(len(s.window))
}
```

Look at the unnecessary complexity. They're maintaining a buffer, shifting
elements around, iterating through N values every single update. If you
remember back to your digital signal processing class, you might recognize this
as an FIR filter. FIR filters are not memory or CPU efficient. FIR filters have
their place, but not here. Not for generic "signal smoothing".

## Use an EMA instead

Take a look at this alternative approach, using an exponential moving average.
It has just one state variable and just one line of math:

```go
// The elegant way
type ExponentialAverage struct {
    alpha      float64
    value      float64
    initialized bool
}

func NewExponentialAverage(alpha float64) *ExponentialAverage {
    return &ExponentialAverage{
        alpha: alpha,
    }
}

func (e *ExponentialAverage) Update(newValue float64) float64 {
    if !e.initialized {
        e.value = newValue
        e.initialized = true
    } else {
        e.value = e.alpha*newValue + (1-e.alpha)*e.value
    }
    return e.value
}
```

That's **x\[n\] = α·y\[n\] + (1-α)·x\[n-1\]**. One multiplication, one
subtraction, one more multiplication, one addition. Done. All of this operating
on only one memory slot.

The sliding window uses O(N) memory to store all those samples. The EMA uses
O(1)—literally one float64.

The sliding window does O(N) operations per update, summing all those values.
The EMA does O(1)—those same four arithmetic operations every time, regardless
of your smoothing strength.

Want to change the smoothing characteristics? With a sliding window, you're
reallocating buffers and changing your whole data structure. With EMA, you
change one number. You can even adjust it on the fly based on conditions... try
doing that with your sliding window.

## "But I need the exact average!"

Do you? Really? More often then not, you probably don't. Take a look at your
application's requirements and check whether you need a literal average (in
the mathematical sense), or if the requirement really just needs a "smooth
history" of some signal/data/input/etc. In my experience, this requirement is
misjudged quite often.

Fine, there are legitimate uses for the sliding window:

- You're computing the median (you need all the values)
- Regulatory requirements specify "average of last N samples"
- You're doing frequency domain analysis with specific requirements
- You actually need to know which samples fall outside the window for some domain-specific reason

But if you can't point to a clear need for data relating to the individual
samples in your window, take a long and hard look, before you go
over-engineering things.

I'll stop the rant here: Next time you see someone smoothing a signal with a
sliding window / moving average filter, suggest an EMA instead. And maybe send
them this post!
