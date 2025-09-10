+++
date = '2025-09-09T23:48:48-05:00'
draft = false
title = 'Building a CPU Graph for Waybar'
+++

Recently I decided to dive down the Hyprland rabbit hole. Rather than using an
existing Hyprland config, I decided to build up my own environment from
scratch. For simplicity, I opted to use Waybar to build a minimal-but-useful
top bar. While Waybar has loads of great modules to choose from, I could not
find a CPU graph module. So, I hacked one together myself, and it turned out
pretty cool.

Waybar has great built-in modules for system monitoring, but they mostly (all?)
show only the current state. I wanted something that would show a graph of the
CPU usage over the last 60s. I've always admired how 'docker' uses braille
characters to create a progress animation in a little 2x4 dot matrix. I decided
to extend that clever trick and use a string of braille characters to draw a
longer dot matrix; one large enough to render a small graph.

This is what I came up with:

{{< img src="screenshot-short.png" alt="Screenshot2" caption="CPU history graph module" >}}

And here it is in the Waybar:

{{< img src="screenshot-long.png" alt="Screenshot1" caption="CPU graph integrated into Waybar" >}}

## The Implementation

You can find the full implementation in my
[dotfiles](https://github.com/phreaknik/dot-config/blob/cec5f05a94bf0bd2274ea52b2ece8bd2f2aa128d/waybar/scripts/cpu_history.py).

### The graph script

The implementation is fairly straight forward. I wrote a python script that
runs on a 1 second tick from Waybar. On every run, the script:

1) Reads current CPU usage
2) Maps usage to a 0-4 dot level
3) Saves the reading to a history file
4) Reads the history and maps the entries to braille characters

Since each braille character is a 2x4 dot pattern, each character must
represent two entries in the history file (or 2 seconds of history). To keep
things simple, I scan the history in groups of two, and look up the braille
character from a dictionary of two-value tuples:

```python
BRAILLE_PATTERNS = {
    (0, 0): '⠀', (1, 0): '⡀', (2, 0): '⡄', (3, 0): '⡆', (4, 0): '⡇',
    (0, 1): '⢀', (1, 1): '⣀', (2, 1): '⣄', (3, 1): '⣆', (4, 1): '⣇',
    (0, 2): '⢠', (1, 2): '⣠', (2, 2): '⣤', (3, 2): '⣦', (4, 2): '⣧',
    (0, 3): '⢰', (1, 3): '⣰', (2, 3): '⣴', (3, 3): '⣶', (4, 3): '⣷',
    (0, 4): '⢸', (1, 4): '⣸', (2, 4): '⣼', (3, 4): '⣾', (4, 4): '⣿',
}
```

One point to mention, I didn't do a simple uniform quantization from CPU usage
to the dot level. In practice the difference between 0.1% load and 5% is
significant, and I want the graph to show that. A uniform quantization would
give a mapping of `(0-20%, 20%-40%, 40%-60%, 60%-80%, 80%-100%)`. Any CPU value
between 0%-20% wouldn't even show on the graph... that's useless to me. So I
decided to choose different quantization levels to emphasize the usage ranges I
care about:

- 0-1%: No dots (idle)
- 1-12.5%: 1 dot (light usage) 
- 12.5-25%: 2 dots (low usage)
- 25-50%: 3 dots (moderate usage)
- 50%+: 4 dots (high usage)

### Waybar Integration

I then created a [custom Waybar
module](https://github.com/phreaknik/dot-config/blob/cec5f05a94bf0bd2274ea52b2ece8bd2f2aa128d/waybar/modules.json#L42)
to render the graph in my top bar. I
added a toggle mode to hide the graph, and a tool-tip to show the instantaneous
per-core CPU usage:

```json
  "custom/cpuhistory": {
    "exec": "~/.config/waybar/scripts/cpu_history.py -d 60",
    "format": "  {}",
    "interval": 1,
    "on-click": "~/.config/waybar/scripts/cpu_history.py toggle",
    "return-type": "json"
  },
```

## Closing Thoughts

This was a quick and dirty solution. There's a few things I might've done
differently, if this was more than just a quick script for some cosmetic eye
candy; e.g. making parameters easier to tweak, or re-thinking the dot-levels,
etc. That said, I'm more excited by how little effort it took to build
something genuinely useful, and I don't want to ruin that balance of utility vs
effort 😉
