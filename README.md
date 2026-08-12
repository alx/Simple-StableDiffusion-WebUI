# SSD WebUI (Simple Stable Diffusion WebUI)

A minimal web interface for driving a [stable-diffusion.cpp](https://github.com/leejet/stable-diffusion.cpp)
server (`sd-server` mode). Single-file Python script, **zero external dependency**
(standard library only, Python 3.9+), all configuration done through command-line
options.

It talks to the sd.cpp server through its compatible API
(`/sdapi/v1/txt2img` and `/sdapi/v1/img2img`).

## Features

- **Text2Image** tab: classic prompt vers image generation (size, steps, CFG scale,
  seed, batch size, sampler, scheduler).
- **Image2Image** tab: upload/drag-and-drop a source image and edit it with a
  dedicated parameter set (denoising strength + the usual sampling controls).
- **Gallery** tab: browse images that were explicitly saved to disk.
- Per-image **"Open in new tab"** and **"Save image"** links on every
  generated thumbnail.
- **Privacy by default**: generated images are returned to the browser only
  (as base64 data-URLs) and are **never written to the server's disk** unless
  you explicitly tick "Save this generation to disk" for that specific
  generation. An admin flag (`--disable-save`) can remove that possibility
  entirely for the whole instance.
- No database, no build step, no framework: one `.py` file you can read
  top to bottom.

## Requirements

- Python 3.9 or newer (standard library only, nothing to install with `pip`).
- A running `stable-diffusion.cpp` server, started with `sd-server` /
  HTTP mode enabled and reachable over HTTP.

## Quick start

1. Start the web interface, pointing it at that server:

   ```bash
   python3 ssd_webui.py --sd-url http://127.0.0.1:8082 --port 8083 --open-browser
   ```

2. Open `http://127.0.0.1:8083` in your browser (it opens automatically with
   `--open-browser`):
   - **Text2Image**: write a prompt, tune parameters, click *Generate*.
   - **Image2Image**: drop a source image, write a prompt, tune the
     denoising strength, click *Generate*.
   - **Gallery**: shows everything that was explicitly saved to disk.

## Command-line options

| Option           | Default                  | Description                                                                 |
| ----------------- | ------------------------- | ----------------------------------------------------------------------------- |
| `--sd-url`         | `http://127.0.0.1:8082`   | Base URL of the stable-diffusion.cpp server                                   |
| `--host`           | `127.0.0.1`                | Listen address for the web interface                                          |
| `--port`           | `8083`                     | Listen port for the web interface                                             |
| `--output-dir`     | `./outputs`                 | Folder used to save generated images (only when saving is requested)          |
| `--timeout`        | `300`                       | Timeout in seconds for calls to the sd.cpp server                             |
| `--open-browser`   | off                         | Automatically open the default browser on startup                             |
| `--disable-save`   | off                         | Forbid any image from ever being written to disk (hides the save checkbox, gallery and `/files` are disabled) |

Run `python3 ssd_webui.py --help` at any time for the full list.

### Example: strict "no persistence" deployment

```bash
python3 ssd_webui.py --sd-url http://127.0.0.1:8082 --host 0.0.0.0 --port 8083 --disable-save
```

In this mode, the save checkbox and the Gallery tab disappear, and no
generated image can ever be written to the server's filesystem — everything
stays transient, in the HTTP response sent to the browser. (useful to
run with a systemd service)

## Running as a systemd service

An example unit file, `ssd_webui.service`, is provided to run SSD WebUI as a
background service (listening on port **8083** in the example) that starts
on boot and restarts automatically if it crashes.

1. Create a dedicated system user and install the app:

   ```bash
   sudo useradd --system --no-create-home --shell /usr/sbin/nologin ssd-webui
   sudo mkdir -p /opt/ssd-webui/outputs
   sudo cp ssd_webui.py /opt/ssd-webui/
   sudo chown -R ssd-webui:ssd-webui /opt/ssd-webui
   ```

2. Copy the unit file and adjust it to your setup :

   ```bash
   sudo cp ssd_webui.service /etc/systemd/system/ssd_webui.service
   sudo systemctl daemon-reload
   ```

3. Enable and start the service:

   ```bash
   sudo systemctl enable --now ssd_webui.service
   ```

4. Check its status and logs:

   ```bash
   sudo systemctl status ssd_webui.service
   sudo journalctl -u ssd_webui.service -f
   ```

The service listens on `127.0.0.1:8083` by default in http. Put
a reverse proxy (nginx/Caddy) with TLS and authentication in front of it if
you need to expose it beyond localhost, since SSD WebUI itself has no
built-in authentication. 

To stop or restart the service:

```bash
sudo systemctl stop ssd_webui.service
sudo systemctl restart ssd_webui.service
```

