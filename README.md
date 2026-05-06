# ReSpeaker XVF3800

## Home-Assistant integration

1. install [ESPHome](https://www.home-assistant.io/integrations/esphome) integration.
2. Add the reSpeaker (should be detected on your wifi)
3. Configure the "Voice assistants" from Settings menu then "Add assistant"
4. Configure the device (wake-word, volume etc...) and hook it to the voice-assistant by going to its configuration menu

### Using custom STT

Install [Wyoming Protocol](https://www.home-assistant.io/integrations/wyoming) integration.

1. Deploy a [hass-piper](https://www.home-assistant.io/integrations/piper/) container (docker-compose example):

    ```yaml
    services:
      hass-piper:
        container_name: hass-piper
        image: rhasspy/wyoming-piper
        volumes:
          - /volume1/docker/home-assistant/piper/data:/data
        command: --voice fr_FR-gilles-low
          --update-voices
        ports:
          - 127.0.0.1:10200:10200
          - 127.0.0.1:15000:5000
        restart: unless-stopped
        networks: [default]
    ```

2. Deploy a custom *hass-sherpa-onnyx* container (docker-compose example):

    ```yaml
    services:
      hass-sherpa-onnyx:
        container_name: hass-sherpa-onnyx
        build:
          pull: true
          no_cache: true
          context: https://github.com/fargies/sherpa-onnx-tts-stt.git#wip-new-models
          args:
            LANGUAGE: "fr-FR"
            PREFETCH: 1
        environment:
          STT_USE_ONLINE_PROCESSING: 1
        ports:
          - 10401:10400
        restart: unless-stopped
    ```

3. Configure both in Wyoming Protocol integration

## Enclosure

STL files for the enclosure are here: [enclosure](./enclosure)

## Firmware

### XMOS Update

Firmwares may be found here: [https://github.com/respeaker/reSpeaker_XVF3800_USB_4MIC_ARRAY]
Documentation on how to flash the device may be found here: [https://wiki.seeedstudio.com/respeaker_xvf3800_xiao_getting_started/](respeaker_xvf3800_xiao_getting_started)

**Note:**: pick the "i2s_master" firmwares.

```bash
# Install dfu-util
yay -S dfu-util
apt-get install dfu-util

sudo dfu-util -l

# To enter safe-mode: press and hold mute button when powering the device

# Flash the firmware provided by *formatBCE*
sudo dfu-util -R -e -a 1 -D ./firmware/application_xvf3800_inthost-lr48-sqr-i2c-v*-release.bin
```

**Note:** I2C firmware doesn't support USB DFU, to enter safe-mode and enable
USB DFU press and hold mute button when powering the device (see <https://wiki.seeedstudio.com/respeaker_xvf3800_introduction/#safe-mode>)

### ESP32 Update

Using podman (also works with Docker):

```bash
podman pull ghcr.io/esphome/esphome

# create secrets.yaml file to configure wifi
echo '
wifi_ssid: "xxx"
wifi_password: "xxx"
ota_password: "xxx"
' > firmware/config/secrets.yaml

# Compile and flash firmware
podman run --rm --privileged \
    -v "${PWD}":/config \
    -v "${PWD}/firmware/root":/root \
    --device=/dev/ttyACM0 \
    -it ghcr.io/esphome/esphome run respeaker-xvf-satellite-example.yaml
```

**Note:** to update Wifi configuration, not re-flashing the firmware, the
following sites may be used (**using Chrome** or **Chromium**,
connecting the ESP32 device on USB):

- [web.esphome.io](https://web.esphome.io/)
- [improv-wifi.com](https://www.improv-wifi.com/)

### Debugging

To get a console:

```bash
# Direct console
sudo minicom -D /dev/ttyACM0

# Using ESPHome tools
podman run --rm --privileged \
    -v "${PWD}":/config \
    -v "${PWD}/firmware/root":/root \
    --device=/dev/ttyACM0 \
    -it ghcr.io/esphome/esphome logs respeaker.yaml

# One can also connect to https://web.esphome.io/
```

#### Direct control/tests

```bash
$ python -m venv venv
$ source venv/bin/activate

$ python -m asyncio

import aioesphomeapi

api = aioesphomeapi.APIClient("192..168.3.33", 6053, "")
await api.connect()
entities = await api.list_entities_services()
key = next(filter(lambda x: x.object_id == 'media_player', entities[0])).key

# from https://filesamples.com/formats/flac
api.media_player_command(key, command=0, media_url="https://dl.espressif.com/dl/audio/gs-16b-2c-44100hz.flac")
api.media_player_command(key, command=0, media_url="https://dl.espressif.com/dl/audio/ff-16b-2c-44100hz.flac")
api.media_player_command(key, command=0, media_url="https://filesamples.com/samples/audio/flac/sample4.flac")
```

## External Links

- [https://wiki.seeedstudio.com/respeaker_xvf3800_xiao_getting_started/]
- [https://wiki.seeedstudio.com/respeaker_xvf3800_xiao_home_assistant/]
