<img src="icon.png" align="right" height="84" />

# Seattle Traffic Cameras [![Svelte](https://img.shields.io/badge/svelte-%23f1413d.svg?style=for-the-badge&logo=svelte&logoColor=white)](https://svelte.dev/) [![License](https://img.shields.io/github/license/the-sink/seattle-traffic-cams)](https://github.com/the-sink/seattle-traffic-cams/blob/main/LICENSE) [![Issues](https://img.shields.io/github/issues/the-sink/seattle-traffic-cams)](https://github.com/the-sink/seattle-traffic-cams/issues)

A static Seattle/King County-area traffic camera viewer hosted on GitHub pages.

You can view the live site **[here](https://the-sink.github.io/seattle-traffic-cams/public/)**!

[![Screenshot](https://i.imgur.com/KS8pxPn.jpeg)](https://the-sink.github.io/seattle-traffic-cams/public/)

Ever wanted to view every single active Downtown Seattle traffic camera at once...? Probably not, but here it is anyways:

[![Screenshot](https://i.imgur.com/4IXfzS0.jpeg)](https://the-sink.github.io/seattle-traffic-cams/public/?videoWidth=10&map=false#CMR-0017,CMR-0332,CMR-0096,CMR-0098,CMR-0184,CMR-0267,CMR-0315,CMR-0097,CMR-0317,CMR-0056,CMR-0241,CMR-0171,CMR-0182,CMR-0106,CMR-0169,CMR-0040,CMR-0170,CMR-0173,CMR-0185,CMR-0168,CMR-0167,CMR-0186,CMR-0059,CMR-0261,CMR-0264,CMR-0174,CMR-0046,CMR-0240,CMR-0239,CMR-0039,CMR-0055,CMR-0058,CMR-0176,CMR-0030,CMR-0016,CMR-0257,CMR-0309,CMR-0255,CMR-0303,CMR-0178,CMR-0304,CMR-0188,CMR-0035,CMR-0069,CMR-0217,CMR-0165,CMR-0191,CMR-0291,CMR-0181,CMR-0305,CMR-0179,CMR-0153,CMR-0256,CMR-0180,CMR-0194,CMR-0318,CMR-0049,CMR-0156,CMR-0033,CMR-0218,CMR-0265,CMR-0310,CMR-0258,CMR-0311,CMR-0320,CMR-0189,CMR-0043)

Click the above screenshot to open all downtown SDOT streams West of I-5. **Only click if you have a device that can handle 60+ simultaneous video streams...** it's a lot.

### To-Do List

- [x]  Ability to save currently open cameras to the URL hash to be re-loaded later
- [ ]  Camera name and close button on individual cameras in list
- [ ]  A built-in method of recording video streams and saving them as .mp4 or .webm files (requires above item first)
- [x]  "Close all" button
- [ ]  Move certain parameters to a config file (such as marker colors)