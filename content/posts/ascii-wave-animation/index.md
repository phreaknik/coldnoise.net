+++
date = '2025-09-25T10:00:00-05:00'
draft = false
title = 'Building an Interactive ASCII Wave Animation'
categories = ['projects']
tags = ['art', 'hacks', 'javascript', 'software']
+++

I recently wanted to add some artistic flair to my homepage — something that
would be both eye-catching and technically interesting without being
distracting. I settled on creating an animated ASCII wave effect that responds
to mouse movements. I was trying to achieve a 'rippling water' effect, but
ended up with some impressionist take on video noise... and I love it:

{{< art-wave height="400">}}

## Simple ASCII Animations

For this animation, I decided to animate a grid of ASCII characters. To do
that, I first need a bit of javascript to map the window into a grid of
characters. I can then scan over each character to apply an animation.

The following code updates the grid size. I call this function on first load,
and every time the window changes size:
```javascript
function updateDimensions() {
    const rect = waveContainer.getBoundingClientRect();
    const charWidth = fontSize * 0.4;
    const lineHeight = fontSize * 1.2;
    width = Math.floor(rect.width / charWidth);
    height = Math.floor(rect.height / lineHeight);
}
```
I then scan the x & y dimensions building a grid of characters. Later, I will
use some math to choose each character, and generate the cool animation.
```javascript
function generateWave() {
    let output = '';
    for (let y = 0; y < height; y++) {
        let line = '';
        for (let x = 0; x < width; x++) {
            // Some logic here to select the next character
            // let char = ...

            // Add character to point on scan line
            line += char
        }

        // Add the line to the output
        output += `<span class="wave-line">${line}</span>\n`;
    }

    // Draw the new output to the container
    waveContainer.innerHTML = output;

    // Advance time for the next frame
    time += waveSpeed;
}
```

## Combining Waveforms

I wanted to use ASCII characters that flickered to create some artistic
pattern. I thought that I might be able to generate some overlapping sine waves
and map their outputs to select characters for different 'intensity levels'.
The sine waves would hopefully give a rippling water effect.

I decided to juxtapose 3 waves moving different directions at different
frequencies:
```javascript
const wave1 = Math.sin((x * 0.1) + time) * waveHeight;
const wave2 = Math.sin((x * 0.05 + y * 0.1) - time * 0.7) * (waveHeight * 0.6);
const wave3 = Math.cos((x * 0.08 - y * 0.05) + time * 1.2) * (waveHeight * 0.4);
```

{{< art-wave-demo1 height="100" caption="Wave 1: Simple horizontal sine wave" >}}

{{< art-wave-demo2 height="100" caption="Wave 2: Diagonal wave with different frequency" >}}

{{< art-wave-demo3 height="100" caption="Wave 3: Counter-diagonal cosine wave" >}}

I'd hoped combining these waves would yield a nice rippling water effect. What
I actually got was something like a lava-lamp:
{{< art-wave-demo4 height="400" caption="Combined waves create a lava-lamp effect" >}}

But hey, it's interesting. I decided to see where this was going. The simple
'.' characters were boring, so I decided to substitute the characters based on
wave intensity. The peaks of the waves would get one of a series of heavy bars,
and the less intense fringes around the wave would get less significant
characters. Finally, I decided to throw some random sparkles in the mix.

The character palette includes:
- Wave characters: `~`, `≈`, `∼`, `～`, `〜`, `⁓`, `˜`
- Peak characters: `▓`, `▒`, `░`, `█`, `▄`, `▀`
- Sparkle characters: `✦`, `✧`, `⋆`, `✵`, `✶`, `✷`, `✸`
- Periphery: `.` and `·` for subtle texture

The combined result looked something like this:
{{< art-wave-demo5 height="400" caption="Final result with character substitution and sparkles" >}}

## Interactivity

For fun, I decided to add some mouse interactivity. The following snippet of
code measures the distance of the mouse from the current pixel, and applies
some distortion proportional to that distance. The effect is that wave
particles near the mouse are pushed and shoved in an entertaining way.

```javascript
const mouseDist = Math.sqrt((x / width - mouseX) ** 2 + (y / height - mouseY) ** 2);
const mouseEffect = Math.sin(mouseDist * 10 - time * 2) * (1 - mouseDist) * 3;
```

## Performance & Responsive Design

The animation runs at 60 FPS using `requestAnimationFrame` to sync with the
browser's repaint cycle, ensuring smooth performance. Each frame is built as a
single string before updating the DOM, minimizing reflow costs.

For responsive design, I made the animation adapt to different screen sizes
with dynamic font sizing:

```css
@media (max-width: 768px) {
    /* Each instance gets a unique ID for scoping */
    #wave-container-[timestamp] {
        font-size: 10px;
        letter-spacing: 0.05em;
    }
}
```

On smaller screens, the font size decreases while maintaining the wave effect,
ensuring the animation remains performant and visually appealing across all
devices.

## The Result

The end result was not at all what I expected -- more chaotic and hypnotic than
I was aiming for. It's fun for now, so I'm leaving it up on the landing page.
You can see the full code [on
github](https://github.com/phreaknik/coldnoise.net/blob/master/layouts/shortcodes/art-wave.html).
