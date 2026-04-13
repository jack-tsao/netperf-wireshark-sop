# SOP: NetPerf TCP Performance Capture & Analysis with Wireshark

## Purpose

Capture and analyze TCP_RR and TCP_CRR network performance between two machines using NetPerf, tcpdump, and Wireshark. This procedure helps identify connection-level latency bottlenecks (e.g., SYN→SYN-ACK timing) when comparing boards or NIC controllers.

---

## Prerequisites

- Linux server and Linux or Windows client connected via Ethernet
- `netperf` installed on both (`apt install netperf` or `yum install netperf`)
- `tcpdump` installed on the client (`apt install tcpdump`)
- Wireshark installed on the client or a separate machine for analysis
- Ethernet cable(s) and optionally a switch

---

## 1. Network Setup

### Option A: Connected Through a Switch (No DHCP)

A switch does not assign IP addresses. Set static IPs manually on both machines.

**On the server:**

```bash
# Identify the Ethernet interface (not "lo" or "wlan")
ip link show

# Assign a static IP
sudo ip addr add 192.168.20.1/24 dev <interface_name>
sudo ip link set <interface_name> up
```

**On the client:**

```bash
sudo ip addr add 192.168.20.3/24 dev <interface_name>
sudo ip link set <interface_name> up
```

**If the client is Windows**, go to Settings → Network & Internet → Ethernet → Edit IP settings → Manual → IPv4 On:

- IP: `192.168.20.3`
- Subnet: `255.255.255.0`
- Gateway: leave blank
- DNS: leave blank

### Option B: Connected Through a Router (DHCP)

Both machines should receive IPs automatically. Find them with:

```bash
# Linux
ip addr show <interface_name>

# Windows
ipconfig
```

Note the IP addresses assigned to each machine.

### Verify Connectivity

```bash
# From the client, ping the server
ping <server_ip>
```

If pinging from Linux to Windows fails, disable Windows Firewall for ICMP or run on Windows (as Administrator):

```cmd
netsh advfirewall firewall add rule name="Allow ICMPv4" protocol=icmpv4:8,any dir=in action=allow
```

---

## 2. Start NetPerf Server

On the **server** machine:

```bash
netserver
```

If you get an error like `unable to start netserver with 'IN(6)ADDR_ANY' port '12865'`, it is already running. Confirm with:

```bash
ps aux | grep netserver
```

---

## 3. Capture Traffic with tcpdump

On the **client** machine, start the packet capture before running any tests:

```bash
sudo tcpdump -i <interface_name> -w /tmp/capture.pcap -s 0 &
TCPDUMP_PID=$!
```

- `-i <interface_name>`: the Ethernet interface (e.g., `eno1`, `enp0s31f6`)
- `-w /tmp/capture.pcap`: output file path
- `-s 0`: capture full packets (no truncation)
- `&`: run in background
- `TCPDUMP_PID=$!`: save the process ID so you can stop it cleanly later

---

## 4. Run NetPerf Tests

On the **client** machine, with tcpdump running in the background:

```bash
# TCP_RR: Request/Response over a single persistent connection
netperf -H <server_ip> -t TCP_RR -l 60

# TCP_CRR: Connect/Request/Response (new connection per transaction)
netperf -H <server_ip> -t TCP_CRR -l 60
```

- `-H <server_ip>`: IP address of the server (e.g., `192.168.20.1`)
- `-t TCP_RR` or `-t TCP_CRR`: test type
- `-l 60`: test duration in seconds (use `-l 10` for a quick test)

**Record the Trans/sec value** from each test output.

### Understanding the Tests

| Test | What It Does | What It Measures |
|------|-------------|-----------------|
| TCP_RR | Sends requests/responses over one open connection | Sustained connection throughput |
| TCP_CRR | Opens a new TCP connection for every transaction | Connection setup/teardown overhead |

**Higher Trans/sec = better.** If TCP_RR is similar across boards but TCP_CRR differs, the bottleneck is in connection establishment (NIC, driver, or OS stack).

---

## 5. Stop the Capture

Stop the tcpdump process using the saved PID:

```bash
sudo kill -INT "$TCPDUMP_PID"
```

Verify the capture file exists:

```bash
ls -lh /tmp/capture.pcap
```

---

## 6. Open the Capture in Wireshark

If Wireshark is installed on the client, open the file directly:

```bash
wireshark /tmp/capture.pcap &
```

If Wireshark is on a different machine, transfer the pcap file first:

```bash
# From the Wireshark machine, pull the file via SCP
scp <user>@<client_ip>:/tmp/capture.pcap ~/Desktop/

# Or copy via USB
cp /tmp/capture.pcap /media/<usb_mount>/
```

---

## 7. Analyze in Wireshark

Open `capture.pcap` in Wireshark.

### View All New Connections (SYN packets)

In the filter bar:

```
tcp.flags.syn==1 && tcp.flags.ack==0
```

Each result is a new TCP connection. The total count should roughly equal `Trans/sec × test duration`.

### View a Single Connection

Pick a source port from the SYN list (e.g., `11264`) and filter:

```
tcp.port==11264
```

This shows all ~10 packets of one TCP_CRR transaction:

1. **SYN** (client → server) — connection request
2. **SYN-ACK** (server → client) — server acknowledges
3. **ACK** (client → server) — handshake complete
4. **Data exchange** — request and response
5. **FIN / FIN-ACK** — connection teardown

The time gap between packet 1 (SYN) and packet 2 (SYN-ACK) is the **server response latency** — this is the key metric to compare across boards.

### Change Time Display Format

Right-click the **Time** column header → choose **Seconds Since Previous Displayed Packet** to easily see gaps between consecutive packets.

### Check for Problems

| Filter | What It Shows |
|--------|--------------|
| `tcp.analysis.retransmission` | Retransmitted packets (packet loss) |
| `tcp.flags.reset==1` | Forced connection resets |
| `tcp.analysis.duplicate_ack` | Duplicate ACKs (congestion signal) |

Any of these in significant numbers indicate network or driver issues beyond simple latency.

### Useful Statistics

- **Statistics → Conversations → TCP tab**: shows duration of every connection
- **Statistics → I/O Graphs**: visualize throughput over time
- **Analyze → Expert Information**: summary of warnings and errors

---

## 8. Command-Line Analysis with tshark (Optional)

If Wireshark GUI is not available, use `tshark` on Linux:

```bash
# Install tshark
sudo apt install tshark

# Count SYN packets (total new connections)
tshark -r /tmp/capture.pcap -Y "tcp.flags.syn==1 && tcp.flags.ack==0" | wc -l

# View first 20 packets of a specific connection
tshark -r /tmp/capture.pcap -Y "tcp.port==11264" | head -20

# Export SYN timing for analysis
tshark -r /tmp/capture.pcap \
  -Y "tcp.flags.syn==1 && tcp.flags.ack==0" \
  -T fields -e frame.time_relative -e tcp.srcport
```

---

## Notes

- Always run tcpdump **before** starting the NetPerf test and stop it **after** the test completes.
- For comparing boards, keep the server, switch, cable, and switch port the same — only swap the client board.
- Use shorter test durations (`-l 10`) when capturing to keep pcap files manageable.
- A 60-second TCP_CRR test at ~5000 trans/sec generates ~500,000 connections and a large pcap file. For Wireshark analysis, 10 seconds is usually enough.