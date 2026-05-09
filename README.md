# gnosis_vpn-monitor

A small Python script that records and plots `ping` / `curl` traces for
gnosisVPN. Useful for spotting outages, latency spikes and throughput
drops.

The script depends only on the Python 3 standard library — no `pip
install` required. The recorders shell out to system `ping` and `curl`;
the plotter has no external dependencies at all.

## Layout

```
gnosis_vpn-monitor   the script (one file)
data/                recordings (created automatically; samples ship here)
output/              rendered SVG charts
README.md
```

Both `data/` and `output/` are resolved relative to the script itself,
so the paths are stable regardless of which directory you run from. The
script auto-creates parent directories for any `-o` path it writes to.

## Subcommands

```
gnosis_vpn-monitor ping [HOST]   # record ping latency
gnosis_vpn-monitor curl [URL]    # record curl throughput
gnosis_vpn-monitor plot FILE     # render an SVG chart
```

Run any subcommand with `--help` for the full option list.

## Recording

### Ping latency

```sh
./gnosis_vpn-monitor ping                        # default host: google.com
./gnosis_vpn-monitor ping example.org            # custom host
./gnosis_vpn-monitor ping example.org -o run.txt # custom output file
```

Internally this is `ping -D <host>` — `-D` prefixes every reply line with
a `[unix-seconds.microseconds]` tag, which is what the plotter keys off.
Output is teed to stdout so you see progress live; a copy is also
written to `data/ping.txt` by default. Stop with Ctrl-C.

### Curl throughput

```sh
./gnosis_vpn-monitor curl                        # default URL: kernel.org tarball
./gnosis_vpn-monitor curl https://example.com/big.bin
./gnosis_vpn-monitor curl https://... -o run.txt
```

The script runs `curl -o /dev/null <url>` and timestamps each progress
update. We discard the body (we only care about throughput numbers) so
repeated runs don't accumulate downloaded files. Output is teed to
stdout and written to `data/curl.txt` by default. The recording ends
when the download finishes or you Ctrl-C out.

This is a pure-Python equivalent of the original shell pipeline:

```sh
curl -O <url> 2>&1 \
  | stdbuf -oL tr '\r' '\n' \
  | awk '{ print strftime("[%s.%6N]"), $0; fflush() }' \
  | tee curl.txt
```

…but with proper microsecond timestamps (no `%6N` literal artefacts on
systems whose `strftime` lacks the GNU `%N` extension).

### Skipping the file copy

Both recorders write to the file by default. Pass `-o ''` to record only
to stdout:

```sh
./gnosis_vpn-monitor ping -o '' | grep -v 'time=1[0-9][0-9]'
```

## Plotting

```sh
./gnosis_vpn-monitor plot data/ping.txt
./gnosis_vpn-monitor plot data/curl.txt
```

Each input file's type is auto-detected from its content. With no
`-o`, the chart is written to a timestamped path under `output/`:

```
output/ping-2026-05-09--22-19-39.svg
output/ping-2026-05-09--22-19-39.png
```

A sibling PNG is rendered alongside the SVG so the chart drops into
docs and chat tools that don't speak SVG. The PNG step needs one of
`rsvg-convert` (debian: `apt install librsvg2-bin`), ImageMagick, or
Inkscape; if none is installed the SVG is still written and a one-line
note explains the missing converter.

Pass `-o some/path.svg` to override the location (the `.png` sibling
follows the same name), or `-o -` to write the SVG to stdout (in which
case no PNG is generated).

### Linestyle, marker, colour (`--style`)

Each series is styled with a matplotlib-style format string. The default
is `xb` — blue **x** markers, no line — which makes individual samples
visible without a connecting line that would smooth over outages.

```sh
./gnosis_vpn-monitor plot data/ping.txt --style o-r        # red solid line + open circles
./gnosis_vpn-monitor plot data/ping.txt --style '.--g'     # green dashed line + dots
./gnosis_vpn-monitor plot --double-y data/ping.txt data/curl.txt \
    --style 'x-b' 's--g'                                   # blue line+x, green dashed+squares
```

Combine any of these in any order:

| component  | codes                              |
|------------|------------------------------------|
| linestyle  | `-` `--` `-.` `:`                  |
| marker     | `.` `o` `x` `+` `*` `s` `D` `^` `v`|
| colour     | `b` `g` `r` `c` `m` `y` `k` `w`    |

Pass two strings for `--double-y` to style each series independently;
pass one and both series share it. Quote any string containing `*` to
keep the shell from globbing.

### Image size and PNG resolution

```sh
./gnosis_vpn-monitor plot data/ping.txt --width 2400 --height 1000
./gnosis_vpn-monitor plot data/ping.txt --png-scale 2     # 3600×1680 PNG
```

`--width` / `--height` set the SVG dimensions in pixels (default
1800×840). `--png-scale` multiplies the SVG dimensions when rendering
the PNG — e.g. `--png-scale 2` gives a PNG with double the pixel
count of the SVG, useful for retina displays or print without
inflating the on-disk SVG.

### Combined chart (`--double-y`)

Plot ping latency and curl throughput on one chart with independent
left/right y-axes:

```sh
./gnosis_vpn-monitor plot --double-y data/ping.txt data/curl.txt
# → output/combined-2026-05-09--22-19-39.{svg,png}
```

The first file is plotted against the left axis (steel blue), the
second against the right (orange).

### Logarithmic y-axis (`--log-y`)

```sh
./gnosis_vpn-monitor plot --log-y data/ping.txt
./gnosis_vpn-monitor plot --double-y --log-y data/ping.txt data/curl.txt
```

`--log-y` switches the y-axis to log10. With `--double-y` it applies to
both axes independently. Non-positive samples (e.g. the leading zeros
emitted by curl before the download starts) are dropped, since they
have no place on a log scale.

## File format

A recording is a plain text file with one timestamped sample per line:

```
[1778315104.443890] 64 bytes from ... time=177 ms
[1778315105.426565] 64 bytes from ... time=352 ms
```

Or, for curl:

```
[1778315901.123456]   0  149M    0  111k    0     0  84162  ...  84137
[1778315902.234567]   0  149M    0  847k    0     0   359k  ...   359k
```

Headers and other non-data lines are tolerated and skipped during
parsing. This means hand-edited recordings, recordings made with the
older shell pipeline, and recordings from this script all parse with
the same code path.

## Source layout

The whole tool lives in one file (`gnosis_vpn-monitor`) so it can be
copied around without packaging. The file is divided into clearly
labelled sections:

- **Project layout / Recording defaults** — paths and defaults
- **Parsing** — regexes and `parse_recording()`
- **Value formatting** — millisecond / bytes-per-second tick labels
- **Tick generation** — `linear_ticks()` and `log_ticks()`
- **Axes** — the `Axis` class (linear or log)
- **SVG rendering** — geometry constants, `Plot` helper, `render()`
- **Recording** — `record_ping()` and `record_curl()` (subprocess + tee)
- **CLI** — `argparse` subcommand dispatch
