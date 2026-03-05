# dvgrab

## Overview

This project uses a [fork of dvgrab](https://github.com/rpster/dvgrab) (v3.6.0) for FireWire DV capture. dvgrab is a command-line tool that captures DV and HDV video data via IEEE 1394 (FireWire) on Linux.

The original dvgrab was last maintained in 2016 and lacks features needed for headless, camera-controlled capture on the Raspberry Pi 5. This fork adds those features and fixes several bugs.

**Repository:** [https://github.com/rpster/dvgrab](https://github.com/rpster/dvgrab)

## Fork Enhancements

### `--record-start` flag

Polls the camera's transport status and begins capture only when the camera enters the record state. Unlike the default behavior (which sends a play command), this flag waits passively for the camera's own record button to trigger capture. The flag re-arms automatically between recording sessions.

This is the mode used by the controller's **Camera Controlled ON** mode:

```
dvgrab --record-start /mnt/dvmedia/captures/clip-
```

### Memory optimization

Replaced fixed-size static buffers with dynamic allocation throughout the codebase, reducing memory usage by approximately 88%. Frame data arrays are sized to actual need (144 KB for DV, 1 MB for HDV), decoder instances are shared via reference counting, and PES buffers grow progressively with caps. This is particularly important when running on the Raspberry Pi 5.

### Signal handling fix

Fixed Ctrl+C not terminating the process during device discovery. Uses `volatile sig_atomic_t` flags and `sigaction()` without `SA_RESTART` so signals properly interrupt blocking calls.

### Date correction

Corrected the year calculation from libdv, which returns raw 2-digit years. Applied century correction to prevent dates from displaying incorrectly (e.g., 2026 showing as 1926).

### Wind-stop fix

Fixed premature exits by requiring a play-to-wind-stop transition before treating wind-stop as end-of-tape.

## How the Project Uses dvgrab

The controller runs dvgrab as a child process managed by `dvgrab_manager.py`. Communication happens over a pseudo-terminal (`pty.fork()`) for bidirectional I/O.

### Capture Modes

| Mode | Command | Trigger |
|------|---------|---------|
| Camera Controlled (switch ON) | `dvgrab --record-start <prefix>` | Camera's record button |
| Interactive (switch OFF) | `dvgrab -i <prefix>` | Controller sends `c` to start, `Escape` to stop |

### Output Monitoring

The controller monitors dvgrab's stdout for these patterns:

| Pattern | Meaning |
|---------|---------|
| `Capture started` | Recording has begun |
| `Capture stopped` | Recording has ended |
| `send oops` | Camera disconnected |

See [Architecture](Architecture) for details on the state machine and `dvgrab_manager.py` module.

## Building from Source

### Dependencies

Install the FireWire development libraries (these are also installed by `install.sh`):

```bash
sudo apt-get install libraw1394-dev libavc1394-dev libiec61883-dev
```

### Build

```bash
git clone https://github.com/rpster/dvgrab.git
cd dvgrab
./bootstrap
./configure
make
sudo make install
```

This installs the `dvgrab` binary to `/usr/local/bin/dvgrab`, which is the path the controller expects (configured as `DVGRAB_BIN` in `config.py`).

### Verify

```bash
/usr/local/bin/dvgrab --version
```
