# Polybar Spotify Module

![polybar-spotify-module](https://github.com/corigne/polybar-spotify-module/raw/main/screenshots/capture2.png)

This is a fork of [mihirlad55/polybar-spotify-module](https://github.com/mihirlad55/polybar-spotify-module/)
which addresses breaking changes introduced by polybar updates 3.6 onward.

The service/daemon (depending on how you choose to run it) signals polybar via
specific polybar-msg calls to trigger hooks in the module on service start and
when a song is played, paused, or changed. There is also a program that 
retrieves the title and artist of the currently playing song on spotify.

Originally, I created this fork to address breaking changes from polybar in 
versions 3.6 and 3.7, which deprecated direct IPC pipe writes and changed IPC 
module configurations, respectively.

I have also added some minor features to the spotify-listener daemon.
Now the daemon checks for a running instance of spotify when the daemon launches 
or when spotify first launches after the daemon is started.
The daemon will update the polybar module immediately instead of updating on the 
first instance when the user presses 'play' or 'next/previous' in the app. 
This is accomplished with glib2/gio instead of raw dbus access, for brevity.

I will continue to at a minimum maintain this fork against future polybar 
updates until one of the maintainers has a chance to review for merge,
or indefinitely in the event that never happens.

## Requirements
DBus and glib2, a library which provides some modern types and bindings for dbus,
are used for listening to spotify track changes and when spotify starts/stops.
Obviously you need spotify and polybar as well:

`dbus glib2 polybar spotify`

To compile the program, you will need `make`.

You most likely already have the above packages installed. Systemd depends on
DBus, and you wouldn't be looking at this, if you didn't have polybar and
spotify installed.

## How to Set Up

### Installing the Program
Run the following code to clone the repo and install the programs.
```
git clone https://github.com/corigne/polybar-spotify-module
cd polybar-spotify-module/src/
sudo make install
```
Uninstall using `sudo make uninstall` from the src directory.

### Running spotify-listener in the Background
`spotify-listener` must run in the background for it to be able to listen for
track changes and communicate with polybar. There are two ways to keep it
running in the background. If you are using systemd, you can have it start at
system startup by running:

```
systemctl --user enable spotify-listener
```

Then, you can start the service now by running:

```
systemctl --user start spotify-listener
```

If you are not using systemd, make sure the `spotify-listener` program is
executed at startup. Something like
```
spotify-listener &
disown
```
in a startup script should accomplish that.


### Configuring Polybar
Your polybar configuration file should be located at `.config/polybar/config`

First make sure IPC is enabled. The following should be in your configuration
file under the `[bar/<your bar name>]` section:
```
[bar/main]
enable-ipc = true
```

If you plan to use icons for next/play/previous/pause buttons, make sure you
add (and have installed) an icon font accordingly such as NerdFonts or
FontAwesome under the `[bar/<your bar name>]` section:
```
[bar/main]
font-1 = Font Awesome 5 Free:size=10;1
```
Note that the font is specified in modules as a 1-based index, so `font-1` is
specified by `format-font = 2`.

Next, add the following spotify modules:
```
[module/previous]
type = custom/ipc
format = <output>
format-font = 2
; Default
hook-0 = echo ""
; When spotify active
hook-1 = echo "Previous"
click-left = "spotifyctl -q previous"


[module/next]
type = custom/ipc
format = <output>
format-font = 2
; Default
hook-0 = echo ""
; When spotify active
hook-1 = echo "Next"
click-left = "spotifyctl -q next"


[module/playpause]
type = custom/ipc
format = <output>
format-font = 2
; Default
hook-0 = echo ""
; Playing
hook-1 = echo "Pause"
; Paused
hook-2 = echo "Play"
click-left = "spotifyctl -q playpause"


[module/spotify]
type = custom/ipc
format = <output>
; Default
hook-0 = echo ""
; Playing/paused show song name and artist
hook-1 = spotifyctl -q status --format '%artist%: %title%'
```
You can replace the text for Pause/Play/Next/Previous with icons for each of
the hooks.

Lastly, make sure the new spotify modules are part of your bar. Make sure one of
the following lines is part of your modules.
```
modules-center = spotify previous playpause next
modules-left = spotify previous playpause next
modules-right = spotify previous playpause next
```

## Status Formatting
The `spotifyctl status` command has multiple formatting options. You can
specify the:
- Maximum output length
- Maximum artist length
- Maxiumum track title length
- Output Format
- Truncation string

By default, the above lengths are `INT_MAX` (no limit). Additionally, if max
length is specified, the artist and track title will not be truncated if
the untruncated output satisfies the output max length constraint.

The tokens `%artist%` and `%title%` can be used to specify the output format.

For example for the artist `Eminem` and track title `Sing For The Moment`
```
spotifyctl status --format '%artist%: %title%' --max-length 20 \
    --max-title-length 10 --max-artist-length 10 --trunc '...'
```
would result in the following output
```
Eminem: Sing Fo...
```

For more information and examples, you can run the command `spotifyctl help`.


## How it Works
The spotify-listener program connects to the DBus Session Bus and listens for
two particular signals:

- `org.freedesktop.DBus.ProprtyChanged` for track changes
- `org.freedesktop.DBus.NameOwnerChanged` for spotify disconnections

Using this information, it sends messages to spotify polybar custom/IPC modules
to show/hide spotify controls and display the play/pause icon based on whether
a song is playing/paused.

The spotifyctl program calls `org.mpris.MediaPlayer2.Properties.Get` method to
retreive status information and calls methods in the
`org.mpris.MediaPlayer2.Player` interface to pause/play and go to the
previous/next track.

Glib interface bindings are created to query the mpris2 interface on both
service launch and when state changes are detected to get the current status 
of spotify.

## Credits

[mihirlad55](https://github.com/mihirlad55): <https://github.com/mihirlad55/polybar-spotify-module>
Many thanks to @mihirlad55 for his work on the original module.
I enjoy the module immensely, and I use it on all of my systems!
I intend this project as a respectful contribution to mihirlad55's original work,
which helped me understand dbus better and encouraged me to start my own projects.
