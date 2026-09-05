<picture>
  <source media="(prefers-color-scheme: dark)" srcset="/docs/images/klor-font-logo-dark.svg">
  <source media="(prefers-color-scheme: light)" srcset="/docs/images/klor-font-logo-bright.svg">
  <img alt="KLOR logo font" src="/docs/images/klor-font-logo-bright.svg">
</picture>

# ZMK CONFIG FOR THE KLOR SPLIT KEYBOARD

[Here](https://github.com/GEIGEIGEIST/qmk-config-klor) you can find the QMK config for the KLOR.\
[Here](https://github.com/GEIGEIGEIST/klor) you can find the hardware files and build guide.

KLOR is a 36-42 key column-staggered split keyboard. It supports a per key RGB matrix, encoders, OLED displays, a Pixart Paw3204 trackball and four different layouts, through brake off parts.

![KLOR layouts](/docs/images/klor-layouts.svg)

Polydactyl is the default layout. If you choose one of the other layouts you can use the matching template in the default keymap.


## HOW TO USE

- fork this repo
- `git clone` your repo, to create a local copy on your PC (you can use the [command line](https://www.atlassian.com/git/tutorials) or [github desktop](https://desktop.github.com/))
- adjust the klor.keymap file (find all the keycodes on [the zmk docs pages](https://zmk.dev/docs/codes/))
- `git push` your repo to your fork
- on the GitHub page of your fork navigate to "Actions"
- scroll down and unzip the `firmware.zip` archive that contains the latest firmware
- connect the left half of the KLOR to your PC, press reset twice
- the keyboard should now appear as a mass storage device
- drag'n'drop the `klor_left-nice_nano_v2-zmk.uf2` file from the archive onto the storage device
- repeat this process with the right half and the `klor_right-nice_nano_v2-zmk.uf2` file.


## KNOWN ISSUES

- ~~The encoder on the secondary side doesn't work yet.~~ No longer true on this
  config. With the dongle central, *both* halves are peripherals and both
  encoders report fine. Each half sends its sensor's index within
  `sensors = <&left_encoder &right_encoder>`, so entry 0 of `sensor-bindings`
  is always the left encoder and entry 1 the right, whichever half sent it.
- Need to add the code for the Pixart Paw3204 trackball.

## MIC MUTE SETUP (macOS)

The right encoder's push button sends **Ctrl+Shift+Opt+Cmd+M**, not a mic mute.
ZMK cannot send a real one: its HID report descriptor carries only the
Keyboard, Consumer and Mouse pages, and "Phone Mute" lives on the Telephony
page, which ZMK doesn't implement. `C_MUTE` is speaker mute. macOS has no
built-in global mic-mute shortcut either, so the Mac has to listen for the
combo:

1. Open **Shortcuts.app** and create a new shortcut named `Toggle Mic`.
2. Add a **Run Shell Script** action with:

   ```bash
   if [ "$(osascript -e 'input volume of (get volume settings)')" = "0" ]; then osascript -e 'set volume input volume 100'; else osascript -e 'set volume input volume 0'; fi
   ```

3. In the shortcut's details pane, **Add Keyboard Shortcut** and press
   Ctrl+Shift+Opt+Cmd+M (the encoder button sends exactly this, so you can
   just press the encoder).

This mutes at the system input level, so it applies to every app rather than
one conferencing client. Note that some apps (Zoom among them) hold their own
mute state independently of the system input volume.
