# Babashka Status Bar

An efficient textual status bar for use with dwm, tmux, etc.

Written in clojure, runs with the amazing [babashka](https://babashka.org/) (requires version >= 8.3)

Artificial output example (normally only noteworthy info is shown) with my personal config:
```
cpu:     chrome   1% /  2% | mem: 23% | wifi: 38% | cloud: 2 mach | vm: 1 mach | disk: 20% | prof: low-power | git: 3 dirty | mnt: google-drive, usb | vpn: on | security: ok | arch: 51 updates | temp: 37C | batt: 78% D 06:07 | vol:  52% | Sun 28 May 23:44:16
```

It is not meant to be used as-is - customise it by changing the code. If you have a problem, check the BUGS section below, check the log, use the `debug` target.

This script is the one I use at home and is very linux-centric. Many of the monitors make sense only on my own setup, but you won't be surprised to find clojure is a great fit for implementing your own ones. Plus babashka starts up very quickly (and is trivial to deploy as it is just one binary).

See variables `targets` and `monitor-specs` for configuration/examples.

# Usage
```
$ status-bar
Usage: status-bar run TARGET
   or: status-bar run TARGET MONITOR ...
       Can be used during development to only start specific monitors,
       whatever the value of their `enabled?`` property.
   or: status-bar log TARGET [COMMAND ...]
       View the log file using less (or the specified command).
   or: status-bar trigger MONITOR ...
       If the monitors have a `compute` function, requests an immediate update.

Targets:  dwm tmux stdout debug
Monitors: alert alert-test stderr-test time time-with-command time-with-thread volume profile battery temperature arch secrets security cloud vpn mounts git wifi vm disk memory cpu dnsmasq
```
Examples:
```
# The stdout target echoes the bar to stdout
$ status-bar run stdout
logging to /home/henri/.cache/status-bar/stdout.log
batt: 62% D 01:29 | vol:  17% | Thu 26 May 16:45:12
python2  21% / 57% | batt: 62% D 01:29 | vpn: BUG | vol:  17% | Thu 26 May 16:45:13
...

# Refresh the volume monitor immediately
$ status-bar trigger volume

# In the output above, you see "vpn: BUG", meaning that an exception
# was raised when computing the value. You can see the details in the log:
$ status-bar log stdout
...

# During development it may be useful to use the debug target, so
# that both the bar and logging goes to stdout:
$ status-bar run debug
2022/05/28 23:03:47.223 ptoseis INFO [user:586] - starting
2022/05/28 23:03:47.336 ptoseis DEBUG [user:525] - consuming [:temp nil]
2022/05/28 23:03:47.342 ptoseis INFO [user:136] - bar=
2022/05/28 23:03:47.343 ptoseis DEBUG [user:525] - consuming [:battery "batt: 87% D 06:06"]
2022/05/28 23:03:47.343 ptoseis INFO [user:136] - bar=  batt: 87% D 06:06
...

# Again for development, you can specify the monitors you want to
# run, here only time and cpu:
$ status-bar run debug time cpu
...
```

## Targets

The TARGET command line parameter specifies which system to publish the bar to: tmux, dwm, stdout, debug.

See the `targets` variable for details.

It is possible for one user to run multiple status-bar processes concurrently as long as they have different targets (eg tmux and dwm).

On startup, status-bar kills any instance running with the same target. I.e. `status-bar run tmux` will kill any pre-existing `status-bar run tmux` process. This makes it easier to upgrade the script.

## Triggers

For the monitors using a `compute` function, it is possible to trigger them from the outside.

E.g. my vpn status is only checked every minute, meaning that it could take up to 59 seconds for the status bar to reflect a change.

But I manage the vpn from a script, which calls the command `status-bar trigger vpn` after switching the vpn on/off, allowing for an immediate update of the bar. Same with the audio volume monitor.

## Implementing a monitor

When defining a monitor in `monitor-specs` you can either:
1. Provide a `compute` function, to be called periodically. It should return a string to be shown on the bar. The monitor is hidden when `compute` returns nil. Only monitors of this type can be triggered externally.
2. Or provide a command to be run as a child process. Each new line on stdout will be displayed on the bar. The monitor is hidden when the command prints an empty line.
3. Or provide an `thread` function, which will be running continuously in a thread, and will push its text to channel `results-chan`. The monitor is hidden when the function pushes `nil`.

With my own config, only the time and the volume are always shown. But for example the cpu or battery monitors may show up temporarily, if there is something noteworthy to report.

### Examples

These monitors are implemented using each of the methods described above, but all produce the same result:
```clojure
{:id :time-with-compute
 :label "time: "
 :interval 1000
 :compute #(.format (LocalDateTime/now) time-formatter)}

{:id :time-with-command
 :label "time: "
 :command ["bash" "-c" "while :; do date; sleep 1; done"]}

; There is no label parameter here, the thread must add any label itself
{:id :time-with-thread
 :thread #(while true
            (push-result :time-with-thread
                         (str "time: " (.format (LocalDateTime/now) time-formatter)))
            (Thread/sleep 1000))}
```
More examples in the script.

### Using external commands

Monitors using commands allow using any programming language. The commands control what appears on the bar by just printing a line to stdout. When a line is blank, the monitor is temporarily hidden from the bar.

For example, CPU monitoring is now done by an external babashka script, see `./status-bar-cpu`.

You can run it manually:
```
# sampling period 2 seconds, cpu threshold 20%
$ ./status-bar-cpu 2 20
      node*2  15% /  27%
    chrome*3  13% /  21%

    chrome*2  27% /  30%
...
```
Interpretation of the first line: 2 `node` processes were together using 15% cpu (other `node` processes may exist, but they used no cpu over the sampling period), and the total cpu usage on the machine was 27%.

The third line was blank because the total cpu usage was less than the threshold passed as a second parameter. This blank line instructs the main `status-bar` script to hide the monitor.

The fourth line was not blank, the monitor would reappear on the bar.

## Bugs

- Not tested on mac
