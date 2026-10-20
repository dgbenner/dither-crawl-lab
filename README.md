# Dither Crawl Lab

Halftone "crawling" shadows on live UI, in a handheld-LCD style.

The form controls are real, invisible HTML elements that handle clicks, typing, focus and screen readers. A low-resolution canvas on top reads their position and state every frame and redraws them in three colors. Each shadow is a soft shape thresholded against noise; re-rolling part of the noise on every update makes the edge pixels flip while the core holds still.

Open `index.html` in a browser. No build step.
