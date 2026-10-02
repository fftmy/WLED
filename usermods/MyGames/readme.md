# MyGames

A collection of small 2D games for LED matrices, provided as a usermod that
registers additional effects.

## Pong effect

The usermod registers a 2D **Pong** effect: the player paddle (a 1×3 block
in the rightmost column) against a red AI-controlled enemy paddle in the
leftmost column, with a small ball bouncing between them. Whoever lets the
ball pass their paddle loses; the winner's color flashes and the game then
waits for the gamepad's **A button** to restart. Without a connected
gamepad the player paddle is AI-driven too and the game restarts by itself
after a few seconds, so the panel keeps playing as a demo.

The effect requires a 2D matrix setup. On a 1D strip it just fills the strip
with the foreground color.

### Effect settings

| Setting | Function |
|---|---|
| Speed slider | Ball speed, 1-12 px/s (default 6.5) |
| Intensity slider | AI paddle top speed as a fraction of the ball speed (30-95%) |
| Color slot 0 | Player paddle + ball color |
| Color slot 1 | Background color |

The enemy paddle is always red. The ball reverses one pixel before reaching
a paddle, so it never visually touches it.

### Controls

| Control | Function |
|---|---|
| Left stick (up/down) | Move the paddle while held (fixed 200 px/s) |
| D-pad up / down | One pixel per press, auto-repeat while held |
| A button | Restart the game after it ended |

Hitting the ball with the paddle edge steers the return angle - that is how
the AI (which simply tracks the ball at a limited speed) can be beaten.

## Bluetooth gamepad (optional)

When the usermod is compiled with `-DMYGAMES_BLUEPAD32` (requires the
NimBLE-Arduino library, no platform fork needed), a BLE gamepad can control
the Pong effect:

| Input | Action |
|---|---|
| Left stick (up/down) | Move the paddle |
| D-pad up / down | Move the paddle one pixel per step |
| A button | Restart the game after it ended |

The usermod implements a small BLE HID host: it scans, connects, pairs
(Just Works bonding) and parses the gamepad's HID Report Map to learn the
input report layout, so no report format is hardcoded. Supported gamepads
include the Xbox Wireless Controller in BLE mode (Series X\|S, or One S with
firmware >= 4.x; update firmware with the Xbox Accessories app if needed),
and in principle any BLE gamepad exposing a standard HID service (0x1812)
with a gamepad/joystick report. Controllers that omit the Report ID byte in
their input notifications (despite declaring one in the map) are detected
automatically after a few reports.

Pairing: hold the controller's pair button until it advertises; the pairing
key is stored in NVS and the controller reconnects automatically afterwards
(and after a WLED reboot).

Timing note: the BLE stack starts ~10-45 s after boot - once WLED's network
is up, or well past boot in AP/unconfigured mode. The gamepad therefore
(re)connects shortly after WLED has finished booting, not instantly. This
delay is required: flash writes while the BLE controller is running abort
in spi_flash on ESP32-C3, and WLED writes WiFi credentials to NVS during
boot.

WiFi modem sleep note (ESP32-C3/S3): the coexist firmware requires WiFi
modem sleep to be enabled whenever both radios are active, otherwise the
coexist core aborts during `esp_bt_controller_enable` ("Should enable WiFi
modem sleep when both WiFi and Bluetooth are enabled"). WLED defaults to
`noWifiSleep = true`, so MyGames forces `WiFi.setSleep(true)` right before
starting BLE. The "No WiFi sleep" setting is therefore ineffective in
gamepad builds; WiFi latency may be slightly higher while the gamepad is
enabled.

See `platformio_override.ini` in the repository root for a ready-made
personal environment (`esp32s3dev_16MB_opi_gamepad`):

```ini
[env:esp32s3dev_16MB_opi_gamepad]
extends = env:esp32s3dev_16MB_opi
lib_deps = ${env:esp32s3dev_16MB_opi.lib_deps}
  h2zero/NimBLE-Arduino@^1.4.2
build_flags = ${env:esp32s3dev_16MB_opi.build_flags} -DMYGAMES_BLUEPAD32
```

Reference: NimBLE-Arduino (BLE HID host on NimBLE, compatible with the
Arduino core 2.0.x / ESP-IDF 4.4 used by WLED main):
https://github.com/h2zero/NimBLE-Arduino

The Xbox BLE input report layout (left stick in the first two 16-bit fields,
unsigned axes centered at 32768, hat switch logical range 1..8 with 0 =
neutral) was cross-checked against the author's working `BluePadPIO` ESP-IDF
HID host library (local balance-car project, not published).

## Installation

Add `MyGames` to `custom_usermods` of your build environment in
`platformio.ini` (or better, a personal `platformio_override.ini`):

```ini
custom_usermods = ${env:esp32dev.custom_usermods}
  MyGames
```

Note: changes to `platformio.ini` itself require maintainer approval when
submitted in a PR; see `platformio_override.ini.sample` for a template of
custom build environments.
