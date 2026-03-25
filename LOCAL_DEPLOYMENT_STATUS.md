# OTTO local deployment status

## Completed

- Cloned repositories:
  - `firmware` from `78/xiaozhi-esp32`
  - `server` from `xinnan-tech/xiaozhi-esp32-server`
  - `mqtt-gateway` from `xinnan-tech/xiaozhi-mqtt-gateway`
- Started full server stack in `server/main/xiaozhi-server` using `docker-compose_all.yml`.
- Created `server/main/xiaozhi-server/data/.config.yaml` and wired `manager-api.url`.
- Synced `manager-api.secret` from DB (`sys_params.server.secret`) so `xiaozhi-esp32-server` can boot.
- Configured management parameters in DB:
  - `server.websocket=ws://192.168.77.30:8000/xiaozhi/v1/`
  - `server.ota=http://192.168.77.30:8002/xiaozhi/ota/`
  - `server.mqtt_gateway=192.168.77.30:1884`
  - `server.mqtt_signature_key=OttoMqttKey_2026`
  - `server.udp_gateway=192.168.77.30:8885`
  - `server.mqtt_manager_api=192.168.77.30:8008`
- Verified OTA endpoint is reachable:
  - `curl http://127.0.0.1:8002/xiaozhi/ota/ ...`
  - response includes `websocket` and `mqtt` blocks.
- Prepared OTTO firmware defaults file:
  - `firmware/sdkconfig.defaults.otto-local`

## Running services

- Web console: `http://127.0.0.1:8002`
- WebSocket: `ws://192.168.77.30:8000/xiaozhi/v1/`
- OTA: `http://192.168.77.30:8002/xiaozhi/ota/`
- Local MQTT gateway process from this workspace:
  - MQTT: `192.168.77.30:1884`
  - UDP: `192.168.77.30:8885`
  - API: `192.168.77.30:8008`

## Flutter project migration

- Migrated full Flutter project from:
  - `/Users/lss/Desktop/AI_MCP/apps/ai-pet-provisioner`
- To:
  - `/Users/lss/Desktop/esp_project/esp32_s3/apps/ai-pet-provisioner`
- Validation:
  - `flutter pub get` succeeded
  - `flutter analyze` completed with warnings/info only (no blocking compile errors)
- Quick start:
  - `cd /Users/lss/Desktop/esp_project/esp32_s3/apps/ai-pet-provisioner`
  - `flutter pub get`
  - `flutter run -d macos` (or choose your target device)

## Blockers encountered

- Firmware build cannot complete with local ESP-IDF `5.4.x`.
- Project dependency resolver reports: `project depends on idf (>=5.5.2)`.

## Next command after upgrading ESP-IDF to >= 5.5.2

```bash
cd firmware
source "$HOME/esp-idf/export.sh"
rm -rf build sdkconfig
idf.py set-target esp32s3
idf.py -D SDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.defaults.esp32s3;sdkconfig.defaults.otto-local" build
idf.py -p /dev/tty.usbmodem* flash monitor
```
