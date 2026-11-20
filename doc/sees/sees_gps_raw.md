# `SEES_GPS_RAW` message in a Sees MAVLink dialect

- **Status:**
  - Implemented in this repo ([`message_definitions/v1.0/sees.xml`](../../message_definitions/v1.0/sees.xml)) in `33af200_dev` branch, based on upstream `33af200d` used by PX4 v1.17.
  - PX4 sends it (see SEESAI/Firmware `v1.17.0-dev` branch).
  - MAVASK builds against it and decodes it into `Telemetry::GpsInput` for SeesInterface2 (See SEESAI/MAVSDK `v3.7.0_dev` branch)
- **Direction:** PX4 (autopilot) → companion computer. PX4 streams `SEES_GPS_RAW` from its `sensor_gps` topic
- **Replaces:** the `GPS_INPUT` changes on `origin/3b52eac_dev` used with PX4 v1.13
- **Rollout:** For changes to Firmware, MAVSDK and SeesInterface2, see `docs/sees/sees_gps_raw.md` in [SEESAI/Firmware](https://github.com/SEESAI/Firmware). This doc covers only the changes to this repo.

## Summary

Our GPS pipeline needs three fields that standard `GPS_INPUT` (#232) doesn't have:

- `course_accuracy`
- `heading_accuracy`
- `time_utc_usec`

PX4 publishes this data to the companion computer over MAVLink. For PX4 v1.13 we added these fields by editing `GPS_INPUT` in `common.xml` directly. For v1.17 we propose leaving `common.xml` unmodified. Instead, a Sees-specific dialect (`sees.xml`) will define a new message, `SEES_GPS_RAW`. It contains every `GPS_INPUT` field plus the three Sees fields.

The message is named after `GPS_RAW_INT`, the standard autopilot → companion GPS message, not after `GPS_INPUT`. `GPS_INPUT` carries data *into* the autopilot, the opposite direction.

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

## Changes

### 1. Adds a Sees dialect

Creates `message_definitions/v1.0/sees.xml`, which includes `development.xml` (which in turn includes `common.xml`). PX4's simulation target (`px4_sitl_default`) and upstream's main boards build with `development`, so PX4 still sees every definition it expects.

```xml
<?xml version="1.0"?>
<mavlink>
  <include>development.xml</include>
  <messages>
    <message id="13000" name="SEES_GPS_RAW">
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

`SEES_GPS_RAW` is a new message, so every field is a regular field and there is no `<extensions/>` section. The full field list is copied from `GPS_INPUT` in `common.xml` at the time of implementation.

The fields are the same as the v1.13 patched `GPS_INPUT`: the same 22 names, types and units. Only the declaration order differs: `yaw` now comes before the Sees fields. Code that decodes into the generated struct reads the same field names as before.

Field semantics for the Sees additions:

| Field | Type | Units | Meaning | Unknown value |
|---|---|---|---|---|
| `course_accuracy` | `float` | rad | Course-over-ground accuracy, 1-sigma (standard deviation) | `NaN` |
| `heading_accuracy` | `float` | rad | Heading accuracy, 1-sigma (standard deviation) | `NaN` |
| `time_utc_usec` | `uint64_t` | us | GPS UTC time, UNIX epoch | `0` |

Both accuracies are standard deviations, not variances, as is the inherited `speed_accuracy`.

Generated details (C, MAVLink 2):

| | Value |
|---|---|
| Message ID | 13000 |
| Payload length | 81 bytes (min length = max length) |
| CRC_EXTRA | 9 |

Standard `GPS_INPUT` is unchanged (CRC_EXTRA 151, as upstream).

### 2. Message ID allocation

The IDs used by the dialects in this repo cluster in these ranges:

- 0–12xxx
- 17xxx
- 42xxx
- 50xxx
- 52xxx
- 60xxx

The range **13000–13999** is unused. We propose reserving it for Sees messages, starting with `SEES_GPS_RAW = 13000`.

Before merging, run mavgen over an `all.xml` that also includes `sees.xml` to check for duplicate IDs. That include should exist only locally and must not be committed upstream.

## Benefits

| | v1.13 approach | Proposed |
|---|---|---|
| Change to `common.xml` | Yes | None |
| Compatible with standard `GPS_INPUT` | No (`yaw` displaced) | Yes (untouched) |
| Definition mismatch | Silent misread | Message dropped (CRC fails) |
| Upstream extensions can collide | Yes | No |
| Effort to rebase onto upstream | Re-apply patch to `common.xml` | Nothing (new file only) |
| QGC rebuild | Not needed: `GPS_INPUT` is standard (but QGC misread the Sees fields) | Needed to show `SEES_GPS_RAW`: QGC must be built with its definition |

## Costs and risks

- **No wire compatibility with v1.13.** Anything built for the v1.13 `GPS_INPUT` layout must be updated to use `SEES_GPS_RAW`.
- **Every user must generate from the same `sees.xml`.** Changing any field changes the CRC_EXTRA. A sender and receiver built from different definitions drop the message instead of misreading it.
- **QGroundControl must be rebuilt.** Stock QGroundControl, and the Sees build up to v4.3.0-0.0.5, don't know message 13000. They drop it as a CRC error, and count each one as a lost message in their link statistics, because PX4 also sends it at 1 Hz on the GCS link. To show `SEES_GPS_RAW` in the MAVLink Inspector, the Sees QGroundControl is built with headers that include its definition. QGroundControl needs no code change; the Firmware doc, section *QGroundControl*, has the details.
- **Upkeep of the copied fields.** `SEES_GPS_RAW` duplicates the `GPS_INPUT` field list. If upstream extends `GPS_INPUT` and we want the new fields, we copy them in deliberately.

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

PX4 and the SEESAI MAVSDK fork both build the `sees` dialect from this branch, with this repo's pymavlink submodule. When `sees.xml` changes, re-pin both to the same new commit; a commit that doesn't change `sees.xml` needs no re-pin.

## Implementation checklist

- [x] Add `message_definitions/v1.0/sees.xml` with the full `SEES_GPS_RAW` definition
- [x] Run `./scripts/format_xml.sh -c`; it passes
- [x] Run `scripts/xml_consistency_check.py -f sees.xml`. It reports one false positive: "enum GPS_INPUT_IGNORE_FLAGS does not exist". In single-file mode the script doesn't follow `<include>`, and the enum is defined in `common.xml`.
- [x] Generate C and Python for the `sees` dialect with mavgen; a C encode → parse → decode round-trip test passes
- [x] Check for message ID collisions: generating from `all.xml` with `sees.xml` included locally succeeds
- [x] Rename the message from `SEES_GPS_INPUT` to `SEES_GPS_RAW`
