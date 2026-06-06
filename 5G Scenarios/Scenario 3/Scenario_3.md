# (Scenario 3) Free5GC (VM) + 2 UPF (docker) + UERANSIM GNB (docker) / UE (docker)

# (Scenario 3) Free5GC (VM) + 2 UPF (docker) + UERANSIM GNB (docker) / UE (docker)

We further increase the complexity of the network configuration. A second UPF is added, and therefore another DNN to provide Internet access.

IMPORTANT: There is only one slice, although with two different DNNs. UE1 uses "internet" and UE2 uses "internet2".

images/165-1.png

## PREVIOUS STEP: enable GTP5G in GNS3 VM HV and enable FORWARDING

Make sure to repeat the steps described in Scenario 2.

## Core: FREE5GC

images/165-2.png

The CORE network is based on the Free5GC_GNS3 template.

Repeat the initial process from Scenario 2 up to the SMF configuration changes.

Ensure correct configuration of UPF1.

Then add UPF2 by modifying the original SMF structure:

1. Duplicate the UPF structure
2. Rename them as UPF1 and UPF2
3. Keep one NSSAI for UPF1 (sd: 010203, dnn: internet)
4. Keep one NSSAI for UPF2 (sd: 010203, dnn: internet2)

Add the following line to both UPFs:

```
ulcl: false
```

This disables ULCL (Uplink Classifier), which is not required for this setup.

Then update amfcfg.yaml:

- Ensure AMF IP is correct
- Add internet2 to supportDnnList

## UPF1 and UPF2 (docker)

images/165-3.png

Use template: gitunican-UPF-free5gc

- UPF1: 10.10.10.20
- UPF2: 10.10.10.21

Each connects to a different NAT (DNN simulation).

## UE Routing (uerouting.yaml)

Used to route traffic to the correct UPF depending on UE:

- UE1 → UPF1 → internet
- UE2 → UPF2 → internet2

## UERANSIM (GNB + UE1 + UE2)

images/165-4.png
images/165-5.png
images/165-6.png

Same configuration as previous scenarios.

IMPORTANT:
UE2 must use APN "internet2".

Create second subscriber in WebConsole with DNN internet2.

Once connected, configure UE default routes via uesimtun0.

If ping fails, verify:
- UPF default route
- NAT connectivity

## Packet capture validation

- Capture on UPF1 → UE1 traffic
- Capture on UPF2 → UE2 traffic

images/165-12.png
images/165-13.png

## File extraction from Docker

Use volumes (typically /gns3volumes).

Example:

```
docker cp <container_id>:/gns3volume/free5gc/config/upfcfg_2.yaml
```

## File transfer without SSH

On VM:
```
python3 -m http.server 8080
```

On host:
```
curl http://IP_VM:8080/file -o file
```

## Shared folders (VirtualBox)

Mount example:

```
sudo mkdir -p /mnt/shared
sudo mount -t vboxsf shared /mnt/shared
```

## CRLF / LF issues

Convert with sed:

```
sed -i 's/\r$//' file.txt
```

Or use the provided batch script to convert files recursively.

