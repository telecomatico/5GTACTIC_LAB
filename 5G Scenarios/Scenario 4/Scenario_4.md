# (Scenario 4) Scenario 3 + 2 slices

# (Scenario 4) 2 Slice - Free5GC (VM) + 2 UPF (docker) + UERANSIM GNB (docker) / UE (docker)

The topology is the same as in Scenario 3. The only difference is that two different slices are defined, one on each UPF.

- UPF1: slice 010203 → provides access to "internet"
- UPF2: slice 112233 → provides access to "internet2"

images/167-1.png

Modifications are minimal.

## PREVIOUS STEP: enable GTP5G in GNS3 VM HV and enable FORWARDING

Repeat the steps described in Scenario 3.

## Core: FREE5GC

images/167-2.png

The CORE network is based on the Free5GC_GNS3 template.

Repeat the setup from Scenario 2 until the SMF configuration stage.

You can start from the `smfcfg.yaml` from Scenario 3 and modify only the parts related to UPF2:

### Changes in SMF configuration

- Add the new S-NSSAI:

```
- sNssai:
    sst: 1
    sd: 112233
  dnnInfos:
    - dnn: internet
    - dnn: internet2
```

- Modify UPF2 so it serves only the new slice:

```
UPF2:
  nodeID: 10.10.10.21
  addr: 10.10.10.21
  sNssaiUpfInfos:
    - sNssai:
        sst: 1
        sd: 112233
      dnnUpfInfoList:
        - dnn: internet2
```

The complete `smfcfg.yaml` follows the same structure as Scenario 3, with:

- UPF1 → slice 010203 → internet
- UPF2 → slice 112233 → internet2

## UERANSIM (GNB + UE1 + UE2)

images/167-3.png
images/167-4.png
images/167-5.png

Use the same Docker images as in previous scenarios.

Configuration is identical except for UE2.

### UE2 configuration change

Modify the configuration file to use the new slice (112233):

```
sessions:
  - type: 'IPv4'
    apn: 'internet2'
    slice:
      sst: 0x01
      sd: 0x112233

configured-nssai:
  - sst: 0x01
    sd: 0x112233

default-nssai:
  - sst: 0x01
    sd: 0x112233
```

Start UE2 with this new configuration file.

## Subscriber configuration

Register the new UE (UE2/UE3 depending on naming) in the WebConsole:

- Assign a new IMEI
- Configure S-NSSAI with SD = 112233
- Set DNN = internet2

