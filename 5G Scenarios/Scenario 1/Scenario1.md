# (Scenario 1) Free5GC (VM) + Ueransim GNB (docker) + Ueransim UE (docker)

# Free5GC (VM) + Ueransim GNB (docker) + Ueransim UE (docker)

This is the simplest possible configuration, where CORE and RAN are in the same network (10.10.10.0/24). All elements are interconnected through a simple switch. The Core node is the only one that has Internet access.

![](images/160-1.png)

## Core: FREE5GC

The CORE network is based on the basic virtual machine recommended by Free5GC (see 5GTACTIC--CORE--FREE5GC--Instalación_en_Ubuntu_22.04_30.html)

The VM is created on Aprendiendo_Python--VirtualBox_83.html, and a template has been created in GNS3 named Free5GC_GNS3

It includes two network interfaces:

- enp0s3: Connects to the operator network, with IP 10.10.10.155/24  
- enp0s8: Connects to the Internet using a NAT node  

Its configuration is located at:

```
/etc/netplan/00-installer-config.yaml
```

![](images/160-2.png)

The default route must always go through interface enp0s8 (Internet), which will act as the Core DNN.

To start the Core:

1. Check that the gtp5g module is active:
```
modprobe gtp5g
```

2. Enable NAT forwarding to act as Internet gateway:
```
./reload_host_config.sh enp0s3
```

3. Start the MongoDB docker:
```
sudo docker run -d --name mongodb -p 27017:27017 mongo:4.2
```

4. Launch the core in background:
```
sudo ./run.sh &
```

## Editing the Subscriber Database

If subscription parameters need to be modified, you must enable the WebConsole application in the Core VM:

```
cd ~/free5gc/webconsole
./bin/webconsole
```

To access the application, a docker container with a browser is used. The template is automatically created in GNS3 by following:

*NewTemplate -> Install an appliance from GNS3 Server -> Guests -> Webterm*

Once the Webterm node is added, configure the network by modifying the text file at:

*Configure -> Network (Edit)*

```
# Static config for eth0
auto eth0
iface eth0 inet static
 address 10.10.10.3
 netmask 255.255.255.0
 gateway 10.10.10.1
```

Once the node is started, access the console (it opens a VNC), which directly shows the browser. Enter the Core node URL, using:

- user: *admin*  
- password: *free5gc*  

If there is no subscriber in the database, create one with default values, except change:

- Operator Code Type from **OPC** to **OP**

![](images/160-3.png)

## UERANSIM (GNB + UE)

The docker images used for GNB and UE are obtained from the Free5GC distribution (see 5GTACTIC--CORE--FREE5GC--Instalación_en_docker_39.html).

In this case, images of all elements have been generated (base images appear as free5gc/xx, e.g., free5gc/ueransim), but only the UPF docker and ueransim docker have been built (make), resulting in:

- free5gc-compose-upf  
- free5gc-compose-ueransim  

The installation must be performed directly on the GNS3 VM HV machine, which must have Internet access (see Virtualización--GNS3--GNS3_con_Hyper-V_VM--Añadir_acceso_a_Internet_a_la_GNS3_VM_158.html)

IMPORTANT: GNB and UE versions must be the same (different versions cannot be mixed)

The template in GNS3 is created from these images and assigned two network interfaces:

- One connected to 10.10.10.0/24  
- One for optional Internet access  

Configuration:

```
# Static config for eth0
auto eth0
iface eth0 inet static
 address 10.10.10.2
 netmask 255.255.255.0
 gateway 10.10.10.1

# DHCP config for eth1
auto eth1
iface eth1 inet dhcp
```

TIP: If you need to install packages, stop the docker container and connect eth1 to a NAT cloud to provide Internet access.

Currently there is only one template for both GNB and UE:  
**free5gc-ueransim**

IMPORTANT:
The docker images contain only the basics to run the function:
- No text editor  
- No network tools (except `ip`)  
- Configuration files are not included → must be created manually  

TODO:
Check whether specific templates for GNB or UE can be created including startup commands and configuration files.

---

### GNB Configuration File (free5gc-gnb.yaml)

```
mcc: '208'
mnc: '93'
nci: '0x000000010'
idLength: 32
tac: 1

linkIp: 10.10.10.7
ngapIp: 10.10.10.7
gtpIp: 10.10.10.7

amfConfigs:
  - address: 10.10.10.155
    port: 38412

slices:
  - sst: 0x1
    sd: 0x010203

ignoreStreamIds: true
```

Start it with:

```
./nr-gnb -c config/free5gc-gnb.yaml &
```

---

### UE Configuration File (free5gc-ue.yaml)

```
supi: 'imsi-208930000000001'
mcc: '208'
mnc: '93'

protectionScheme: 0

homeNetworkPublicKey: '...'
homeNetworkPublicKeyId: 1
routingIndicator: '0000'

key: '...'
op: '...'
opType: 'OP'
amf: '8000'

imei: '356938035643803'
imeiSv: '4370816125816151'

gnbSearchList:
  - 10.10.10.7

sessions:
  - type: 'IPv4'
    apn: 'internet'
    slice:
      sst: 0x01
      sd: 0x010203
```

Start it with:

```
./nr-ue -c config/free5gc-ue.yaml &
```

---

## Process

Start the CORE and launch Free5GC  

Start the GNB:

![](images/160-4.png)

Start the UE:

![](images/160-5.png)

Association is visible on the GNB:

![](images/160-6.png)

Once the UE-UPF link is established, configure the UE default route:

![](images/160-7.png)
