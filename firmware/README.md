# Precompiled Tasmota with NeoPool

This folder contains [Tasmota](https://github.com/arendst/Tasmota) binary files including the [driver for Sugar Valley NeoPool Controller](https://tasmota.github.io/docs/NeoPool/).

The binaries are based on [Tasmota v15.1.0 Stella](https://github.com/arendst/Tasmota/tree/v15.1.0).

## Files for initial flash

These files are for initial flashing an ESP only. Do not use these for OTA or file upload updates.

To **initial flash** an ESP32/ESP8266 it is recommended using [ESP_Flasher](https://github.com/Jason2866/ESP_Flasher), use one of the files listed below that corresponds to your ESP chip type:

- [NeoPool_ESP32_tasmota32.factory.bin](https://github.com/alexdelprete/HA-NeoPool-MQTT/raw/refs/heads/main/firmware/NeoPool_ESP32_tasmota32.factory.bin) - all ESP32 chip types (like M5Stack [Atom Lite](https://docs.m5stack.com/en/core/ATOM%20Lite))
- [NeoPool_ESP32S3_tasmota32s3.factory.bin](https://github.com/alexdelprete/HA-NeoPool-MQTT/raw/refs/heads/main/firmware/NeoPool_ESP32S3_tasmota32s3.factory.bin) - all ESP32 S3 chip types (like M5Stack [Atom S3](https://docs.m5stack.com/en/core/AtomS3) or [Atom S3 Lite](https://docs.m5stack.com/en/core/AtomS3%20Lite))
- [NeoPool_ESP32C3_tasmota32c3.factory.bin](https://github.com/alexdelprete/HA-NeoPool-MQTT/raw/refs/heads/main/firmware/NeoPool_ESP32C3_tasmota32c3.factory.bin) - all ESP32 C3 chip types
- [NeoPool_ESP8266_tasmota.bin](https://github.com/alexdelprete/HA-NeoPool-MQTT/raw/refs/heads/main/firmware/NeoPool_ESP8266_tasmota.bin) - all ESP8266 chip types

## Files for updating Tasmota

To update an existing Tasmota device via WebUI **Firmware Upgrade**/[**Upgrade by file upload**](https://tasmota.github.io/docs/Upgrading/#upgrade-by-file-upload), use one of the files listed below that corresponds to your ESP chip type:

- [NeoPool_ESP32_tasmota32.bin](https://github.com/alexdelprete/HA-NeoPool-MQTT/raw/refs/heads/main/firmware/NeoPool_ESP32_tasmota32.bin) - all ESP32 chip types (like M5Stack [Atom Lite](https://docs.m5stack.com/en/core/ATOM%20Lite))
- [NeoPool_ESP32S3_tasmota32s3.bin](https://github.com/alexdelprete/HA-NeoPool-MQTT/raw/refs/heads/main/firmware/NeoPool_ESP32S3_tasmota32s3.bin) - all ESP32 S3 chip types (like M5Stack [Atom S3](https://docs.m5stack.com/en/core/AtomS3) or [Atom S3 Lite](https://docs.m5stack.com/en/core/AtomS3%20Lite))
- [NeoPool_ESP32C3_tasmota32c3.bin](https://github.com/alexdelprete/HA-NeoPool-MQTT/raw/refs/heads/main/firmware/NeoPool_ESP32C3_tasmota32c3.bin) - all ESP32 C3 chip types
- [NeoPool_ESP8266_tasmota.bin.gz](https://github.com/alexdelprete/HA-NeoPool-MQTT/raw/refs/heads/main/firmware/NeoPool_ESP8266_tasmota.bin.gz) - all ESP8266 chip types
