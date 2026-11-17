# TI84-Flashing

Get a TI-84 calculator online and send apps and programs to it over Wi-Fi, using an ESP32.

The ESP32 plugs into the calculator's link port and talks to it using the TI link protocol. It connects to Wi-Fi, logs in to a home Synology NAS, and lets the calculator browse and download files from a folder on that NAS.

## What it does

- **Link port communication:** the ESP32 speaks the TI link protocol to the calculator (sending and receiving variables).
- **Calculator-side file browser:** a program on the calculator asks the ESP32 for a file list, shows it, and requests the file you pick.
- **NAS client:** logs in to a Synology NAS over HTTPS, lists TI files (`.8xk`, `.8xp`, `.8xl`, `.8xv`, `.8xn`, `.8xi`), and downloads them.
- **App flashing library:** parses `.8xk` flash applications (Intel HEX inside a TI wrapper) and sends them to the calculator in 128-byte chunks.
- **Wi-Fi setup from the calculator:** the calculator sends `SSID:PASSWORD` to the ESP32, which saves it and connects, so the Wi-Fi details don't need to be hard-coded.
- **Serial debug console:** commands over USB serial for testing without the calculator (see below).

## Hardware

| Part | Details |
|---|---|
| Microcontroller | ESP32 (Wemos D1 Mini32 board) |
| Calculator | TI-84 Plus |
| Wiring | The ESP32 is wired internally to the calculator's link port. The tip and ring lines connect to ESP32 pins `TIP_PIN` and `RING_PIN`. |
| Server | Synology NAS with File Station enabled |

## Project structure

```
src/main.cpp              Startup, Wi-Fi connect, main loop (serial + link events)
lib/TICalcLink/
  tiComms.cpp             Link port handling and the calculator <-> ESP32 protocol
  nasClient.cpp           Synology NAS login, file list, file download
  commands.cpp            USB serial debug commands
lib/TIFlashing/
  tiAppSend.cpp           .8xk parsing and flash app transfer
lib/ArTICL/               Third-party TI link library (git submodule)
data/                     Sample file for testing
platformio.ini            PlatformIO build config
```

## How the calculator and ESP32 talk

The calculator sends values through variables, and the ESP32 answers by queuing variables for the calculator to fetch.

| Calculator sends | Meaning | ESP32 replies |
|---|---|---|
| `Str9` = `"SSID:PASS"` | Connect to this Wi-Fi network | `A` = 1 (success) or 0 (failed) |
| `T` = 1 | List files on the NAS | Up to 6 filenames in `Str1`-`Str6` (cut to 14 characters), and `N` = file count (`-1` not connected, `0` listing failed) |
| `T` = 2, then `F` = n | Download file number n and send it to the calculator | File transferred as a flash app (`.8xk`) or a program/variable |

## Serial commands

| Command | What it does |
|---|---|
| `CONNECT` | Connect to Wi-Fi and log in to the NAS |
| `DISCONNECT` | Log out and disconnect |
| `LIST` | List TI files on the NAS |
| `SEND:file.8xk` | Download a file from the NAS and send it to the calculator |
| `A=9` | Queue a real variable for the calculator |
| `STR1=HELLO` | Queue a string for the calculator |

## Known limitations

- The calculator-triggered download is limited to about 24 KB per file (48 KB over serial).
- The calculator file list shows 6 files at a time, with names cut to 14 characters.
- HTTPS certificate checking is turned off (`setInsecure()`) because a home NAS typically uses a self-signed certificate.
- Wi-Fi details received from the calculator are stored unencrypted in the ESP32's flash.

## What I learned

I didn't know C/C++ when I started, and I built this to learn it, along with how hardware devices communicate at a low level: reading a serial protocol, parsing binary file formats, and working within a microcontroller's memory limits.

I used AI (mainly Claude) throughout the project to write and explain code, and I worked through the code until I understood how it all fits together. The idea and the direction came from the people credited below.

More about the project: [portfolio write-up](https://tntren.github.io/webdev/#TI84Wifi)

## Credits

- Inspired by [ChromaLock](https://www.youtube.com/@ChromaLock) and [KermMartian](https://github.com/KermMartian)
- [ArTICL](https://github.com/KermMartian/ArTICL) by KermMartian, the library that lets custom hardware like an ESP32 talk to a TI calculator. I modified it slightly to work with my setup.
- [ArduinoJson](https://arduinojson.org/) by Benoit Blanchon
- `data/noteflio.8xk` notes app made by Texas Instruments