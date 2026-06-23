# Configs

Ready-to-use configuration files referenced in the main [README](../README.md). Copy these in as a starting point, then adjust paths, usernames, and identifiers (IMSI/K/OPc) to match your own setup.

| File | Used by | Goes on |
|---|---|---|
| [`gnb_srsRAN.yml`](./gnb_srsRAN.yml) | srsRAN Project gNB | PC2 |
| [`ue.conf`](./ue.conf) | srsUE (srsRAN_4G) | PC1 |
| [`open5gs-webui.service`](./open5gs-webui.service) | Open5GS WebUI (systemd unit) | PC2 |

### Notes

- In `open5gs-webui.service`, replace `<your-user>` with your actual Linux username and confirm the `WorkingDirectory` matches where you cloned `open5gs/webui`.
- In `gnb_srsRAN.yml`, the `cu_cp.amf.addr` / `bind_addr` values (`127.0.0.5` / `127.0.0.1`) assume Open5GS is running locally on the same machine as the gNB. Confirm with:
  ```bash
  sudo ss -lnp | grep 38412
  ```
- In `ue.conf`, the `[usim]` block (`imsi`, `k`, `opc`) **must exactly match** the subscriber record you create in the Open5GS WebUI. Mismatched values are the most common cause of authentication failure.
- Both `gnb_srsRAN.yml` and `ue.conf` set `clock: internal` / `device_args = type=b200,clock=internal`. This works for short test sessions but will drift over time — see [Phase 8 in the main README](../README.md#phase-8--clock-drift-issue-on-reconnection) if you need a long-running connection.
