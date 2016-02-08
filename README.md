# PulseAudio telephone audio interface
This document provides a reference configuration for recording high-quality, broadcast-style telephone interviews using PulseAudio.

## Requirements
The configuration assumes the following:

- Talent (person on local end of call)
- Microphone
- Headphones
- Computer audio interface for mic and headphones (for example, analog in/outs on soundcard)
- Linux (tested on Ubuntu 14.04)
- PulseAudio (tested on 4.0)
- PulseAudio Volume Control
- Google Voice (running in Gmail in Google Chrome)
- Caller (person on other end of call)
- Audacity (tested on 2.0.5)

These are not hard requirements. Speakers can be used instead of headphones if care is taken to avoid a feedback loop. The same type of configuration will also work on other platforms supporting PulseAudio, and with other software like Google Hangouts, Skype, VoIP softphones, etc. Any two-channel recording software will do.

## Objective
The goal is to allow the talent to speak with the caller through the mic and headphones, while recording both the talent and the caller on separate audio channels. (Recording on separate audio channels is important for flexibility in editing later; for example, it allows removing noise from one side of the call without affecting the other side.) The talent should be recorded in full fidelity directly from the mic, while the caller is recorded from the phone. More specifically, the routing should be as follows:

- Talent speaking into mic and listening to headphones
- Mic routed to phone input
- Mic also routed to left channel of recording
- Phone output routed to headphones
- Phone output also routed to right channel of recording

## PulseAudio
The configuration makes use of PulseAudio's advanced virtual mixing capabilities.

### Background
The following concepts exist in PulseAudio:

- A *module* is a unit of functionality which can be loaded, unloaded, and configured.
- A *source* is an input device, which can be physical (like a mic input on a soundcard) or virtual (like an equalized version of another source).
- A *source output* is a stream of audio coming from a source and going somewhere. (Think of it like a virtual cable connection.) A source can have many outputs.
- A *sink* is an output device, which can be physical (like a headphone output on a soundcard) or virtual.
- A *sink input* is a stream of audio coming from somewhere and going to a sink. A sink can have many inputs.
- Every sink has an associated *monitor*, which is a source that plays the audio sent to the sink.
- A *null sink* is a special type of virtual sink which just discards the audio sent to it. (This would seem useless were it not for the associated monitor, which can be used to capture the audio sent to the sink.)
- A *loopback* sends audio from a source to a sink, with the necessary source output(s) and sink input(s). A source or sink can have many loopbacks.

PulseAudio can be configured from the command line with `pactl` (for individual commands) or `pacmd` (for interactive mode) using the PulseAudio CLI language. Commands can be put in a configuration file (for example, `~/.pulse/default.pa`) for persistence. The PulseAudio Volume Control GUI is also useful for making configuration changes.

### Configuration
To achieve the previously stated objective, the following are created:

- A virtual source `phonein` which clones the mic input but allows independent volume adjustment. (Strictly speaking this is optional, but useful.)
- A source output of `phonein` to the phone (Google Voice) input.
- A null sink `phonemix` which contains the stereo mix of talent and caller for recording.
- A loopback sending the left channel of the mic input to the left channel of `phonemix`.
- A loopback sending the right channel of the headphone output monitor (which contains the phone output) to the right channel of `phonemix`.
- A source output of the `phonemix` monitor to the recorder.

More specifically, the following commands are first run (or added to `~/.pulse/default.pa` for persistence, then PulseAudio restarted):

```
load-module module-virtual-source source_name=phonein master=<mic source name> use_volume_sharing=no
update-source-proplist phonein device.description="Phone Input"
load-module module-null-sink sink_name=phonemix
update-sink-proplist phonemix device.description="Phone Mix"
update-source-proplist phonemix.monitor device.description="Monitor of Phone Mix"
load-module module-loopback source=<mic source name> sink=phonemix channel_map=left
load-module module-loopback source=<headphone monitor name> sink=phonemix channel_map=right
```
(The required source names can be found with `pactl list sources`.)

After a phone call is initiated in Google Voice, and recording initiated in Audacity, the source outputs to Google Voice and Audacity are configured using the PulseAudio Volume Control GUI. PulseAudio persists these mappings.

![PulseAudio Volume Control](https://raw.githubusercontent.com/blargoner/pulseaudio-phone-config/master/pavucontrol.png)

Levels are adjusted to ensure that the talent and caller are audible to each other, and audible in the recording.

![Audacity](https://raw.githubusercontent.com/blargoner/pulseaudio-phone-config/master/audacity.png)
