# Meshtastic MQTT Manual

> Compressed reference: all transport modes (Direct, Client Proxy), payload formats (Protobuf, JSON), topic structure, broker bridging, board support matrix, and operational pitfalls. Based on firmware source analysis (meshtastic/firmware master, circa 2.7.x).
> 
> Live STE (Simplified Technical English) manual: https://gwyntel.github.io/meshtastic-mqtt-manual/  
> Raw URL for AI agents: https://raw.githubusercontent.com/gwyntel/meshtastic-mqtt-manual/master/manual.md

---

## MQTT Modes Overview

Meshtastic firmware has **two transport modes** and **two payload formats**:

**Transport:**
- **Direct** — radio connects to broker itself (needs WiFi/ETH). `proxy_to_client_enabled: false`
- **Client Proxy** — phone app forwards MQTT for radio. Radio MQTT client stays dormant. `proxy_to_client_enabled: true`

**Payload format:**
- **Protobuf (binary)** — encrypted or unencrypted ServiceEnvelope bytes. Default.
- **JSON** — human-readable JSON. `json_enabled: true`. NOT on nRF52 (unless `NRF52_USE_JSON` defined).

## Config Settings (CLI)

```
meshtastic --set mqtt.enabled true
meshtastic --set mqtt.address mqtt.gwyn.tel
meshtastic --set mqtt.username meshdev
meshtastic --set mqtt.password '<pass>'
meshtastic --set mqtt.root 'msh/US/bayarea'
meshtastic --set mqtt.tls_enabled true       # port 8883
meshtastic --set mqtt.encryption_enabled true # encrypt MQTT transport layer
meshtastic --set mqtt.json_enabled true       # enable JSON uplink/downlink
meshtastic --set mqtt.proxy_to_client_enabled false  # radio connects directly
meshtastic --set mqtt.map_reporting_enabled false    # node visible on maps
```

Chain commands to avoid reboot between each:
```
meshtastic --set mqtt.enabled true --set mqtt.json_enabled true
```

## Channel Uplink/Downlink

Each channel has independent MQTT settings:
- `uplink_enabled: true` — channel messages forwarded TO broker
- `downlink_enabled: true` — broker messages injected TO radio

Set per-channel:
```
meshtastic --host <ip> --ch-index 0 --ch-set uplink_enabled true
meshtastic --host <ip> --ch-index 0 --ch-set downlink_enabled false
```

CLI uses snake_case: `downlink_enabled` NOT `downlinkEnabled`.

## Topic Structure

Firmware constructs topics from `mqtt.root` setting:

```
{root}/2/c/{channelName}/{nodeId}   — protobuf uplink
{root}/2/json/{channelName}/{nodeId} — JSON uplink
{root}/2/c/{channelName}/+          — protobuf downlink subscribe
{root}/2/json/{channelName}/+       — JSON downlink subscribe
{root}/2/map/                       — map reports
{root}/2/c/PKI/{nodeId}             — PKI-encrypted uplink
{root}/2/c/PKI/+                    — PKI downlink subscribe
```

**IMPORTANT:** Binary segment is `/2/c/` (crypt), NOT `/2/e/`.

Channel name in topic = `channels.getGlobalId(chIndex)`:
- Unnamed primary (index 0) → modem preset name (e.g. `MediumFast`)
- Named channels → channel name string (e.g. `test`, `mqtt`)
- PKI-encrypted → `PKI`

Root defaults to `msh` if empty. Community root example: `msh/US/bayarea`.

## JSON Downlink (Sending TO mesh via MQTT)

**Requirements:**
1. `mqtt.json_enabled: true`
2. Channel MUST be named `"mqtt"` (case-insensitive, firmware check via `strncasecmp`)
3. Channel must have `downlink_enabled: true`
4. Publish to `{root}/2/json/mqtt/{gateway_id}`

**JSON envelope schema:**
```json
{
  "from": 3254555032,
  "type": "sendtext",
  "payload": "Hello mesh!",
  "channel": 2,
  "to": 4294967295,
  "hopLimit": 3,
  "sender": "!gw01"
}
```

**Validation rules (firmware `isValidJsonEnvelope`):**
- `from` REQUIRED — must equal node's own decimal number (`!c1fc9198` = 3254555032)
- `type` REQUIRED — `"sendtext"` or `"sendposition"`
- `payload` REQUIRED — string (sendtext) or object (sendposition)
- `channel` optional — channel index 0-7
- `to` optional — destination node. 4294967295 = broadcast
- `hopLimit` optional — if present must be number
- `sender` optional — gateway ID. If present, must NOT equal node's own ID (loop prevention)

**sendposition payload:**
```json
{
  "from": 3254555032,
  "type": "sendposition",
  "payload": {
    "latitude_i": 377743000,
    "longitude_i": -1224194000,
    "altitude": 10,
    "time": 1689235200
  },
  "channel": 2,
  "to": 4294967295
}
```

## JSON Uplink (Node → Broker)

```json
{
  "id": 1234567,
  "timestamp": 1689235200,
  "to": 4294967295,
  "from": 3254555032,
  "channel": 2,
  "type": "text",
  "sender": "!c1fc9198",
  "rssi": -45,
  "snr": 9.5,
  "hops_away": 1,
  "hop_start": 3,
  "payload": {
    "text": "Hello from mesh!"
  }
}
```

If text payload is valid JSON, firmware puts raw JSON in `payload` instead of `{"text": "..."}`.

## Protobuf (Binary) Format

ServiceEnvelope (published as raw bytes, NOT base64):
```protobuf
message ServiceEnvelope {
  MeshPacket packet     = 1;
  string channel_id     = 2;
  string gateway_id     = 3;
}
```

`encryption_enabled` controls transport:
- `false` (default) — decoded MeshPackets, readable
- `true` — encrypted MeshPackets, need channel AES key

Data.payload limited to ~240 bytes. MQTT buffer: 1024 bytes.

## Node ID Format

```
!XXXXXXXX  (8 hex chars = 32-bit unsigned number)
```

Derived from last 4 bytes of MAC address. Node number = `int(nodeId[-8:], 16)`. Broadcast = `!ffffffff` = 4294967295.

## Board Support Notes

| Board | WiFi | Ethernet | JSON | TLS | Notes |
|------|------|----------|------|-----|-------|
| ESP32 (XIAO S3, RAK, Heltec, T-Beam, etc.) | ✓ | varies | ✓ | ✓ (MQTT_SUPPORTS_TLS) | Full MQTT support |
| nRF52 (RAK4631) | ✗ (no WiFi) | ✗ | ✗ unless `NRF52_USE_JSON` | ✗ | Must use client proxy — no direct network |
| Portduino (Linux SIM) | ✓ (sim) | ✓ (sim) | ✓ | ✓ | For device simulator |

**nRF52 limitations:**
- No WiFi → cannot connect to broker directly
- `proxy_to_client_enabled` MUST be `true` (firmware rejects config with error: "proxy_to_client_enabled must be enabled on nodes that do not have a network")
- JSON not supported unless `NRF52_USE_JSON` compile flag set (issue #2804)
- Phone app (Android/iOS) handles all MQTT traffic

**ESP32 notes:**
- XIAO S3: BT auto-disabled when WiFi active (shared radio)
- TLS uses `setInsecure()` — no cert validation in firmware
- `pubSub.setBufferSize(1024, 1024)` — 1KB max MQTT payload
- Reconnect: 5 attempts, then triggers WiFi reconnect
- TCP connect is "EXPENSIVE" (firmware comment) — retry interval 30 sec on fail

**HAS_NETWORKING flag:**
- ESP32 with WiFi: yes
- ESP32 with Ethernet (WS5500, CH390D): yes
- nRF52: no
- Portduino: yes

## Client Proxy Mode

When `proxy_to_client_enabled: true`:
- Radio does NOT connect to broker
- Phone app uses radio's credentials to connect
- Client ID: `MeshtasticAppleMqttProxy-!nodeId` or `MeshtasticAndroidMqttProxy-!nodeId`
- Radio's own MQTT client stays dormant until setting disabled
- `wantsLink()` returns true (proxy counts as "connected")
- Firmware still constructs topics, still publishes via proxy message queue

When `proxy_to_client_enabled: false`:
- Radio connects directly using WiFi/ETH
- Client ID: `!nodeId` (e.g. `!c1fc9198`)
- Subscribe + publish happen in firmware
- `runOnce()` loop: 200ms when connected, 30sec retry on fail

**CHECK THIS FIRST** when node won't connect: `proxyToClientEnabled` in `--info` output.

## TLS

- `tls_enabled: true` → port 8883, `tls_enabled: false` → port 1883
- Firmware uses `setInsecure()` — no certificate validation
- `MQTT_SUPPORTS_TLS` compile flag — if not set, firmware rejects `tls_enabled: true` with error
- nRF52: no TLS support
- ESP32: TLS supported

## Mosquitto Broker Setup

**Bridge to community broker:**
```
connection baymesh-bridge
address mqtt.bayme.sh:1883
topic msh/US/bayarea/2/c/MediumFast/# both 0
topic msh/US/bayarea/2/c/test/# both 0
topic msh/US/bayarea/2/json/# both 0
clientid gwyntel-bridge
remote_username meshdev
remote_password large4cats
start_type automatic
```

**Per-channel bridge exclusion:**
- Mosquitto `topic` directive supports wildcards (`+`, `#`) but NOT exclusion
- Replace catch-all `#` with explicit per-channel lines
- Unlisted channels stay local to broker

**Private channel example:**
```
# Forward MediumFast + test + JSON, but NOT mqtt or gwn channels:
topic msh/US/bayarea/2/c/MediumFast/# both 0
topic msh/US/bayarea/2/c/test/# both 0
topic msh/US/bayarea/2/json/# both 0
```

## Community Broker Defaults

`mqtt.meshtastic.org`:
- Username: `meshdev`
- Password: `large4cats`
- Root: `msh` (global)

Community roots: `msh/US/bayarea`, `msh/US/sacramento`, etc.
These are shared community credentials, NOT IoT default passwords.

Root topic case-sensitive. `msh/US/bayarea` ← capitals matter.

## Dedup (QoS 1 + Bridge Echo)

paho-mqtt QoS 1 = at-least-once. Mosquitto bridge can echo messages back. Same packet `id` arrives multiple times.

Fix: dedup by packet `id` field in JSON uplink. 200-entry cache sufficient.

## Map Reports

`map_reporting_enabled: true` → node publishes to `{root}/2/map/` periodically.
- Includes: name, ID, position (with precision), hardware model, role, firmware version, LoRa region, modem preset, primary channel name
- Default position precision: ~1459m deviation
- Default publish interval: 3600 sec (1 hour, minimum)
- Unencrypted — visible on public maps
- Available from firmware 2.3.2+

## Pitfalls

### proxyToClientEnabled after reboot
Setting can revert to `true` after `--reboot` or config writes. Always verify:
```
meshtastic --host <ip> --info | grep proxyToClient
```

### CLI socket error after config write
`OSError: [Errno 9] Bad file descriptor` — known CLI bug. Config DOES get applied. Verify with `--info`.

### PSK via CLI
`--ch-set psk "TQ=="` fails: `expected bytes, str found`. Use Python API for custom PSKs.

### --ch-set-url side effects
Resets `network.wifi_ssid` and `network.wifi_psk` to empty. Re-apply WiFi after channel import. May not import all channels — verify.

### hopLimit reset
Can revert to 7 (firmware default) after config writes. Always re-set.

### "mqtt" channel name
JSON downlink ONLY works on channel named "mqtt" (case-insensitive). Firmware checks via `strncasecmp` against `Channels::mqttChannel`. No other channel name accepts JSON downlink.

### Root topic mismatch
Node `mqtt.root` must match community broker topic hierarchy. `msh/US` ≠ `msh/US/bayarea` — traffic silently goes nowhere if mismatch.

### Don't mess with radio MQTT config without asking
NEVER change `mqtt.address`, `mqtt.tls_enabled`, `mqtt.port`, `mqtt.root` without explicit permission. Check `proxyToClientEnabled` FIRST.

## PortNum Reference

| Name | Value | Payload |
|------|-------|---------|
| TEXT_MESSAGE_APP | 1 | UTF-8 plaintext |
| POSITION_APP | 3 | protobuf (Position) |
| NODEINFO_APP | 4 | protobuf (User) |
| ROUTING_APP | 5 | protobuf (Routing) |
| TELEMETRY_APP | 67 | protobuf (Telemetry) |
| TRACEROUTE_APP | 70 | protobuf (RouteDiscovery) |
| NEIGHBORINFO_APP | 71 | protobuf (NeighborInfo) |
| MAP_REPORT_APP | 73 | protobuf (MapReport) |

TEXT_MESSAGE_APP encoding: `Data.payload` = raw UTF-8 bytes, no protobuf wrapping.
