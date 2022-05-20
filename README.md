# mpv-sockets
A fork of a collection of bash scripts to allow easier and programmatic interaction with `mpv` sockets. This fork **highly** recommends the use of [mpvSockets.lua](https://github.com/johndovern/mpvSockets-umpv) for socket creation. There is no need for a wrapper when mpvSockets does the hard work of creating sockets.

This fork is focused on extending the functionality of seanbreckenridge's original scripts and making them more adaptable when using something like [umpv](https://github.com/johndovern/mpvSockets-umpv/tree/master/umpv) or [mpv-music](https://github.com/johndovern/mpvSockets-umpv/tree/master/mpv-music).

# Scripts
## Prompts
In the original version of these scripts fzf was used to respond to prompts. These scripts default to using dmenu. However, any script that may use a prompt can be given the `-f, --fzf` flag to use fzf instead of dmenu.

I use these scripts with hot-keys making dmenu the perfect way to respond to any prompts without the need to also spawn a terminal.

## mpv-active-sockets
Run `mpv-active-sockets` to get a list of all active sockets.

```bash
$ mpv-active-sockets
/tmp/mpvSockets/1643226338072141025
/tmp/mpvSockets/1643226355764534189
```

The `mpv-active-sockets` script is very useful when paired with other scripts like `mpv-communicate` or `mpv-currently-playing`.

The `mpvSockets.lua` script creates unique sockets based on the unix time of an instance. Because of this you can use something like

```bash
$ mpv-active-sockets --unique | tail -n1
```

to get the most recent _unique_ instance of mpv. I make use of just this with something like the following:

```bash
mpv /path/to/playlist.m3u & sleep 1 && mpv-communicate \
  "$(mpv-active-sockets --unique | tail -n1)" \
  '{ "command": ["script-binding", "playlistmanager/shuffleplaylist"] }'
```

Something like this would start playing a playlist and then after one second shuffle that same freshly created instance. The above requires [playlistmanager](https://github.com/jonniek/mpv-playlistmanager) to be installed for that example to work.

### Flags

`-h, --help` to show a help message.

`-s, --socket` takes a path to a socket. If it is an active mpv socket then mpv-active-sockets will exit successfully. If it is not an active mpv socket it will be removed and mpv-active-sockets will exit with an error code of 1.

`-m, --music` will list all but the `mpv_music` socket. See [environment variables](#environment-variables) for more info.

`-u, --umpv` will list all but the `umpv_socket` socket. See [environment variables](#environment-variables) for more info.

`-U, --unique` will list all sockets but `mpv_music` or `umpv_socket`.

## mpv-communicate
mpv-communicate is a basic `socat` wrapper to send commands to the IPC server.

To illustrate:

If I have two instances of `mpv` open:

```bash
$ mpv-active-sockets
/tmp/mpvSockets/1643226338072141025
/tmp/mpvSockets/1643226355764534189
```

To get metadata from the oldest launched `mpv` instance I can run:

```bash
$ mpv-communicate "$(mpv-active-sockets --unique | head -n 1)" '{ "command": ["get_property", "metadata"] }' | jq
{
  "data": {
    "title": "Roundabout",
    "album": "Fragile",
    "genre": "Progressive Rock",
    "track": "01/9",
    "disc": "1/1",
    "artist": "Yes",
    "album_artist": "Yes",
    "date": "1972"
  },
  "request_id": 0,
  "error": "success"
}
```

`--unique` ensures only unique instances will be returned. Pipping that to `head -n 1` gives us the oldest unique socket.

### Flags
`-m, --music` to send the given command to the `mpv_music` socket. No socket is needed.

`-u, --umpv` to send the given command to the `umpv_socket` socket. No socket is needed.

## mpv-currently-playing
`mpv-currently-playing` is a `mpv-get-property` wrapper that gets information about the currently playing mpv instance. If there are multiple sockets, prints multiple lines, with one for each socket.

By default that will print the full path of the song that's currently playing, but you can provide the `--socket` flag to print the sockets instead.

Lots of scripts here use `mpv-currently-playing` internally, as interacting with the currently playing `mpv` instance is pretty useful.

### Flags
`-a, --all` will include paused sockets.

`-M, --media-title` can be used to get the value of a sockets media-title if one exists (it will be the path if this value is not present).

`-m, --music` to ignore the `mpv_music` socket.

`-u, --umpv` to ignore the `umpv_socket` socket.

`-U, --unique` to only check unique instances of mpv.

## mpv-get-property
Takes a socket as the first argument and interpolates the second argument into the `get_property` `command` syntax.

```bash
$ mpv-get-property "$(mpv-active-sockets)" path  # this works if there's only one instance of mpv active
Music/Yes/Yes - Fragile/01 - Roundabout.mp3
```
See the [mpv docs](https://mpv.io/manual/master/#properties) for available properties that can be queried using `mpv-get-property`.

This script takes no flags.

## mpv-next
Formerly `mpv-next-song`.

Running `mpv-next` without any flags will get all currently playing instances of mpv then play the next item in the playlist of the most recent instance of mpv. At least if you don't mave an instance of umpv or mpv-music open.

### Flags
`-f, --fzf` if you want to be prompted by fzf instead of dmenu.

`-m, --music` go to the next file for the `mpv_music` socket.

`-p,--pick` choose the mpv instance you want to move to next file.

`-u, --umpv` go to the next file for the `umpv_socket` socket.

`-s, --socket` goes to the next file for the given socket.

## mpv-pick
`mpv-pick` is used internally by the following scripts:

- `mpv-next`
- `mpv-quit`
- `mpv-prev`
- `mpv-toggle`

By itself it allows a user to interactively pick an mpv socket. If a socket is chosen it will be printed to the terminal.

### Flags
`-f, --fzf` use fzf as your prompt instead of dmenu. Any of the scipts that use `mpv-pick` also accept this flag.

`-F, --full` By default mpv-pick will return only the socket of the chosen instance. If you also want the media-title/path then use this flag.

`-m, --music` will ignore the `mpv_music` socket.

`-u, --umpv` will ignore the `umpv_socket` socket.

`-U, --unique` will list only the unique instances of mpv.

## mpv-prev
This is a new script to this repo. I have no clue why it is not in the original, I was convinced it was in the original but I can't find it there at the time of writing this.

`mpv-prev` does the same thing as mpv-next except it goes to the previous file in the playlist. This takes the same flags as mpv-next.

### Flags
`-f, --fzf` if you want to be prompted by fzf instead of dmenu.

`-m, --music` go to the prev file for the `mpv_music` socket.

`-p,--pick` choose the mpv instance you want to move to prev file.

`-u, --umpv` go to the prev file for the `umpv_socket` socket.

`-s, --socket` goes to the prev file for the given socket.

## mpv-quit
Formerly `mpv-quit-pick`. I have integrated the functionality of `mpv-quit-latest` into this script and removed that script.

Run `mpv-quit` and be prompted to select a currently playing instance of mpv. The selected instance will be quit. If only one instance exists then it will be quit and you will not be prompted.

### Flags
`-f, --fzf` if you want to be prompted by fzf instead of dmenu.

`-l, --latest` quit the latest instance of mpv. This does not include instances of umpv or mpv-music. No prompt is shown.

`-m, --music` quit mpv-music. No prompt is shown.

`-u, --umpv` quit umpv. No prompt is shown.

`-s, --socket` quit the given socket.

## mpv-seek
Run `mpv-seek 10` to pick an instance of mpv and skip forward 10 seconds. If no number is given the chosen instance will seek forward 5 seconds. If only one instance exists you will not be prompted.

### Flags
`-f, --fzf` if you want to be prompted by fzf instead of dmenu.

`-l, --latest` seek the latest instance of mpv. This does not include instances of umpv or mpv-music. No prompt is shown.

`-m, --music` seek mpv-music. No prompt is shown.

`-u, --umpv` seek umpv. No prompt is shown.

`-s, --socket` seek the given socket.

## mpv-song-description & mpv-song-description-py
`mpv-song-description` constructs a description from the `metadata`

```bash
$ mpv-song-description
Can You Feel My Heart - Bring Me The Horizon (Sempiternal)
```

## mpv-toggle
Formerly `mpv-play-pause`.

Run `mpv-toggle` to toggle the most recently toggled instance of mpv. This tries to be smart and tracks the last instance that was toggled. If that instance no longer exists then it will attempt to detect if any instances exist. If more than one instance is running (not playing) then you will be prompted to choose which instance to toggle. If only one instance exists then you will not be prompted.

### Flags
`-f, --fzf` if you want to be prompted by fzf instead of dmenu.

`-m, --music` toggle mpv-music. No prompt is shown.

`-u, --umpv` toggle umpv. No prompt is shown.

`-s, --socket` toggle the given socket. No prompt is shown.

## Environment Variables
The following environment variables are used by the scripts in this repo.

You can set any of these in you `.bashrc` or `.zprofile` or however you normally set your environment variables.

### `$MPV_SOCKET_DIR`
If unset this will default to `/tmp/mpvSockets`

This should be set to a directory that you have full permission to access and write to.

### `$MPV_MUSIC_SOCKET`
If unset this will default to `${MPV_SOCKET_DIR}/mpv_music`. If your `$MPV_SOCKET_DIR` variable is unset the default path will be `/tmp/mpvSockets`

When set this should be the full path to the location where you want your mpv-music socket to be located.

### `$MPV_SOCKET_DIR`
If unset this will default to `${MPV_SOCKET_DIR}/umpv_socket`. If your `$MPV_SOCKET_DIR` variable is unset the default path will be `/tmp/mpvSockets`

When set this should be the full path to the location where you want your umpv socket to be located.

## Install

Dependencies: [`mpv`](https://mpv.io/), [`socat`](https://linux.die.net/man/1/socat), [`jq`](https://github.com/stedolan/jq), `sed`, ([dmenu](https://tools.suckless.org/dmenu/) or [`fzf`](https://github.com/junegunn/fzf) for `mpv-next`, `mpv-prev`, `mpv-quit`, and `mpv-toggle`)

```bash
git clone https://github.com/johndovern/mpv-sockets
cd ./mpv-sockets
make install
```

By default these scripts will be installed to `~/.local/bin`. If this is not in your `$PATH` you will want to adjust the Makefile or manually place them in your desired directory.

If you only want to install some but not all of the scripts here be aware that some of them depend on eachother. The following is a list of which scripts depend on another script.

- `mpv-active-sockets` Deps: None
- `mpv-communicate` Deps: None
- `mpv-currently-playing` Deps:
  - `mpv-active-sockets`
  - `mpv-get-property`
- `mpv-next` Deps:
  - `mpv-communicate`
  - `mpv-currently-playing`
- `mpv-pick` Deps:
  - `mpv-active-sockets`
  - `mpv-get-property`
- `mpv-prev` Deps:
  - `mpv-communicate`
  - `mpv-currently-playing`
- `mpv-quit` Deps:
  - `mpv-communicate`
  - `mpv-currently-playing`
  - `mpv-pick`
- `mpv-seek` Deps:
  - `mpv-communicate`
  - `mpv-currently-playing`
  - `mpv-pick`
- `mpv-toggle` Deps:
  - `mpv-active-sockets`
  - `mpv-communicate`
  - `mpv-currently-playing`
  - `mpv-pick`

Pay attention to the scripts that depend on `mpv-currently-playing` as that script has two of it's own dependencies.

### Daemon

I run [`mpv-history-daemon`](https://github.com/seanbreckenridge/mpv-history-daemon) in the background, which polls for new sockets at `/tmp/mpvsockets`, grabbing file info, metadata, and whenever I play/pause/skip anything playing in `mpv`. That creates a local scrobbling history for `mpv` - letting me create a `mpv` history, and do statistics on which songs/videos I listen to often.

```
1598956534118491075|1598957274.3349547|mpv-launched|1598957274.334953
1598956534118491075|1598957274.335344|working-directory|/home/sean/Music
1598956534118491075|1598957274.3356173|playlist-count|12
1598956534118491075|1598957274.3421223|playlist-pos|2
1598956534118491075|1598957274.342346|path|Masayoshi Takanaka/Masayoshi Takanaka - Alone (1988)/02 - Feedback's Feel.mp3
1598956534118491075|1598957274.3425295|media-title|Feedback's Feel
1598956534118491075|1598957274.3427346|metadata|{'title': "Feedback's Feel", 'album': 'Alone', 'genre': 'Jazz', 'album_artist': '高中正義', 'track': '02/8', 'disc': '1/1', 'artist': '高中正義', 'date': '1981'}
1598956534118491075|1598957274.342985|duration|351.033469
1598956534118491075|1598957274.343794|resumed|{'percent-pos': 66.85633}
1598956534118491075|1598957321.3952177|eof|None
1598956534118491075|1598957321.3955588|mpv-quit|1598957321.395554
Ignoring error: [Errno 32] Broken pipe
Connected refused for socket at /tmp/mpvsockets/1598956534118491075, removing dead socket file...
/tmp/mpvsockets/1598956534118491075: writing to file...
```
