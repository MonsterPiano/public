# Monster Piano Public API

## Websockets

The Monster Piano API is used via an internal HTTP server and a standard WebSocket upgrade.  Currently the only supported version is RFC6455 / version 13.  There are no plans to support older versions of the protocol.  We plan to support new versions as needed.

## Connection and Security

There are two network interfaces to connect your Piano to other systems.  The Piano's built-in WIFI, and wired Ethernet.  Ethernet is only available inside the head of the center octave.  It is not brought out to a panel anywhere.  The Piano software does not require authentication, so only trusted devices should be allowed to connect to the Piano.

The default WIFI IP address for the Piano is 192.168.172.30.  The default wired Ethernet IP address is 192.168.0.30.  The default port is standard http port 80.

## Basic API Use

The API is mixed-format.  Both Binary and Text messages are supported.  Currently most messages are only supported in one format.  That is, you can't send a binary Set-Property message and you can't send a text note-on message.  This will be expanded in time.  The documentation below will specify what is currently implemented and what has yet to be done.  If you need a particular message completed, let us know.

The first payload byte of the WebSocket message is the command byte.  These commands are the same for a binary message or for a text message.  As such, they are all printable characters.  Bytes following that byte are dependent on the command.

The API is currently case sensitive for most everything.

The basics are listed first.  Details will follow in the sections below.

Commands are:

* U - Hardware key up event
* D - Hardware key down event
* N - Note On event  - equivalent to the MIDI event
* F - Note Off event - equivalent to the MIDI event
* L - LED change command / event

* S - Set Property
* P - Property Change Notification
* C - Call RPC function
* R - RPC Response (text/binary based on opcode)
* X - Subscribe to events.  Types listed below.

# Event Subscription

There are two ways to subscribe to events.  The first is at connection time.  The URL in the initial HTTP GET request should be the ascii decimal number associated with the event bits ORed together.  The second is with the X subscribe command. Subscribing to non-implemented events listed below will have no effect, but will not cause any problems, either.

* 0x01 - Hardware keypress events - both key up and key down - "U" and "D" commands
* 0x02 - MIDI keypress events - for all MIDI inputs, USB and hardware MIDI. - NOT IMPLEMENTED -
* 0x04 - MIDI note events - equivalent to MIDI output - "N" and "F" commands
* 0x08 - LED events
* 0x10 - Hardware add/remove events - When octaves are added or removed, or Piano is otherwise resized - NOT 
IMPLEMENTED -
* 0x20 - Property Change Notification - receive notification of all property changes

## X Subscribe command

To subscribe or change subscriptions simply send "X" followed by the ascii decimal number associated with the event bits ORed together.  This is the same number as the URL subscription number.  This completely overwrites any previous subscription, so you must include all types in this message, not just additional bits.

# Properties

The Monster Piano software is devided into modules.  Each module provides it's own services to the rest of the Piano software, and each has it's own properties.

"S" and "P" commands follow the same format:

SModule.Property=Value

Optionally, multiple properties may be set in the same message:

SModule.Property=Value\n
Module2.Property2=Value2\n
Module3.Property3=Value3

Notification of property changes have the same format, only with a "P" at the beginning of the change.  Property changes may be sent multiple times depending on the internal implementation.  Some property changes may also change other properties, in which case you will receive notifications of all property changes.  Property changes may occur as a result of internal events and may not be initiated by the user.  Notifications will occur in any event.

There is no reasonable limit to the number of properties you may set in one message.  You may set a single property multiple times in the same message.  Properties will be set in the order they appear in the message.

# Property List

Conventions below include RO - Read-only; RW - Read-Write; WO - Write-only.

Many modules have an info property.  This is intended to give a human-readable description of the module's current settings.  It is always read-only, always a string, and in many cases is unimplemented.  It is listed below where it is fully or partially implemented.  It's future purpose is to simplify saving default property settings for power-on.

* Piano.state    integer RW - 0=normal play, 1=custom control
  * in custom control mode the piano will not respond to keypresses or midi events by itself, it must be controlled by an application.  Demo songs and other sources will still work.
* Piano.volume   integer RW - software MIDI volume 0-127
* Piano.voice    integer RW - selected MIDI voice 0-127
* Piano.velocity integer RW - MIDI velocity for hardware keypresses (MIDI velocity used for MIDI commands)
* Piano.minkey   integer RW - minimum hardware key - MIDI note number
* Piano.maxkey   integer RW - maximum hardware key - MIDI note number
* Piano.minled   integer RW - minimum note for LED commands.  Allows for virtual piano sizes, i.e. 5-octave on screen piano or other lighting, while only having a 2 or 3 octave piano.
* Piano.maxled   integer RW - maximum note for LED commands.
* Piano.elapsed  int64 RO - not present in default configs, but may be read individually.  Milliseconds since startup
* Piano.model    string RO - Monster Piano model name
* Piano.serialno string RO - Monster Piano serial number
* Piano.version  string RO - Monster Piano version string
- Eq module not tested yet, but should work
* Eq.LFmin integer - RO - Minimum frequency for LF band
* Eq.LFmax integer - RO - Maximum frequency for LF band
* Eq.LFfreq integer - RW - sets frequency in LFmin..LFmax range, 0 - 127
* Eq.LFgain real - RW - current hardware has a range of -12  - +6
* Eq.HFmin integer - RO - Minimum frequency for HF band
* Eq.HFmax integer - RO - Maximum frequency for HF band
* Eq.HFfreq integer - RW - sets frequency in LFmin..LFmax range, 0 - 127
* Eq.HFgain real - RW - current hardware has a range of -12db - +12db
* Eq.state - RW - 0 disables sending EQ MIDI commands, 1 Enables it. If the Eq is adjusted in state 1 and then disabled, the effective Eq will be the same as it was before changing state=0.  If you want to shut off any EQ you must set LFgain and HFgain to 0 first.
* Harmony.info - see above
* Harmony.state integer - bits for the harmony mode
  *    0 - disabled
  *    1 - Major chord
  *    2 - Minor chord
  *    3 - 1 octave lower
  *    4 - 1 octave higher
  * 0x10 - Major key
  * 0x20 - Minor key
    * when major and minor keys are active, the lower 4 bits determine the key the piano is in -- 0=C, 1=C#, etc.
* Lighting.color string - RW - hex value of single color mode - RRGGBB, same as CSS color, only without "#" and requires all digits.  3 digit CSS shortcut doesn't work.
* Lighting.state - RW 
  * 0 = disabled, or custom lighting mode.  You may send "L" commands to set the lights as you see fit.
  * lower 12 bits determine selected palette (&0xfff) 0x001 is always single color mode - notes color is determined by "Lighting.color"  0x002 is the first multi-color palette
  * 0x1000 - Inverted trigger - LEDs normally on unless triggered off
  * 0x2000 - Always on or off, based on inverted flag
  * 0x4000 - Trigger on Notes played, including demos, midi events, etc
  * 0x8000 - Trigger on hardware keypresses only
  * default mode is 0x4002 - Trigger on all notes played, palette 1
* Lighting.palette integer - RO - currently selected multi-color palette (even if single-color mode is enabled or lighting is disabled altogether)
* Lighting.paletteList StringList - RO - list of palette names.  First entry in the list = state 0x002.
* Lighting.info - see above
* LightningRound.state integer RW - 0=disabled, 1=enabled; other temporary states for game over, player1/2 winning, etc
* LightningRound.speed integer RW - time between key moves, in ms
* LightningRound.min   integer RW - =Piano.minkey
* LightningRound.max   integer RW - =Piano.maxkey
* LightningRound.current - NOT IMPLEMENTED -
* Metronome.state   integer RW - 0=disabled, 1=enabled
* Metronome.timesig integer RW - Time signature 2/2 2/4 3/2 3/4 4/4 6/8 8/8 12/8.  Handy numerical definitions - 2/2 = 0x22, 3/4 = 0x34.  The only tough one... 12/8 = 0xc8. 
* Metronome.tempo   integer RW - tempo BPM
* Octaves - Octaves module.  Several properties, however not currently documenting it here.  If you need it, let me know.
* Pedals.state integer RW - Several piano pedals, including the common three
  * Soft       0x01
  * Sostenuto  0x02
  * Damper     0x04
  * Portamento 0x08
  * Legato     0x10
* Sequence.demolist  stringlist RO - List of demo song names
* Sequence.tracklist stringlist RO - List of playalong track names
* Sequence.userlist  stringlist RO - List of user tracks (songs on vfat partition "USER")
* Sequence.looplist  stringlist RO - List of rhythm names
* Sequence.playing string RO - Playing file name
* Sequence.uri string RW - simple file URI containing category from above, and position in name array, ex. demo_1, track_2, user_5, loop_16
* Sequence.mode integer RW - 0=normal, 1=demo mode - auto play demo song when current ends
* Sequence.state integer RW - 0=stopped 1=playing 4=paused
* Sleep.timeout integer RW - Start sleep after x seconds, 0 disables sleep entirely
* Sleep.interval integer RW - Update LEDs every x milliseconds
* Sleep.state - 0=not currently sleeping, 1=sleeping
* Sleep.info - see above
* Speech.notespeak integer RW - convenience, sets state (&0xf0)
  * 0x00 - disabled, only speak for interactive events
  * 0x20 - Solfege, "do, re, me, fa, so, la, ti, do"
  * 0x40 - NoteSpeak, "C, C Sharp, etc"
  * 0x80 - Operatic, sing notes with human voice
* Speech.personality integer RW - convenience, sets state (&0x0f).  personality selected is lower 4 bits
* Speech.state integer RW - Speech.notespeak | Speech.personality

# RPC Command

RPC commands are formed by sending a text (opcode=1) message with the following format

CModule.Function
<body text>

C is the Call command.  An optional newline (\n) may be included after the function name, followed by text to send to the function.  An RPC call may or may not respond with an R command.  The response will have a similar format:

RFunction
<body text or binary data>

A binary response will have the same format, but will be with opcode=2.  The newline is still in place.

The Piano may send an RPC response without a call in certain circumstances, for instance, when first connecting and subscribing to property changes, the system sends the current state of all properties as a json object.  No call is required.  In that case the message appears as below:

RnewState
{JSON OBJECT}

Only one RPC call may appear in a single WebSocket message

# RPC Command list

There are only a few RPCs currently defined.  Below they are listed without the "C" call and without the newline before the body data.

* Octaves.Connect {min:int, max:int} - min and max are for keys (midi numbers)
* Piano.GetState {full: 1} - return "R" newState with JSON object of full or abbreviated config.  full config returned once automatically on connect with subscribe to properties.
* System.Reboot - INCOMPLETE - will shut down the Piano, will probably not reboot
* System.SaveConfig <body> = config file

# LED command/event

LED commands and events are identical.  You may send an LED command even if you don't subscribe to the events.  They are binary messages with opcode=2.  The first byte is "L", the second is the midi note number of the starting (minimum) LED.

Color data follows in RGB format. One byte for each color channel, 0-255 each.  Any number of colors may follow, up the maximum midi note number of 127.

Generally the Piano sends two different LED messages, one for single note changes, and one to update the whole piano:

LnRGB

or

LnRGBRGBRGBRGBRGBRGBRGBRGBRGB...

where L="L", n=Piano.minkey, and then a bunch of RGB bytes for each LED for the whole Piano.  The number of LEDs updated is determined by the length of the message.

# Key up / Key Down command/event

These messages are identical binary messages, with opcode=2.  They are two or three bytes total.

DnV, UnV, Dn, Un

D="D", U="U", n=MIDI note number, V=MIDI velocity

Generally on hardware events there is no attached velocity, so a 2-byte message is used.  When the Piano software notifies you of the event, a velocity is attached, from Piano.velocity.  If you sent a velocity with the event, the notification will return that velocity

# Note On / Note Off command/event

Very similar to key commands and events.  The difference is that this is not necessarily from a hardware event, it may be from a midi event, sequencer, game, or other source.  It folows exactly the same format.

NnV, FnV, Nn, Fn

N="N", F="F", n=MIDI note number, V=MIDI velocity

The Piano will always attach a velocity.  I'm not sure that a command without velocity even makes sense in this case.
