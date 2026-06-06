# (Scenario 2) Free5GC (VM) + UPF (docker) + UERANSIM GNB (docker) / UE (docker)

# Free5GC (VM) + UPF (docker) + UERANSIM GNB (docker) / UE (docker)

We continue testing simple configurations. In this case, the Free5GC network functions are executed in the VM, but without including the UPF, which is added as an external Docker. All elements remain in the same network (10.10.10.0/24), but Internet access is only available through the UPF (IMPORTANT: THIS MEANS THAT ONLY THE UE WILL HAVE INTERNET ACCESS ONCE CONNECTED!!!)

Except for the UPF, all other elements are identical to those in Scenario 1, except that the NAT is connected only to the UPF, and the VM configuration must be modified.

images/161-1.png

## PREVIOUS STEP: Enable GTP5G in GNS3 VM HV and enable FORWARDING

We must ensure that the host where the Docker containers are executed (in this case the UPF) includes the GTP5G library. Since the containers run on the GNS3 VM HV, we install it there.

Install the GTP kernel module used by the UPF:

```
sudo git clone -b v0.9.5 https://github.com/free5gc/gtp5g.git
cd gtp5g
sudo make
sudo make install
modprobe gtp5g
```

NOTE: If it fails, reinstall.

NOTE: During compilation, the message “Skipping BTF generation ... due to unavailability of vmlinux” may appear (documented separately).

Before proceeding, ensure that FORWARDING is enabled on the GNS3 VM so that Docker containers inherit this configuration (required for the UPF to act as an Internet gateway):

```
sudo sysctl -w net.ipv4.ip_forward=1
```

IMPORTANT: Normally iptables rules are also configured here, but in this case it is sufficient to configure them directly in the UPF container.

## Core: FREE5GC

images/161-2.png

The CORE network is based on the template named Free5GC_GNS3.

Although it includes two network interfaces, only the first is connected to the switch to belong to the 10.10.10.0/24 network:

- enp0s3: operator network (10.10.10.155/24)
- enp0s8: left disconnected

At this point, the default route has no meaning.

To launch the Core, first modify the run.sh script by commenting out the lines that start the UPF.

images/161-3.png

Save it as run2.sh.

Before running the script, configure the SMF (config/smfcfg.yaml):

Modify:
- nodeID: 10.10.10.155
- listenAddr: 10.10.10.155
- externalAddr: 10.10.10.155
- UPF nodeID: 10.10.10.20
- UPF addr: 10.10.10.20

Complete file:

images/161-4.png
images/161-5.png
images/161-6.png
images/161-7.png
images/161-8.png

IMPORTANT: Also ensure that amfcfg.yaml is updated as in Scenario 1.

Now:

1. Check gtp5g module:
```
modprobe gtp5g
```

2. Enable NAT forwarding:
```
./reload_host_config.sh enp0s3
```

3. Start MongoDB:
```
sudo docker run -d --name mongodb -p 27017:27017 mongo:4.2
```

4. Launch core:
```
sudo ./run2.sh &
```

## Editing the Subscriber Database

The database may be reset after restarting the machine or the MongoDB container. You can enable the webconsole and recreate the subscriber (changing OPC to OP):

```
cd ~/free5gc/webconsole
go run server.go &
```

TODO: Add a persistent volume to MongoDB to preserve subscribers.

## UPF (docker)

images/161-9.png

The Docker image used is gitunican/upf-free5gc:seeder.

A GNS3 template named "gitunican-UPF-free5gc" has been created.

Like UERANSIM, it includes two interfaces:
- First: connects to CN (10.10.10.20/24)
- Second: connects to NAT (DNN)

UPDATE: The image includes configuration files and tools (nano, net-tools, ping).

Once the container is running, open a console and start the UPF:

```
cd free5gc
./upf -c config/upfcfg.yaml &
```

Then configure iptables:

```
iptables -t nat -A POSTROUTING -s 10.60.0.0/16 ! -o upfgtp -j MASQUERADE
```

Ensure the default route goes through NAT.

## UERANSIM (GNB + UE)

images/161-10.png
images/161-11.png

The Docker images are the same as in Scenario 1. Configuration is identical.

Start the containers and launch GNB and UE.

Once connected, configure the UE default route via uesimtun0.

NOTE: If ping to 8.8.8.8 fails from the UE, check the UPF default route (must go through NAT).

images/161-12.png
