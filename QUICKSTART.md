# Quick Start - Driving the UIAutomator2 Server from Python

A light JSON-RPC server that runs on the device and lets you drive the UI from
your PC. Verified working on a Pixel 9a (Android SDK 37) against a Unity game.

## 1. Build the APK (uses Unity's bundled Android toolchain - no Android Studio)

```bash
./build.sh            # debug APK (default)
./build.sh release    # release APK
UNITY_VERSION=6000.0.55f1 ./build.sh   # force a specific Unity editor
```

Output: `app/build/outputs/apk/debug/app-debug.apk`

`build.sh` auto-detects a Unity editor's JDK + SDK + build-tools (prefers
`6000.3.9f1`) and writes `local.properties` for you.

## 2. Deploy & start the server on the device

```bash
adb push app/build/outputs/apk/debug/app-debug.apk /data/local/tmp/udt/atx-uia2.jar
adb forward tcp:9008 tcp:9008
adb shell CLASSPATH=/data/local/tmp/udt/atx-uia2.jar app_process / com.wetest.uia2.Main
```

The last command blocks (runs the server in the foreground). To run it in the
background from a script, launch it with `nohup ... &` and poll `/ping`.

> The server dies if the device sleeps or the adb shell session ends. Re-launch
> it and re-run `adb forward` if `/ping` stops returning `pong`.

## 3. Smoke test

```bash
curl http://127.0.0.1:9008/ping                                   # -> pong
curl -X POST -d '{"jsonrpc":"2.0","id":"1","method":"deviceInfo","params":{}}' \
     http://127.0.0.1:9008/jsonrpc/0
```

Or use the bundled interactive tester (auto-runs `adb forward`):

```bash
python3 tools/test_jsonrpc.py              # interactive menu
python3 tools/test_jsonrpc.py ping
python3 tools/test_jsonrpc.py click 540 1212
```

## 4. Protocol basics (IMPORTANT)

- **Endpoint:** `POST http://127.0.0.1:9008/jsonrpc/0`
- **Params MUST be a positional array**, NOT a named object:
  - Correct:  `"params":[540, 1212]`
  - Wrong:    `"params":{"x":540,"y":1212}`  -> "method parameters invalid"
- **Screenshot:** use the JSON-RPC `takeScreenshot` method (returns base64 JPEG).
  The `GET /screenshot/0` endpoint was flaky on the test device.

### Minimal Python client

```python
import json, base64, urllib.request

URL = "http://127.0.0.1:9008/jsonrpc/0"

def rpc(method, *params):
    body = json.dumps({"jsonrpc": "2.0", "id": "1",
                       "method": method, "params": list(params)}).encode()
    req = urllib.request.Request(URL, data=body,
                                 headers={"Content-Type": "application/json"})
    resp = json.loads(urllib.request.urlopen(req, timeout=30).read())
    if "error" in resp:
        raise RuntimeError(resp["error"])
    return resp["result"]

info = rpc("deviceInfo")
W, H = info["displayWidth"], info["displayHeight"]

# normalized tap (0..1) -> pixels, clamped to screen
def tap_normalized(nx, ny):
    x = min(round(nx * W), W - 1)
    y = min(round(ny * H), H - 1)
    return rpc("click", x, y)

def screenshot(path="shot.jpg", scale=1.0, quality=85):
    b64 = rpc("takeScreenshot", scale, quality)
    open(path, "wb").write(base64.b64decode(b64))
    return path
```

## 5. Common methods

| Call | Params (positional) | Notes |
|------|---------------------|-------|
| `ping` (GET /ping) | - | liveness, returns `pong` |
| `deviceInfo` | - | resolution, rotation, current package, sdkInt |
| `click` | `x, y` | pixel tap |
| `click` | `x, y, ms` | long-press for `ms` milliseconds |
| `swipe` | `startX, startY, endX, endY, steps` | ~5ms per step |
| `takeScreenshot` | `scale, quality` | base64 JPEG |
| `dumpWindowHierarchy` | `compressed(bool)` | XML UI tree |
| `pressKey` | `"back"` / `"home"` / etc. | named keys |
| `wakeUp` | - | wake screen |

Full catalog with every method + example values: `tools/methods.json`.

## 6. Coordinate space (for normalization)

Verified identical on the test device:

- `deviceInfo().displayWidth/Height` = true full-screen pixels
- `click` / `swipe` coordinates = those same pixels
- `takeScreenshot` (scale=1.0) = captured at those same dimensions

So: `pixelX = round(normX * displayWidth)` (clamp to `displayWidth - 1`).

Resolution is **rotation-adjusted** - re-fetch `deviceInfo()` after the app
rotates (e.g. a game forcing landscape flips 1080x2424 -> 2424x1080).

## 7. Two kinds of screens

1. **Native Android dialogs** (permission prompts, `com.google.android.
   permissioncontroller`): have a real accessibility tree. Click by selector:

   ```python
   # Allow a notification permission dialog (robust, not coordinate-based)
   MASK_RESOURCEID = 0x200000
   rpc("click", {"mask": MASK_RESOURCEID,
                 "resourceId": "com.android.permissioncontroller:id/permission_allow_button"})
   ```

   Selector fields need a **bitmask** telling the server which are active
   (combine with bitwise OR):
   `MASK_TEXT=0x01`, `MASK_TEXTCONTAINS=0x02`, `MASK_CLASSNAME=0x10`,
   `MASK_DESCRIPTION=0x40`, `MASK_RESOURCEID=0x200000`.

2. **Unity game canvas** (the game's own UI): has **no accessibility tree**
   (`dumpWindowHierarchy` shows only bare FrameLayouts). You must use
   **coordinate taps + screenshot verification** - screenshot, locate the
   target visually/by image match, then `click(pixelX, pixelY)`.

## 8. Launching an app

The server's `launchApp` / `executeShellCommand` fail on this device (SDK 37
security + headless shell restrictions). Launch host-side via adb instead:

```bash
adb shell am start -n com.scopely.wwedomination/com.scopely.unity.ScopelyUnityActivity
# find the launchable activity:
adb shell cmd package resolve-activity --brief <package>
```

Then use the server for taps / swipes / screenshots / deviceInfo.
