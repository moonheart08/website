+++
date = 2024-06-02T15:14:13.846Z
title = 'The Webdev Experience'
summary = "A sometimes angry pile of words about the experience of making and maintaining this website. Sometimes updated!"
tags = ["personal", "web"]
+++

## Chrome sub-pixel rendering.
Chrome attempts to render elements with subpixel accuracy. Good, right?
Well.. Here's how this site's buttons look in firefox:

![An image of properly rendered buttons in firefox.](/imgs/firefox-buttons-the-webdev-experience.png)

And here's how Chrome presents the same group of elements as of writing:

![An image of incorrectly rendered buttons in Chrome.](/imgs/chrome-buttons-the-webdev-experience.png)

Can you see it?

The edges of the buttons have a *halo effect* in chrome due to badly applied subpixel rendering. It's even more prominent in the titlebar buttons. WHY?! This is apparently machine specific too, so the buttons will simply never look correct for some viewers. Pain.

## The lack of background-aware text.
This theme you're looking at is a port of a {{% a_rtui %}} stylesheet, and among the other fancy things I've managed to do within RTUI's stylesheet engine, I implemented 'background aware' text elements.

Instead of having to set text color per usecase by myself if the background is too light or too dark, the text element can check the primary color of its background element and compute the contrast, automatically swapping to light or dark based on that.

Needless to say CSS can't do this and while this site *emulates* the effect for nested boxes, it can't actually do it, I had to manually specify it for different backing elements.

<div class="obox">
    This is an example for background aware text, or it would be if CSS could do it.
    <div class="obox">
        And here we see the text color swap, automatically-ish (it's set to do so for <code>.obox .obox .obox</code>)
        <div class="obox">
            Due to how I implemented the site's palette, this uses one set of style specs for both light and dark mode.
            <div class="obox">
                Deeper nesting of <code>.obox</code> isn't supported for other reasons (while it is in the RTUI sheet, the color palette just wraps around, but CSS can't be that fancy and I have to specify it manually for the different nesting levels.)
            </div>
        </div>
    </div>
</div>

## Palette-oriented UI.
This entire site gets its coloration from a central palette specified on the body. Everything is figured out dynamically from there. I just think this is neat, and it makes tweaking the coloration globally much more reliable.

![Two five color palettes, specified on the body element.](/imgs/palette-the-webdev-experience.png)

As a bonus, this allows me to use a very simple trick to create the light theme: The order of colors in the secondary palette gets reversed for light mode usage.

I'll be real, I have no idea if this is a standard approach or anything, I'm not actually a UI designer and I just picked up some ideas from throwing things at the wall.