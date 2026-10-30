# Local piHPSDR build and audio verification
Recorded 2026-10-30 for mdufresne's x86-64 Linux Mint workstation. These are local notes, not upstream instructions. No macOS build or Hermes-Lite 2 hardware was tested.

## Source and build identity
- Upstream: https://github.com/dl1ycf/pihpsdr
- Branch: `Rel-3.0`
- Pinned commit: `3257c385006ccf0035a666d42ca8861162c8926f`
- Commit date: 2026-10-27; the binary's reported build date is derived from this commit, not the actual compilation date.
- Local checkout: `/home/mdufresne/git/pihpsdr`
- Compiler: Ubuntu GCC 13.3.0 (`13.3.0-6ubuntu2~24.04.1`); GNU Make 4.3.
- Build command: `make -C /home/mdufresne/git/pihpsdr -j8`.
- Binary reports `TCI MIDI SOAPYSDR AudioModule=PIPEWIRE`.
- The version includes `-dirty` because of the required local RNNoise Makefile change below.
- WDSP, RNNoise, and libspecbleach are built from the bundled source and linked as static libraries. No separate global WDSP install was needed.

## Build dependencies
The targeted dependency installation added GTK3, PipeWire, ALSA and WebSocket development packages and their dependencies. The existing SoapySDR development package was retained. No SDR drivers or running audio services were replaced for this build.

Development packages for the enabled features:

```bash
sudo apt-get install --no-install-recommends \
  build-essential git pkg-config \
  libgtk-3-dev libpipewire-0.3-dev libasound2-dev \
  libsoapysdr-dev soapysdr0.8-module-rtlsdr \
  libwebsockets-dev libfftw3-dev libopus-dev libsqlite3-dev \
  libcurl4-openssl-dev:amd64 libssl-dev zlib1g-dev
```

Observed package versions:

```text
build-essential                  12.10ubuntu1
libasound2-dev:amd64              1.2.11-1ubuntu0.3
libcurl4-openssl-dev:amd64         8.5.0-2ubuntu10.16
libfftw3-dev:amd64                3.3.10-1ubuntu3
libgtk-3-dev:amd64                3.24.41-4ubuntu1.3
libopus-dev:amd64                 1.4-1build1
libpipewire-0.3-dev:amd64          1.0.5-1ubuntu3.3
libsoapysdr-dev:amd64             0.8.1-4build1
libsqlite3-dev:amd64              3.45.1-1ubuntu2.8
libssl-dev:amd64                  3.0.13-0ubuntu3.16
libwebsockets-dev:amd64           4.3.3-1.1build3
soapysdr0.8-module-rtlsdr:amd64    0.3.3-1build1
zlib1g-dev:amd64                  1:1.3.dfsg-3.1ubuntu2.3
```

The machine already has the RTL-SDR Blog driver installed by iNTERCEPT's setup. Reuse it; do not run piHPSDR's broad `LINUX/libinstall.sh` on this configured workstation. That script can rebuild SoapySDR, install unrelated backends, and change audio packages.

## Build configuration
Keep these contents in the source-root `make.config.pihpsdr`, rather than changing the upstream root Makefile:

```makefile
GPIO=OFF
MIDI=ON
USBOZY=OFF
SOAPYSDR=ON
PORTFORWARD=OFF
TCI=ON
AUDIO=PIPEWIRE
```

`SOAPYSDR=ON` is necessary for the RTL-SDR and is not the upstream default. GPIO and USB OZY are unnecessary here. Automatic UPnP router forwarding is deliberately disabled. TCI is compiled in; that does not mean a TCI server has been enabled.

### Required local RNNoise fix
The first build failed with `fatal error: rnnoise.h: No such file or directory` even though the header existed. The bundled RNNoise Makefile placed required include directories in a conditional `CFLAGS?=` assignment. Move them into `COMPILE` so inherited compiler flags cannot remove them.

The exact patch is saved as `local-build/rnnoise-include-path.patch`. For a fresh checkout of the pinned commit:

```bash
git apply --check local-build/rnnoise-include-path.patch
git apply local-build/rnnoise-include-path.patch
make -j8
./pihpsdr -V
```

Copy the local patch and `make.config.pihpsdr` into a fresh checkout before these commands. Do not apply the patch twice; check `git diff -- rnnoise/Makefile` on the existing checkout. After changing compile-time options, clean and rebuild rather than assuming existing objects will be rebuilt.

To obtain the original source independently:

```bash
git clone --branch Rel-3.0 https://github.com/dl1ycf/pihpsdr.git
git -C pihpsdr checkout 3257c385006ccf0035a666d42ca8861162c8926f
```

## Launchers, settings, and audio
- User launcher: `/home/mdufresne/bin/pihpsdr`, executing the binary in the checkout.
- Application-menu entry: `/home/mdufresne/.local/share/applications/pihpsdr.desktop`.
- Linux runtime directory: `/home/mdufresne/.config/pihpsdr/`.
- RTL-SDR preferences: `rtlsdr.props` in that directory.
- Logs: `pihpsdr.stdout` and `pihpsdr.stderr` in that directory; startup rewrites them.
- FFT optimization cache: `wdspWisdom01`. Its first creation took several minutes. Leave the optimization window open; later launches reuse the cache.
- The source directory must remain available because the user launcher points into it.

Start with `pihpsdr` or the application-menu entry. Stop other applications using the dongle first. iNTERCEPT may remain running as a web server, but it must not have an active capture.

For the receive test:
1. Select `Start` alongside `rtlsdr` in Discovery.
2. Use `Menu -> VFO` and enter `162.55`, then `MHz`.
3. Select `MODE -> FMN`.
4. Select `FILT -> 16k` for 5 kHz deviation. The `11k` option instead selects 2.5 kHz deviation.
5. In `Menu -> RX`, choose `Combo384 Amanero Analog Stereo` for `RX Audio Out`. `Only Audio to Radio` does not give local speaker playback.
6. Audio choices can change when switching modes; recheck the RX audio output after selecting FMN.

This machine's playback route is PipeWire -> Amanero Combo384, with 48 kHz stereo audio. It does not use Wine, a separate audio server, or an ALSA-exclusive route.

## RTL-SDR connectivity verification
Hardware: RTL-SDR Blog V4, Rafael Micro R828D tuner, serial `00000001`, USB ID `0bda:2838`.

The independent SoapySDR rate test was:

```bash
timeout --signal=INT --kill-after=3s 10s \
  SoapySDRUtil --args=driver=rtlsdr --rate=1536000 --direction=RX
```

It streamed approximately 1.535 MS/s. A timeout status from this deliberately bounded continuous test is expected. piHPSDR itself subsequently opened the device using CF32 format at 1.536 MS/s, started `soapy_receive_single_thread`, and displayed an updating spectrum/waterfall. All linked libraries resolved.

This verifies this Linux/Blog V4 combination, not HF reception, macOS, transmit operation, or HL2-specific controls. Hermes-Lite 2 uses native HPSDR Ethernet rather than the RTL-SDR Soapy backend.

## Actual piHPSDR audio capture, 2026-10-30
This was piHPSDR's own decoded audio, not a replacement `rtl_fm` capture.

Temporary test settings:
- NOAA weather channel: 162.550 MHz.
- Mode: FMN; 16 kHz filter and 5 kHz deviation.
- RF gain: 40.2 dB, manual; AF gain: -20 dB.
- SDR capture rate: 1.536 MS/s.
- Recording: 20 seconds, 48 kHz, stereo signed 16-bit PCM.

The selected PipeWire stream had node name `pihpsdr-rx` and media name `pihpsdr-playback`. `parec --monitor-stream=INDEX` captured only that stream from its sink monitor; FFmpeg wrote the WAV. Stream indexes change, so rediscover them with `pactl -f json list sink-inputs`. Neither the global default sink nor any application's routing was changed, and browser playback was excluded.

Evidence files, kept outside the source repository:
- `/home/mdufresne/Music/rtl-sdr/pihpsdr-noaa-162550-20261030.wav`
- `/home/mdufresne/Music/rtl-sdr/pihpsdr-noaa-162550-20261030.json`

Measured results:
- Duration: exactly 20.0 seconds.
- Peak: -6.15 dBFS on both channels; RMS: -26.23 dBFS.
- Clipped signed-16-bit samples: 0.
- Longest run of exact zero in both channels: 0.0417 ms, just two frames.
- No 100 ms block below -70 dBFS.
- DC offset: approximately 0.000000153 full scale.
- Both channels are identical, as expected for mono FM duplicated to stereo.
- Approximately 84.8% of measured 20 Hz–24 kHz spectral power is between 300 Hz and 3 kHz. This is consistent with voice-band filtering, not proof of intelligible speech.
- Post-capture `pw-top` observations showed piHPSDR stream `ERR=0`. The Amanero sink showed `ERR=1`, unchanged across four running samples. There was no pre-capture baseline to attribute that counter; zero end-to-end underruns cannot be claimed.

Conclusion: the sampled digital audio output is non-silent, has useful headroom, and shows no clipping or long exact-zero gaps. This is not a listening/intelligibility assessment, an RF SNR or SINAD measurement, a guarantee against audible stutter, or a test of the analog DAC/speakers. Audible hiss and scratchiness still require listening to the recording; weak RF reception or local interference may remain.

Play the evidence:

```bash
play /home/mdufresne/Music/rtl-sdr/pihpsdr-noaa-162550-20261030.wav
```

### State restoration
Before the temporary test, piHPSDR was closed and the saved VFO-A setting was 144.2002 MHz USB. The original `rtlsdr.props` was backed up to:

`/home/mdufresne/.config/pihpsdr/backups/audio-check-20261030-084600/rtlsdr.props`

After capture, piHPSDR was closed normally and that preference file was restored byte-for-byte. The initial closed state was preserved. The temporary NOAA settings above were not left as the user's saved configuration.

## Scope
The build reference, supporting patch, and RNNoise source fix are published on the `Rel-3.0` branch of https://github.com/mjdufresne/pihpsdr, a personal fork of the upstream project. Private runtime preferences, databases, credentials, audio recordings, and settings backups are not included. Temporary test screenshots and launch logs were removed after verification. Future source updates should recheck the build patch, dependency compatibility, and actual receiver operation.
