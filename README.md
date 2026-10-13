# luce-ptp

Cameras over PTP, the Picture Transfer Protocol (ISO 15740), for Luce/Base. Plug a
camera in by USB and a program can find it, read what it is, list the photos on its
cards and download them.

The package speaks the protocol itself: it builds the operation containers, reads the
DeviceInfo, StorageInfo and ObjectInfo datasets, and downloads objects in pieces with
GetPartialObject (whole with GetObject when the camera lacks it), writing beside the
destination and renaming into place.

How the containers reach the camera is a transport's job:

- **macOS** — the system's camera service takes every PTP camera when it is plugged in,
  so this package goes through ImageCaptureCore: an `ICDeviceBrowser` reports cameras,
  a session is opened on each, and the containers pass through untouched with
  `requestSendPTPCommand:outData:completion:`. Checked with a Leica M11-D
  (firmware 2.6.4): DeviceInfo, storages, 26 DNGs listed and downloaded in pieces.
- **virtual** — `ptp.virtual_camera(folder)` answers PTP from the files in a folder, for
  tests and for programs that want a camera without one.
- **Linux and Windows** are not written yet (usbfs bulk transfers, and WPD's PTP
  pass-through); a watcher there reports `available() == false`.

```
let watching = try ptp.watch()            # on the main thread
# … once watcher.ready(0), on a thread of your own:
var camera = try ptp.Camera.open(try watcher.connect(0))
defer camera.close()
var handles: List[u32]
try camera.handles(&handles)
for handle in handles.view():
    let info = try camera.object_info(handle, &pool)
    if not info.is_folder():
        try camera.download(handle, info.size, target)
```

On macOS everything ImageCaptureCore says arrives on the main thread, so the watcher
lives there (the window's event loop runs the run loop it needs) and a camera's
transactions run on another thread, each waiting for the main thread to carry it.

`tests/watch` runs the watcher as a program would; with a camera plugged in it reads the
first camera and, given `--download`, fetches its newest photo:

```
luc test && ./build/luc-test/watch/watch --download
```
