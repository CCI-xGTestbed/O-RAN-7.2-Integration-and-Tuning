# System Testing and Verification

## Starting the System

Follow these steps in order, each in a separate terminal. Some steps will be run on different machines for the disaggregated deployment.

**Step 1 — Reboot O-RU:**

```bash
ssh root@10.10.0.100
reboot
```

Wait for the O-RU to come back online:

```bash
ping 10.10.0.100
```

**Step 2 — Start PTP synchronization (Terminal 1 on DU Machine):**

```bash
sudo su
/usr/local/sbin/ptp4l -i enp13s0f0 -m \
    --domainNumber 24 \
    --masterOnly 1 \
    --slaveOnly 0 \
    --network_transport L2 \
    --tx_timestamp_timeout 1000 \
    --logAnnounceInterval -3 3 \
    --clockClass 6
```

**Step 3 — Start PHC2SYS (Terminal 2 on DU Machine):**

```bash
sudo su
/usr/local/sbin/phc2sys -s enp13s0f0 -w -m \
    -f /home/[YOUR_USER]/srs-ptp-gm.cfg
```

**Step 4 — Verify PTP synchronization on O-RU (Terminal 3) on DU Machine:**

```bash
ssh root@10.10.0.100
tail -f /var/log/pcm4l
```

Wait until the offset stabilizes below ±100ns. The O-RU will also show a lock message on time and frequency.

**Step 5 — Start Open5GS Core (Terminal 4) on CU Machine:**

```bash
cd ~/open5gs/build/tests/app
sudo ./5gc
```

**Step 6 — Start srsRAN CU (Terminal 5) on CU Machine:**

```bash
cd ~/srsRAN_Project/apps/cu
sudo ./srscu -c ran650_cu.yml
```

**Step 7 — Start srsRAN DU (Terminal 6) on DU Machine:**
```bash
cd ~/srsRAN_Project/apps/du
sudo ./srscu -c ran650_du.yml
```

### Verification

**Check O-RU transmission stats** (on the O-RU terminal):

```bash
tail -f /tmp/logs/radio_status
```

Then press `Ctrl+C` and run:

```bash
cd /tmp/logs/
TXMeanPower
kpi.sh
```

When everything is working correctly, you should see:

- PTP synchronization with stable offset values
- Packet statistics in `kpi.sh` showing increasing `RX_ON_TIME_C` count
- `TXMeanPower` showing actual RF power output
- UEs connecting successfully to the network

When a UE has attached, you will see a noticeable increase in the `RX_ON_TIME_C` and `TX_TOTAL` metrics.
