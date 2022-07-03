<img src="icon.png" align="right" height="84" />

# Seattle Traffic Cameras [![Svelte](https://img.shields.io/badge/svelte-%23f1413d.svg?style=for-the-badge&logo=svelte&logoColor=white)](https://svelte.dev/) [![License](https://img.shields.io/github/license/the-sink/seattle-traffic-cams)](https://github.com/the-sink/seattle-traffic-cams/blob/main/LICENSE) [![Issues](https://img.shields.io/github/issues/the-sink/seattle-traffic-cams)](https://github.com/the-sink/seattle-traffic-cams/issues)

A static Seattle/King County-area traffic camera viewer hosted on GitHub pages.

You can view the live site **[here](https://the-sink.github.io/seattle-traffic-cams/public/)** or by clicking the screenshot below!

[![Screenshot](https://i.imgur.com/KS8pxPn.jpeg)](https://the-sink.github.io/seattle-traffic-cams/public/)

## Guide

When the site is first opened, a map will be visible on the bottom 2/3 of the screen. Markers will load in on the map (slowly or quickly depending on your internet connection); these are all traffic cameras. Many of the highway cameras are operated by WSDOT and are only still images that update every minute or so, whereas the rest (in the Seattle area) are operated by SDOT and are live video streams.

The loading process that occurs when you open the site is verifying that the SDOT cameras are working. There are usually quite a few broken SDOT cams at any given time, and that process will remove them from the map. It can be skipped if you're on a poor internet connection or it's being slow for some other reason (by pressing the blue button below the location search bar), but this will cause broken streams to be visible and may become tedious. The results of the camera testing will be cached and future loads should be faster (unless the cache entries are overwritten, which is possible if you're using your browser heavily for a long period of time).

### Cameras

You can open and close cameras by clicking on their associated markers on the map. They will appear below the map, which can be hidden by clicking on the map button at the top right. Alternatively, hovering over camera feeds (or holding down on them on mobile) will show a close button that can be used instead. If you'd like to close *all* open cameras at once, press the red Close All button at the top right.

The green play button at the top right will play and fast-forward all video streams to roughly the current time (it will always be off by about 10-20 seconds, as the SDOT streams send new data in chunks every 10s).

The size of all cameras can be adjusted with the video width slider at the top right. This is adjusted by percentage of total screen width, so 25% means four cameras will fit on each row.

### Searching locations

A location search bar is available at the top left of the map. It uses Nominatim to perform a geolocation search and displays the result location on the map with a red marker. This marker can be clicked to dismiss. You do not need to put "Seattle, WA" or something similar at the end of your search, as "seattle" is appended to the end of each search query automatically. Results are not flawless and don't work at all sometimes.

### Persistence

Currently open cameras, video width, and whether the map is open or not are all properties that are saved to the URL. This means reloading the page (or copying the current URL in your browser and sharing it) should bring up essentially the exact same view.

For example, [here](https://the-sink.github.io/seattle-traffic-cams/public/?videoWidth=24&map=false#CMR-0279,CMR-0213,CMR-0280,CMR-0124,CMR-0281,CMR-0282,CMR-0283,CMR-0284,CMR-0138,CMR-0285,CMR-0286,CMR-0287) is Rainier Ave S from I-90 to S Othello Street with the map closed.

Or, for something a bit more crazy, click the screenshot below for every single (working) Downtown Seattle traffic camera west of I-5. This will load over 60 simultaneous video streams, so make sure your device and internet can handle that!
[![Screenshot](https://i.imgur.com/4IXfzS0.jpeg)](https://the-sink.github.io/seattle-traffic-cams/public/?videoWidth=10&map=false#CMR-0017,CMR-0332,CMR-0096,CMR-0098,CMR-0184,CMR-0267,CMR-0315,CMR-0097,CMR-0317,CMR-0056,CMR-0241,CMR-0171,CMR-0182,CMR-0106,CMR-0169,CMR-0040,CMR-0170,CMR-0173,CMR-0185,CMR-0168,CMR-0167,CMR-0186,CMR-0059,CMR-0261,CMR-0264,CMR-0174,CMR-0046,CMR-0240,CMR-0239,CMR-0039,CMR-0055,CMR-0058,CMR-0176,CMR-0030,CMR-0016,CMR-0257,CMR-0309,CMR-0255,CMR-0303,CMR-0178,CMR-0304,CMR-0188,CMR-0035,CMR-0069,CMR-0217,CMR-0165,CMR-0191,CMR-0291,CMR-0181,CMR-0305,CMR-0179,CMR-0153,CMR-0256,CMR-0180,CMR-0194,CMR-0318,CMR-0049,CMR-0156,CMR-0033,CMR-0218,CMR-0265,CMR-0310,CMR-0258,CMR-0311,CMR-0320,CMR-0189,CMR-0043)

## To-Do List

- [x]  Ability to save currently open cameras to the URL hash to be re-loaded later
- [x]  Camera name and close button on individual cameras in list (hover over stream)
- [ ]  A built-in method of recording video streams and saving them as .mp4 or .webm files (requires above item first)
- [x]  "Close all" button
- [ ]  Move certain parameters to a config file (such as marker colors)
- [ ]  Investigate possibility of a low framerate mode for when many cameras are pulled up at once
- [ ]  Add drag function to camera streams so the layout can be rearranged