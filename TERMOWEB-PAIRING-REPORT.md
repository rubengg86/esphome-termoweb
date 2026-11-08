# Confirmed: from-scratch dialect-A pairing works — field failures can be CC1101 crystal offset (full on-air adoption capture, ATC Sun Ray RF 1800)

## Summary

I paired a **factory-fresh ATC Sun Ray RF 1800** (never bonded to any vendor gateway) from scratch, over the radio, using the **ESP32 + CC1101 gateway** and this integration — and captured the complete adoption sequence on air. As far as your docs record (`radio_protocol.md` §7 marks dialect-A pairing *"documented by ha-termoweb-local, not yet tried here"*), this is the first confirmed from-scratch dialect-A join.

Two results worth your attention:

1. **Your pairing implementation is essentially correct as shipped.** The assignment frame, the network handling, and the reactive answer-the-announcement flow all work. The heater's pairing listen window is generous: it adopted from an assignment that arrived **~150 ms after its probe**, so even WiFi-gateway latency is fine.
2. **The failure mode that made it look broken was hardware, and it is nasty:** my CC1101 module's crystal runs **~93 ppm low**. Every transmission left the ESP ~80 kHz off-channel — **inaudible to the heater at any distance, including point-blank** — while receive worked perfectly (the 270.8 kHz RX filter forgives the offset). RX perfect + TX reported OK + heater totally unresponsive = undiagnosable with the current firmware. The CC1101's own `FREQEST` register diagnoses it in seconds; a frequency recalibration fixes it entirely. I suggest shipping that diagnostic (details below) — clone modules with bad crystals are common, and anyone hitting this will conclude "dialect-A pairing doesn't work."

## Setup

- Heater: ATC Sun Ray RF 1800, factory state (announces on net `0000`), dialect A (sync `2DE5`, PN9, CRC-CCITT) — confirmed by decode.
- Gateway: ESP32-S3 + CC1101, `esphome/termoweb_radio.h` 3.7-esp32 (stock register table), station id `01`.
- Integration: ha-termoweb v2.4.0, dialect A, network id `1B30`.
- RSSI both directions −50…−68 dBm at ~2 m.

## The failure as users will see it

- Pairing announcements received perfectly (CRC ok, strong RSSI): `net 0000, src FF, dst sweeping 01..41, tag 03, payload 77 + 12-byte identity` (note: tag 03 with a dialect-A identity — a hybrid of the two announcement styles documented in §7).
- Every assignment ignored. The heater **never acks anything** (unlike your 2026-09-09 bench, where an already-joined heater *acked* and refused — a different failure). Behavior is identical whether the station answers or stays silent.
- Point-blank range changes nothing (offset is distance-independent).
- `Pairing finished: 0 heater(s) paired`, forever.

Things I ruled out on the way, in case they help your docs: timing (I modified the ESP firmware to answer inside ~10 ms of the probe — irrelevant, and unnecessary, see below), network id (tried the site-derived id and `1B30`), ack-then-assignment sequencing, and the heater-side procedure (manual confirms OK-for-3 s is correct).

## Diagnosis

I added a one-line `FREQEST` (0x32) readout per CRC-valid frame. At the nominal 869 525 kHz dial, **every** heater frame measured `FREQEST = +42` — which is the FOC clamp (±BW/4 with `FOCCFG=0x16`), so the true offset was *beyond* +67 kHz. After retuning +67 kHz the residual read +9 (+14.3 kHz): true offset ≈ **+81 kHz ≈ 93 ppm**. Final dial for this module: **869 606 kHz**, after which `FREQEST ≈ 0` and the heater heard us for the first time.

## The adoption, decoded from the air (first on-channel attempt)

```
t+0        heater probe     18 0000 FF→01 fl00 path FF 01 01 01 01  tag 03  77+identity
t+0.01 s   station auto-ack 05 1B30 01→FF fl80
t+0.15 s   assignment (host T, pairing flow)
                            0C 1B30 01→FF fl00 path 01 FF 00 00 00  tag 04  payload 02
t+0.17 s   heater ack       05 1B30 FF→01 fl80          ← ack from FF on the NEW net
t+0.20 s   heater           0B 1B30 02→01 fl00 path 02 01 01 01 01  tag 84  (empty)
t+0.55 s   heater           0C 1B30 02→01 fl00 path 02 01 01 01 01  tag 00  payload 50
           (the 50 registration; repeated every ~6–9 s, several times)
```

Exactly the sequence §7 documents for dialect B (`proven 2026-10-10`), now observed for dialect A: **ack from FF on the new network → empty tag-84 → `50` registrations.** The heater kept the binding (paired icon on its display), answers status reads on id 2, and its energy counter reads correctly through the integration. Note the heater stopped announcing the instant it adopted — one probe, one assignment, done.

## Findings / suggestions

1. **Ship a frequency-offset diagnostic.** One snippet in `handle_packet_` (ESP firmware; same idea applies to termoweb_rx):

   ```c
   /* read before anything else touches the radio */
   uint8_t freqest = read_status(CC1101_FREQEST);   /* 0x32, signed, f_xosc/2^14 ≈ 1.587 kHz/LSB */
   ...
   if (crc_ok) emit("# freqest=%d", (int8_t) freqest);
   ```

   Surfacing this per valid frame (or in `D`) turns a dead-end "heater ignores everything" into a 30-second diagnosis. Watch for the clamp: a reading pinned at ±BW/4 means "at least this much". Even better: auto-trim (`FSCTRL0`, or retune the dial) from a running average — the gateway would self-calibrate to whatever the installed heaters transmit.

2. **Timing is a non-issue for pairing** — the heater adopted from an answer 150 ms after its probe. The main-repo flow's reactive answering is fast enough over WiFi; no firmware-side fast path is needed. (The "~16 ms ack window" from normal operation does not apply to the pairing listen.)

3. **Confirm flow gap:** because the heater stops announcing immediately on adoption, `pair_heaters` can miss its own success — in my run the window ended with "0 paired" while the heater sat happily bonded; the next id-scan (integration reload) found it on id 2. Suggestion: after transmitting an assignment, poll the assigned id for a few seconds regardless of further announcements, and/or treat the FF-ack + tag-84/`50` frames as the success signal. Also: the heater retries its `50` registration several times, which suggests it expects a station-side registration reply (ha-termoweb-local documents an EB clock sync with the `0x51` prefix); it keeps the binding without one, but answering it may be more polite.

4. **Docs:** a note that "receives perfectly but nothing you transmit gets any reaction, at any range" ⇒ check the module's crystal (FREQEST) would have saved this installation several evenings. The README's "tested with ATC Sun Ray heaters" now also covers from-scratch dialect-A pairing — happy to retest anything you'd like for that.

Full logs (announcement sweeps, decoded frames, FREQEST measurements before/after retune, the adoption capture) available on request.
