---
layout: post
title: Rainbow Light Cycle for the UniFi LED Bar
date: 2024-01-24 00:01:00
description: A small shell script that fades a UniFi LED bar through a continuous rainbow cycle.
tags: unifi led shell IT
categories: IT
---

UniFi devices that expose `/proc/ubnt_ledbar` let you set the LED bar color by writing an `R,G,B` value to `/proc/ubnt_ledbar/custom_color`. The short `/bin/sh` script below uses this to fade the LED through red, orange, yellow, green, blue and purple, then back to red, in an endless loop. A compiled Go version also exists, but this post covers only the shell script.

## The Script

```sh
#!/bin/sh

# This function takes RGB values and writes them to the LED device
set_led_color() {
  echo -n "$1,$2,$3" > /proc/ubnt_ledbar/custom_color
}

# This function smoothly transitions from one color to another
transition_color() {
  local startR=$1
  local startG=$2
  local startB=$3
  local endR=$4
  local endG=$5
  local endB=$6
  local steps=20  # Use fewer steps since we're using a longer sleep duration
  local i=0

  while [ $i -le $steps ]; do
    local r=$(awk "BEGIN {print int($startR + ($endR - $startR) * $i / $steps)}")
    local g=$(awk "BEGIN {print int($startG + ($endG - $startG) * $i / $steps)}")
    local b=$(awk "BEGIN {print int($startB + ($endB - $startB) * $i / $steps)}")
    set_led_color $r $g $b
    sleep 1  # Use full second sleep
    i=$((i + 1))
  done
}

# The main loop to cycle through colors
while true; do
  # Transition from red to orange
  transition_color 254 0 0 254 165 0
  # Transition from orange to yellow
  transition_color 254 165 0 254 254 0
  # Transition from yellow to green
  transition_color 254 254 0 0 254 0
  # Transition from green to blue
  transition_color 0 254 0 0 0 254
  # Transition from blue to purple
  transition_color 0 0 254 160 32 240
  # Transition from purple back to red
  transition_color 160 32 240 254 0 0
done
```

## How It Works

**`set_led_color R G B`** joins its three arguments into an `R,G,B` string and writes it to `/proc/ubnt_ledbar/custom_color`. The `-n` flag keeps `echo` from adding a trailing newline.

**`transition_color startR startG startB endR endG endB`** fades linearly between two colors:

- With `steps=20` and the condition `$i -le $steps`, `i` runs from 0 to 20 inclusive. Each transition therefore makes 21 writes, beginning on the start color and ending exactly on the end color.
- Each channel is computed in `awk` as `start + (end - start) * i / steps`, and `int()` truncates the result to an integer.
- `sleep 1` pauses one second between writes, so one transition takes about 21 seconds.

**The main loop** repeats six transitions forever. Each one ends on the color the next one starts from:

| Key color | R   | G   | B   |
| --------- | --- | --- | --- |
| Red       | 254 | 0   | 0   |
| Orange    | 254 | 165 | 0   |
| Yellow    | 254 | 254 | 0   |
| Green     | 0   | 254 | 0   |
| Blue      | 0   | 0   | 254 |
| Purple    | 160 | 32  | 240 |

A transition's last write and the next transition's first write set the same color, so each key color holds for about two seconds before the next fade starts. A full cycle takes a little over two minutes.

`local` and `echo -n` aren't strictly POSIX, but both work in common `/bin/sh` implementations such as BusyBox ash and dash.

## Usage

Copy the script to the device, make it executable, and start it in the background with `nohup` so it keeps running after you log out:

```sh
scp light-cycle.sh <user>@<device-ip>:/tmp/
ssh <user>@<device-ip>
chmod +x /tmp/light-cycle.sh
nohup /tmp/light-cycle.sh > /dev/null 2>&1 &
```

Run it as a user that can write to `/proc/ubnt_ledbar/custom_color`. To stop it, find its PID with `ps` and `kill` it.
