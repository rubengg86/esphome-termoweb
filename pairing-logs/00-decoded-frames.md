# Decoded frames — ATC Sun Ray RF 1800 from-scratch pairing (2026-11-08)

All frames dialect A: sync `2DE5`, PN9-whitened on air, CRC-16/CCITT (poly 1021,
init 1D0F, xorout FFFF), logical layout `len net(2) src dst flags path(5) tag
payload crc(2)`, total on-air = len byte + 3. Station id `01`, network `1B30`.
Timestamps refer to `02-adoption-capture-869592kHz.txt` unless noted.

## Heater pairing announcement (sweep probe)

Air (dst=01 example, seen at -50..-68 dBm throughout):

    E787B8A6B6A13325565F4A9F79EFEA5F2ABAF72BE6E06BC1CBDFB0

Logical:

    18 0000 FF 01 00 FF 01 01 01 01 03 77 06000F00044330505036302D

net `0000` (unjoined), src `FF`, dst sweeps `01..41` (~210 ms/step), flags `00`,
path `FF <dst> 01 01 01`, tag `03`, payload `77` + 12-byte identity (tail ASCII
`C0PP60-`; HA serial `5036302d` = last 4 identity bytes). Note: tag-03/path
structure of the dialect-B announcement combined with the dialect-A
77+identity payload.

## Station auto-ack to the dst=01 probe

Air `FA9C8858482120EF` → logical `05 1B30 01 FF 80` (acker 01, sender FF).

## Assignment (station → heater), on-air 23:04:03.983

Air `F39C885848A1CDDB575E4B980C6321` → logical:

    0C 1B30 01 FF 00 01 FF 00 00 00 04 02

tag `04`, payload = new node id `02`, dst FF, path `01 FF 00 00 00` —
byte-identical in structure to the stock-gateway capture in ha-termoweb-local
(`0c 1b30 01 ff 00 01ff000000 04 04`). Transmitted ~150 ms after the heater's
dst=01 probe; accepted (no sub-16 ms timing needed).

## Adoption sequence (heater → station)

1. 23:04:04.004 — link ack **from FF on the new network**:
   air `FA9C88A6B621E872` → `05 1B30 FF 01 80`
2. 23:04:04.037 — empty **tag-84** from the new id:
   air `F49C885BB6A1CE25565F4A186785` → `0B 1B30 02 01 00 02 01 01 01 01 84`
3. 23:04:04.394 (repeated ~6–9 s: 04:10.259, 04:19.810, 04:27.965, 04:35.894) —
   **registration**: air `F39C885BB6A1CE25565F4A9C5E21CB` →
   `0C 1B30 02 01 00 02 01 01 01 01 00 50`
   (heater retries the `50` registration awaiting a station reply; keeps the
   binding regardless — paired icon on the display, answers on id 2.)

Station auto-acked each: air `FA9C8858B5214572` → `05 1B30 01 02 80`.

## Normal paired traffic after trim (file 03, ~16 s cycle)

- Heater status report: air `E49C885BB6A1CE25...` → 30-byte frame from 02 to 01.
- Station auto-ack `05 1B30 01 02 80`.
- HA poll: air `F29C8858B5A1CD26575E4B9C59BC253A` (16 bytes, to id 02).
- Heater ack of the poll: air `FA9C885BB6214971` → `05 1B30 02 01 80`.

## FREQEST (frequency-offset) measurements

LSB = f_xosc/2^14 ≈ 1.587 kHz, signed; FOC clamp ≈ ±BW/4 (±42 LSB at 270.8 kHz).

| Carrier dial (kHz) | FREQEST on heater frames | Implied offset |
|---|---|---|
| 869525 (nominal)   | **+42 constant, 65/65 frames** (clamped) | ≥ +67 kHz |
| 869592             | **+9 constant** | +14.3 kHz residual → true ≈ +81 kHz (~93 ppm) |
| 869606 (final)     | **0** | centred; adoption + paired traffic verified |

Consequence of the offset at the nominal dial: gateway TX ~81 kHz off-channel —
the heater never received anything (no acks ever), at any distance, while
gateway RX was unaffected (270.8 kHz RX filter). This was the sole blocker;
protocol, frame contents, network id and timing were all correct throughout.

Note before sharing raw captures: the boot/config sections of the .txt files
include local network details (WiFi SSID, IPs, MAC) — trim if attaching
publicly.
