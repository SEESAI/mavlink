# Proposal: `SEES_GPS_INPUT` message in a Sees MAVLink dialect

**Status:** Implemented in this repo ([`message_definitions/v1.0/sees.xml`](../../message_definitions/v1.0/sees.xml)); PX4 and sender changes pending
**Target:** PX4 v1.17 (mavlink branch `33af200_dev`, based on upstream `33af200d`)
**Replaces:** the `GPS_INPUT` changes on `origin/3b52eac_dev` used with PX4 v1.13

## Summary

Our GPS pipeline needs three fields that standard `GPS_INPUT` (#232) doesn't have:

- `course_accuracy`
- `heading_accuracy`
- `time_utc_usec`

For PX4 v1.13 we added these fields by editing `GPS_INPUT` in `common.xml`. For v1.17 we propose leaving `common.xml` unmodified. Instead, a Sees-specific dialect (`sees.xml`) will define a new message, `SEES_GPS_INPUT`. It contains every `GPS_INPUT` field plus the three Sees fields.

## Background: the v1.13 implementation

Commits `a27f43c3`, `7693cbec`, `fff86bef` and `460529e8` on `origin/3b52eac_dev` together make this change to `GPS_INPUT` in `common.xml`:

```xml
      <extensions/>
+     <field type="float"    name="course_accuracy"  units="rad">GPS course accuracy</field>
+     <field type="float"    name="heading_accuracy" units="rad">GPS heading accuracy</field>
      <field type="uint16_t" name="yaw" units="cdeg">...</field>          <!-- upstream -->
+     <field type="uint64_t" name="time_utc_usec" units="us">...</field>
```

The fields were placed after `<extensions/>` deliberately. Extension fields are not part of the message's CRC_EXTRA, so stock tools such as QGroundControl kept accepting the message without being rebuilt.

### Problems with that approach

1. **It conflicts with standard MAVLink.** The new fields were inserted *before* upstream's `yaw` extension.
   - The payload bytes where every standard implementation expects `yaw` now hold `course_accuracy`.
   - When Sees and standard code are mixed, `yaw` and the new fields are misread.
2. **Errors are silent.** Extension fields are outside the CRC, so a sender and receiver with different definitions still exchange the message without any error. They just get the wrong values.
3. **Upstream changes can collide.** If upstream adds extension fields to `GPS_INPUT`, their byte positions overlap ours with different meanings. Nothing would detect this.
4. **Rebases cost more.** Every move to a new upstream mavlink means carrying and re-checking a patch to `common.xml`.

## Proposal

### 1. Add a Sees dialect

Create `message_definitions/v1.0/sees.xml`, which includes `development.xml`. That is PX4's default dialect, so PX4 still sees every definition it expects.

```xml
<?xml version="1.0"?>
<mavlink>
  <include>development.xml</include>
  <version>1</version>
  <dialect>0</dialect>
  <messages>
    <message id="13000" name="SEES_GPS_INPUT">
      <description>Sees variant of GPS_INPUT with additional accuracy and UTC time fields.
        Field semantics are identical to GPS_INPUT except where noted.</description>
      <!-- All GPS_INPUT fields, copied verbatim (including yaw), as regular fields -->
      <field type="uint64_t" name="time_usec" units="us">...</field>
      <field type="uint8_t"  name="gps_id" instance="true">...</field>
      <field type="uint16_t" name="ignore_flags" enum="GPS_INPUT_IGNORE_FLAGS">...</field>
      <!-- ... remaining GPS_INPUT fields ... -->
      <field type="uint16_t" name="yaw" units="cdeg">...</field>
      <!-- Sees additions -->
      <field type="float"    name="course_accuracy"  units="rad" invalid="NaN">GPS course accuracy. NaN if unknown.</field>
      <field type="float"    name="heading_accuracy" units="rad" invalid="NaN">GPS heading accuracy. NaN if unknown.</field>
      <field type="uint64_t" name="time_utc_usec"    units="us"  invalid="0">GPS UTC time (UNIX epoch). 0 if unknown.</field>
    </message>
  </messages>
</mavlink>
```

`SEES_GPS_INPUT` is a new message, so every field is a regular field and there is no `<extensions/>` section. The full field list is copied from `GPS_INPUT` in `common.xml` at the time of implementation.

Field semantics for the Sees additions:

| Field | Type | Units | Meaning | Unknown value |
|---|---|---|---|---|
| `course_accuracy` | `float` | rad | Course-over-ground accuracy, 1-sigma (standard deviation) | `NaN` |
| `heading_accuracy` | `float` | rad | Heading accuracy, 1-sigma (standard deviation) | `NaN` |
| `time_utc_usec` | `uint64_t` | us | GPS UTC time, UNIX epoch | `0` |

Both accuracies are standard deviations, not variances. The same holds for PX4's `sensor_gps` fields. Despite its name, `c_variance_rad` holds a 1-sigma course accuracy, as `s_variance_m_s` does for speed. Values are therefore copied between `sensor_gps` and `SEES_GPS_INPUT` without conversion, as the v1.13 Sees PX4 code does:

```cpp
msg.speed_accuracy  = gps.s_variance_m_s;
msg.course_accuracy = gps.c_variance_rad;
```

Generated details (C, MAVLink 2):

| | Value |
|---|---|
| Message ID | 13000 |
| Payload length | 81 bytes (min length = max length) |
| CRC_EXTRA | 113 |

Standard `GPS_INPUT` is unchanged (CRC_EXTRA 151, as upstream).

### 2. Message ID allocation

The IDs used by the dialects in this repo cluster in these ranges:

- 0–12xxx
- 17xxx
- 42xxx
- 50xxx
- 52xxx
- 60xxx

The range **13000–13999** is unused. We propose reserving it for Sees messages, starting with `SEES_GPS_INPUT = 13000`.

Before merging, run mavgen over an `all.xml` that also includes `sees.xml` to check for duplicate IDs. That include should exist only locally and must not be committed upstream.

### 3. PX4 v1.17 changes

The details below are to be confirmed against the PX4 v1.17 tree.

1. **Submodule:** point `src/modules/mavlink/mavlink` at this repo's `33af200_dev` branch.
2. **Dialect:** set `CONFIG_MAVLINK_DIALECT="sees"` in the board configuration (`.px4board`).
3. **Receiver:** add `handle_message_sees_gps_input()` in `mavlink_receiver.cpp`.
   - Move the existing `GPS_INPUT` → `sensor_gps` mapping into a shared helper that both handlers use, so they can't drift apart.
   - Map the Sees fields onto the existing `sensor_gps` fields:

     | `SEES_GPS_INPUT` field | `sensor_gps` field |
     |---|---|
     | `heading_accuracy` | `heading_accuracy` |
     | `time_utc_usec` | `time_utc_usec` |
     | `course_accuracy` | `c_variance_rad` |

   - All mappings are direct copies; see the field semantics above.
   - No uORB `.msg` changes are expected.
   - Where PX4 *sends* the message (fills it from `sensor_gps`), the change is in that sending code, a stream or driver, rather than in `mavlink_receiver.cpp`. The same field mapping applies in reverse.
4. **Port the v1.13 changes:** go through the Sees PX4 v1.13 fork for any other code that reads these fields and port it.

### 4. Sender changes

The component that sends GPS data to the autopilot must:

- regenerate its MAVLink library from the same `sees.xml`;
- send `SEES_GPS_INPUT` instead of `GPS_INPUT`.

If the sender and the autopilot have different definitions, `SEES_GPS_INPUT` fails its CRC check and is dropped. It is never silently misread.

### 5. Ground station

**QGroundControl doesn't need to be rebuilt.** `SEES_GPS_INPUT` goes from the companion computer to the autopilot. If it is forwarded to QGC, QGC drops it as an unknown message.

## Benefits

| | v1.13 approach | Proposed |
|---|---|---|
| Change to `common.xml` | Yes | None |
| Compatible with standard `GPS_INPUT` | No (`yaw` displaced) | Yes (untouched) |
| Definition mismatch | Silent misread | Message dropped (CRC fails) |
| Upstream extensions can collide | Yes | No |
| Effort to rebase onto upstream | Re-apply patch to `common.xml` | Nothing (new file only) |
| QGC rebuild | Not needed | Not needed |

## Costs and risks

- **No wire compatibility with v1.13 senders.** Senders built for the v1.13 layout must be updated to send `SEES_GPS_INPUT`. This is acceptable because senders are being rebuilt against v1.17-era MAVLink anyway.
- **Upkeep of the copied fields.** `SEES_GPS_INPUT` duplicates the `GPS_INPUT` field list. If upstream extends `GPS_INPUT` and we want the new fields, we copy them in deliberately.
- **PX4 diff.** A small, self-contained change in `mavlink_receiver.cpp` is required.

## Generating the libraries

From the repo root, with the pymavlink requirements installed (`pip install -r pymavlink/requirements.txt`):

```bash
# C headers (for PX4, companion software, etc.)
python3 -m pymavlink.tools.mavgen --lang=C --wire-protocol=2.0 \
    --output=generated/include/mavlink/v2.0 message_definitions/v1.0/sees.xml

# Python (pymavlink dialect module)
python3 -m pymavlink.tools.mavgen --lang=Python --wire-protocol=2.0 \
    --output=generated/sees.py message_definitions/v1.0/sees.xml
```

Or with CMake: `cmake -Bbuild -DMAVLINK_DIALECT=sees -DMAVLINK_VERSION=2.0`.

## Implementation checklist

- [x] Add `message_definitions/v1.0/sees.xml` with the full `SEES_GPS_INPUT` definition
- [x] Run `./scripts/format_xml.sh -c`; it passes
- [x] Run `scripts/xml_consistency_check.py -f sees.xml`. It reports one false positive: "enum GPS_INPUT_IGNORE_FLAGS does not exist". In single-file mode the script doesn't follow `<include>`, and the enum is defined in `common.xml`.
- [x] Generate C and Python for the `sees` dialect with mavgen; a C encode → parse → decode round-trip test passes
- [x] Check for message ID collisions: generating from `all.xml` with `sees.xml` included locally succeeds
- [ ] Update PX4: submodule, `CONFIG_MAVLINK_DIALECT`, receiver handler, field mapping
- [ ] Update the GPS sender to emit `SEES_GPS_INPUT`
- [ ] Test end to end in SITL: values appear in `sensor_gps` (`listener sensor_gps`)
- [ ] Test on hardware with the real sender
