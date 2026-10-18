# Changelog

## 1.2.0

- Enable remote sources by default by setting `remote_source_status` to `using` before sending values.
- React to source temperature-unit changes and validate the full-precision converted temperature before rounding.

## 1.1.0

- Coalesce rapid source changes and publish the latest state after a configurable quiet period.
- Reject unsupported or missing measurement units and compare freshness without rounding age.
- Make recovery `online` state publication independent of whether an item fallback was selected.
- Clarify MQTT order, QoS, retain behavior, unit requirements, and startup/refresh behavior.

## 1.0.0

- First release.
