# ESP32 + CC1101 TermoWeb radio gateway — ATC Sun Ray RF 1800, paired from scratch

ESPHome configuration and firmware for a DIY 868 MHz radio gateway
([ha-termoweb](https://github.com/ha-termoweb/ha-termoweb)'s ESP32 + CC1101
design) that paired a **factory-fresh ATC Sun Ray RF 1800** radiator to Home
Assistant over the local radio — no vendor gateway, no cloud. To my knowledge
the first confirmed from-scratch **dialect-A** pairing on this stack.

The interesting part: the pairing protocol was never the problem. The blocker
was the CC1101 module's crystal running **~93 ppm low**, which put every
transmission ~81 kHz off-channel — inaudible to the heater at any distance,
while reception worked perfectly. Diagnosed with the CC1101's own `FREQEST`
register and fixed by recalibrating the carrier.

- **[TERMOWEB-PAIRING-REPORT.md](TERMOWEB-PAIRING-REPORT.md)** — full write-up
  (symptoms, diagnosis, the on-air adoption sequence, suggestions upstream).
- **[pairing-logs/00-decoded-frames.md](pairing-logs/00-decoded-frames.md)** —
  every key frame decoded, plus the FREQEST measurements.
- `termoweb_radio.h` — upstream 3.7-esp32 plus: per-frame `# freqest` offset
  diagnostic, module-specific carrier calibration (869 606 kHz — **re-measure
  for your own module**, nominal is 869 525), and an optional pairing assist.
- `termoweb-radio.yaml` — the ESPHome device config (secrets via
  `secrets.yaml`, not included).

If your heater "receives perfectly but ignores everything you transmit, even
point-blank": check your crystal before you blame the protocol.
