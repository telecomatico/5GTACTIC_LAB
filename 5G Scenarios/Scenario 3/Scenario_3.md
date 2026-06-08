<div class="page">

# (Scenario 3) Free5GC(VM)+2UPF(docker)+UERANSIM GNB(docker)/UE(docker)

\

# (Scenario 3) Free5GC(VM)+2UPF(docker)+UERANSIM GNB(docker)/UE(docker)

We continue increasing the complexity of the network configuration. We add a second UPF, and then other DNN giving a new access to Internet.

IMPORTANT: There is only 1 slice, but 2 differentiated DNNs diferenciados. UE1 uses “internet” and UE2 uses “internet2”

![](images/165-1.png)

## PREVIOUS STEP: To activate GTP5G in GNS3 VM HV activating FORWARDING

We must ensure to repeat the steps indicated in [(Scenario2)
Free5GC(VM) + UPF(docker) + UERANSIM
GNB(docker)/UE(docker)](5GTACTIC--GNS3--Training_-_Configuraciones_GNS3--(Scenario2)_Free5GC(VM)_+_UPF(docker)_+_UERANSIM_GNB(docker)-UE(docker)_161.html)

## Core: FREE5GC

![](images/165-2.png)

The CORE network is based en el Template en con el nombre Free5GC_GNS3.

We replicate the initial process described in [(Scenario2)
Free5GC(VM) + UPF(docker) + UERANSIM
GNB(docker)/UE(docker)](5GTACTIC--GNS3--Training_-_Configuraciones_GNS3--(Scenario2)_Free5GC(VM)_+_UPF(docker)_+_UERANSIM_GNB(docker)-UE(docker)_161.html),
Until reaching all the changes into the SMF configuratoin.

We ensure proper configuration of el UPF1:

Changes in:

<div class="codebox">

    linea 52      nodeID: 10.10.10.155 #127.0.0.1 # the Node ID of this SMF
    línea 53      listenAddr: 10.10.10.155 #127.0.0.1 # the IP/FQDN of N4 interface on this SMF (PFCP)
    línea 54       externalAddr: 10.10.10.155 #127.0.0.1 # the IP/FQDN of N4 interface on this SMF (PFCP)

    línea 66       nodeID: 10.10.10.20 #127.0.0.8 # the Node ID of this UPF
    línea 67       addr: 10.10.10.20 #127.0.0.8 # the IP/FQDN of N4 interface on this UPF (PFCP)

    línea 94        - 10.10.10.155 #127.0.0.8

</div>

And we add the new UPF2, modifying original file structure, with two NSSAI into the UPF. In this case:

1.  Duplicate complete structure for UPF
2.  Rename the first as UPF1 and the second as UPF2
3.  Maintain the first NSSAI for UPF1, with sd: 010203 and
    dnn: internet (or coment NSSAI with id:112233)
4.  Maintain the first NSSAI for UPF2, with sd: 010203 but dnn:
    internet2, which appears twice (comment NSSAI with id:
    112233)

We just need to add en los dos UPF, just after addr:

<div class="codebox">

    ulcl: false

</div>

Into **Free5GC**, `ulcl: false`optoion is allocated into the
**`userplane_information`** section of `smfcfg.yaml` file, specifically inside
the UPF node definition. This part indicates if UPF supports **ULCL
(Uplink Classifier)**. If `ulcl` is `true`, SMF will try to
configue all the ULCL rules into the UPF, but this coud fail if UPF did not
support this function. For all the typical tests with UERANSIM and Free5GC its 
values is `false`.

General rules:

- If **no** uses ULCL architecture (split I‑UPF/PSA‑UPF), mantain
  `ulcl: false`, avoiding that SMF could generate uplink classification rules
  that could fail if UPF is not prepared for that.
- If you uses a unique UPF “all-in-one” (N3↔N6), use `ulcl: false`.
- If you want use ULCL (several UPFs with N9 between them), I‑UPF
  uses `ulcl: true` and you would have to add N9 links into `interfaces` and
  `links`, accordingly with this topology.

The resulting file would be:

<div class="codebox">

    :
      version: 1.0.7
      description: SMF initial local configuration

    configuration:
      smfName: SMF # the name of this SMF

      # Service-based interface information
      sbi:
        scheme: http # the protocol for sbi (http or https)
        registerIPv4: 127.0.0.2 # IP used to register to NRF
        bindingIPv4: 127.0.0.2 # IP used to bind the service
        port: 8000 # Port used to bind the service
        tls: # the local path of TLS key
          key: cert/smf.key # SMF TLS Certificate
          pem: cert/smf.pem # SMF TLS Private key

      # the SBI services provided by this SMF, refer to TS 29.502
      serviceNameList:
        - nsmf-pdusession # Nsmf_PDUSession service
        - nsmf-event-exposure # Nsmf_EventExposure service
        - nsmf-oam # OAM service

      # the S-NSSAI (Single Network Slice Selection Assistance Information) list supported by this AMF
      snssaiInfos:
        - sNssai: # S-NSSAI (Single Network Slice Selection Assistance Information)
            sst: 1 # Slice/Service Type (uinteger, range: 0~255)
            sd: 010203 # Slice Differentiator (3 bytes hex string, range: 000000~FFFFFF)
          dnnInfos: # DNN information list
            - dnn: internet # Data Network Name
              dns: # the IP address of DNS
                ipv4: 8.8.8.8
                ipv6: 2001:4860:4860::8888
            - dnn: internet2 # Data Network Name
              dns: # the IP address of DNS
                ipv4: 8.8.8.8
                
      # Optional: PLMN IDs configuration.
      plmnList:
        - mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
          mnc: 93 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
      locality: area1 # Name of the location where a set of AMF, SMF, PCF and UPFs are located

      # PFCP (Packet Forwarding Control Protocol) configuration for N4 interface.
      pfcp:
        # addr config is deprecated in smf config v1.0.3, please use the following config
        nodeID: 10.10.10.155 #127.0.0.1 # the Node ID of this SMF
        listenAddr: 10.10.10.155 #127.0.0.1 # the IP/FQDN of N4 interface on this SMF (PFCP)
        externalAddr: 10.10.10.155 #127.0.0.1 # the IP/FQDN of N4 interface on this SMF (PFCP)
        assocFailAlertInterval: 10s
        assocFailRetryInterval: 30s
        heartbeatInterval: 10s

      # Userplane nodes information, specifying details for AN and UPF.
      userplaneInformation:
        upNodes: # information of userplane node (AN or UPF)
          gNB1: # the name of the node
            type: AN # the type of the node (AN or UPF)
             an_ip: 10.10.10.7
          UPF1: # the name of the node
            type: UPF # the type of the node (AN or UPF)
            nodeID: 10.10.10.20 #127.0.0.8 # the Node ID of this UPF
            addr: 10.10.10.20 #127.0.0.8 # the IP/FQDN of N4 interface on this UPF (PFCP)
            ulcl: false
            sNssaiUpfInfos: # S-NSSAI information list for this UPF
              - sNssai: # S-NSSAI (Single Network Slice Selection Assistance Information)
                  sst: 1 # Slice/Service Type (uinteger, range: 0~255)
                  sd: 010203 # Slice Differentiator (3 bytes hex string, range: 000000~FFFFFF)
                dnnUpfInfoList: # DNN information list for this S-NSSAI
                  - dnn: internet
                    pools:
                      - cidr: 10.60.0.0/16
                    staticPools:
                      - cidr: 10.60.100.0/24
            # The IP address for the N3 interface must match the IP bound in the UPF N3 interface.
            # If UPF binds to a specific IP, set that same IP here.
            # If UPF binds to all IPs (0.0.0.0), you can set any of the available UPF IPs here.
            # Do NOT set 0.0.0.0 as this is not a valid routable address.
            interfaces: # Interface list for this UPF
              - interfaceType: N3 # the type of the interface (N3 or N9)
                endpoints: # the IP address of this N3/N9 interface on this UPF
                  - 10.10.10.20 #127.0.0.8
                networkInstances: # Data Network Name (DNN)
                  - internet
          UPF2: # the name of the node
            type: UPF # the type of the node (AN or UPF)
            nodeID: 10.10.10.21 #127.0.0.8 # the Node ID of this UPF
            addr: 10.10.10.21 #127.0.0.8 # the IP/FQDN of N4 interface on this UPF (PFCP)
            ulcl: false
            sNssaiUpfInfos: # S-NSSAI information list for this UPF
              - sNssai: # S-NSSAI (Single Network Slice Selection Assistance Information)
                  sst: 1 # Slice/Service Type (uinteger, range: 0~255)
                  sd: 010203 # Slice Differentiator (3 bytes hex string, range: 000000~FFFFFF)
                dnnUpfInfoList: # DNN information list for this S-NSSAI
                  - dnn: internet2
                    pools:
                      - cidr: 10.61.0.0/16
                    staticPools:
                      - cidr: 10.61.100.0/24
            # The IP address for the N3 interface must match the IP bound in the UPF N3 interface.
            # If UPF binds to a specific IP, set that same IP here.
            # If UPF binds to all IPs (0.0.0.0), you can set any of the available UPF IPs here.
            # Do NOT set 0.0.0.0 as this is not a valid routable address.
            interfaces: # Interface list for this UPF
              - interfaceType: N3 # the type of the interface (N3 or N9)
                endpoints: # the IP address of this N3/N9 interface on this UPF
                  - 10.10.10.21 #127.0.0.8
                networkInstances: # Data Network Name (DNN)
                  - internet2
        # the topology graph of userplane, A and B represent the two nodes of each link
        links:
          - A: gNB1
            B: UPF1
          - A: gNB1
            B: UPF2

      # retransmission timer for PDU session modification command
      t3591:
        enable: true # true or false
        expireTime: 16s # default is 6 seconds
        maxRetryTimes: 3 # the max number of retransmission

      # retransmission timer for PDU session release command
      t3592:
        enable: true # true or false
        expireTime: 16s # default is 6 seconds
        maxRetryTimes: 3 # the max number of retransmission

      nrfUri: http://127.0.0.10:8000 # a valid URI of NRF
      nrfCertPem: cert/nrf.pem # NRF Certificate
      urrPeriod: 30 # default usage report period in seconds
      urrThreshold: 500000 # default usage report threshold in bytes
      requestedUnit: 1000

    # Logging Configuration
    logger:
      enable: true # true or false
      level: info # how detailed to output, value: trace, debug, info, warn, error, fatal, panic
      reportCaller: false # enable the caller report or not, value: true or false

</div>

Now we need to modify el fichero amfcfg.yaml.

We start testing if AMF IP is correct:

<div class="codebox">

      amfName: AMF # the name of this AMF
      ngapIpList:  # the IP list of N2 interfaces on this AMF
        - 10.10.10.155 #127.0.0.18

</div>

Only one small change remains en la línea 57, addinginternet2:

<div class="codebox">

      supportDnnList:
        - internet
        - internet2  

</div>

The file becomes:

<div class="codebox">

    info:
      version: 1.0.9
      description: AMF initial local configuration

    configuration:
      amfName: AMF # the name of this AMF
      ngapIpList:  # the IP list of N2 interfaces on this AMF
        - 10.10.10.155 #127.0.0.18
      ngapPort: 38412 # the SCTP port listened by NGAP

      # Service-based Interface (SBI) Configuration
      sbi:
        scheme: http # the protocol for sbi (http or https)
        registerIPv4: 127.0.0.18 # IP used to register to NRF
        bindingIPv4: 127.0.0.18  # IP used to bind the service
        port: 8000 # port used to bind the service
        tls: # the local path of TLS key
          pem: cert/amf.pem # AMF TLS Certificate
          key: cert/amf.key # AMF TLS Private key

      # SBI Services offered by this AMF, as per TS 29.518
      serviceNameList:
        - namf-comm # Namf_Communication service
        - namf-evts # Namf_EventExposure service
        - namf-mt   # Namf_MT service
        - namf-loc  # Namf_Location service
        - namf-oam  # OAM service

      # Guami (Globally Unique AMF ID) list supported by this AMF
      servedGuamiList:
        # <GUAMI> = <MCC><MNC><AMF ID>
        - plmnId: # Public Land Mobile Network ID, <PLMN ID> = <MCC><MNC>
            mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
            mnc: 93 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
          amfId: cafe00 # AMF identifier (3 bytes hex string, range: 000000~FFFFFF)

      # the TAI (Tracking Area Identifier) list supported by this AMF
      supportTaiList:
        - plmnId: # Public Land Mobile Network ID, <PLMN ID> = <MCC><MNC>
            mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
            mnc: 93 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
          tac: 000001 # Tracking Area Code (3 bytes hex string, range: 000000~FFFFFF)

      # the PLMNs (Public land mobile network) list supported by this AMF
      plmnSupportList:
        - plmnId: # Public Land Mobile Network ID, <PLMN ID> = <MCC><MNC>
            mcc: 208 # Mobile Country Code (3 digits string, digit: 0~9)
            mnc: 93 # Mobile Network Code (2 or 3 digits string, digit: 0~9)
          snssaiList: # the S-NSSAI (Single Network Slice Selection Assistance Information) list supported by this AMF
            - sst: 1 # Slice/Service Type (uinteger, range: 0~255)
              sd: "010203" # Slice Differentiator (3 bytes hex string, range: 000000~FFFFFF)
            - sst: 1 # Slice/Service Type (uinteger, range: 0~255)
              sd: "112233" # Slice Differentiator (3 bytes hex string, range: 000000~FFFFFF)

      # the DNN (Data Network Name) list supported by this AMF
      supportDnnList:
        - internet
        - internet2
      nrfUri: http://127.0.0.10:8000 # a valid URI of NRF
      nrfCertPem: cert/nrf.pem # NRF Certificate

      # NAS Security Configuration
      # the priority of integrity algorithms
      # the priority of ciphering algorithms
      security:
        integrityOrder:
          - NIA2
          # - NIA0
        cipheringOrder:
          - NEA0
          - NEA2

      # Network Name Information
      networkName:
        full: free5GC
        short: free

      # Optional NGAP Information Elements (IE)
      ngapIE:
        mobilityRestrictionList: # Mobility Restriction List IE, refer to TS 38.413
          enable: true # append this IE in related message or not
        maskedIMEISV: # Masked IMEISV IE, refer to TS 38.413
          enable: true # append this IE in related message or not
        redirectionVoiceFallback: # Redirection Voice Fallback IE, refer to TS 38.413
          enable: false # append this IE in related message or not

      # Optional NAS Information Elements (IE)
      nasIE:
        networkFeatureSupport5GS: # 5gs Network Feature Support IE, refer to TS 24.501
          enable: true # append this IE in Registration accept or not
          length: 1 # IE content length (uinteger, range: 1~3)
          imsVoPS: 0 # IMS voice over PS session indicator (uinteger, range: 0~1)
          emc: 0 # Emergency service support indicator for 3GPP access (uinteger, range: 0~3)
          emf: 0 # Emergency service fallback indicator for 3GPP access (uinteger, range: 0~3)
          iwkN26: 0 # Interworking without N26 interface indicator (uinteger, range: 0~1)
          mpsi: 0 # MPS indicator (uinteger, range: 0~1)
          emcN3: 0 # Emergency service support indicator for Non-3GPP access (uinteger, range: 0~1)
          mcsi: 0 # MCS indicator (uinteger, range: 0~1)
      t3502Value: 720  # timer value (seconds) at UE side
      t3512Value: 3600 # timer value (seconds) at UE side
      non3gppDeregTimerValue: 3240 # timer value (seconds) at UE side
      # retransmission timer for paging message
      t3513:
        enable: true     # true or false
        expireTime: 6s   # default is 6 seconds
        maxRetryTimes: 4 # the max number of retransmission
      # retransmission timer for NAS Deregistration Request message
      t3522:
        enable: true     # true or false
        expireTime: 6s   # default is 6 seconds
        maxRetryTimes: 4 # the max number of retransmission
      # retransmission timer for NAS Registration Accept message
      t3550:
        enable: true     # true or false
        expireTime: 6s   # default is 6 seconds
        maxRetryTimes: 4 # the max number of retransmission
      # retransmission timer for NAS Configuration Update Command message
      t3555:
        enable: true     # true or false
        expireTime: 6s   # default is 6 seconds
        maxRetryTimes: 4 # the max number of retransmission
      # retransmission timer for NAS Authentication Request/Security Mode Command message
      t3560:
        enable: true     # true or false
        expireTime: 6s   # default is 6 seconds
        maxRetryTimes: 4 # the max number of retransmission
      # retransmission timer for NAS Notification message
      t3565:
        enable: true     # true or false
        expireTime: 6s   # default is 6 seconds
        maxRetryTimes: 4 # the max number of retransmission
      # retransmission timer for NAS Identity Request message
      t3570:
        enable: true     # true or false
        expireTime: 6s   # default is 6 seconds
        maxRetryTimes: 4 # the max number of retransmission
      locality: area1 # Name of the location where a set of AMF, SMF, PCF and UPFs are located

      # set the sctp server setting <optinal>, once this field is set, please also add maxInputStream, maxOsStream, maxAttempts, maxInitTimeOut
      sctp:
        numOstreams: 3 # the maximum out streams of each sctp connection
        maxInstreams: 5 # the maximum in streams of each sctp connection
        maxAttempts: 2 # the maximum attempts of each sctp connection
        maxInitTimeout: 2 # the maximum init timeout of each sctp connection
      defaultUECtxReq: false # the default value of UE Context Request to decide when triggering Initial Context Setup procedure

    logger: # log output setting
      enable: true # true or false
      level: info # how detailed to output, value: trace, debug, info, warn, error, fatal, panic
      reportCaller: false # enable the caller report or not, value: true or false

</div>

## UPF1 and UPF2 (docker)

![](images/165-3.png)

We use un Template en GNS3 called “gitunican-UPF-free5gc”.

The template includes 2 interfaces, first connecting to CN
(with IP 10.10.10.20/24 for UPF1 and 10.10.10.21/24 for UPF2), and second
one connects NAT1 and NAT2, our 2 DNN.

**upfcfg.yaml**  for UPF2 is equal than
UPF1(10.10.10.20), but changing the IP of UPF2 (10.10.10.21):

<div class="codebox">

    version: 1.0.3
    description: UPF1 initial local configuration

    # The listen IP and nodeID of the N4 interface on this UPF (Can't set to 0.0.0.0)
    pfcp:
      addr: 10.10.10.20
    #0.0.0.0 # IP addr for listening
      nodeID: 10.10.10.20 # External IP or FQDN can be reached
      retransTimeout: 1s # retransmission timeout
      maxRetrans: 3 # the max number of retransmission

    gtpu:
      forwarder: gtp5g
      # The IP list of the N3/N9 interfaces on this UPF
      # If there are multiple connection, set addr to 0.0.0.0 or list all the addresses
      ifList:
        - addr: 10.10.10.20
    #10.10.10.155
          type: N3
          # name: upf.5gc.nctu.me
          # ifname: gtpif
          # mtu: 1400

    # The DNN list supported by UPF
    dnnList:
      - dnn: internet # Data Network Name
        cidr: 10.60.0.0/16 # Classless Inter-Domain Routing for assigned IPv4 pool of UE
        # natifname: eth0

    logger: # log output setting
      enable: true # true or false
      level: info # how detailed to output, value: trace, debug, info, warn, error, fatal, panic
      reportCaller: false # enable the caller report or not, value: true or false

</div>

To route traffic to internet or internet2, we use “uerouting”,
pconfiguring uerouting.yaml, with the next content:

<div class="codebox">

    info:
      version: 1.0.7
      description: UE Routing Configuration

    ueRoutingInfo:
      UE1:
        members:
          - imsi-208930000000001
        topology:
          - A: gNB1
            B: UPF1
        specificPath:
          - dest: 10.60.0.0/16
            path: [UPF1]
      UE2:
        members:
          - imsi-208930000000002
        topology:
          - A: gNB1
            B: UPF2
        specificPath:
          - dest: 10.61.0.0/16
            path: [UPF2]

</div>

## UERANSIM (GNB + UE1 + UE2)

![](images/165-4.png)![](images/165-5.png)![](images/165-6.png)

The docker containers we will use for GNB and UE are the same as
included configuration files, those are the same as [Autoconfigurar Docker
UERANSIM](ning_-_Configuraciones_GNS3--(Scenario2)_Free5GC(VM)_+_UPF(docker)_+_UERANSIM_GNB(docker)-UE(docker)--Autoconfigurar_Docker_UERANSIM_164.html)

The configuration is identical, and you'd only have to run both Dockers
and start GNB and UE.

The UE2 file becomes:

<div class="codebox">

    # IMSI number of the UE. IMSI = [MCC|MNC|MSISDN] (In total 15 digits)
    supi: 'imsi-208930000000002'
    # Mobile Country Code value of HPLMN
    mcc: '208'
    # Mobile Network Code value of HPLMN (2 or 3 digits)
    mnc: '93'
    # SUCI Protection Scheme : 0 for Null-scheme, 1 for Profile A and 2 for Profile B
    protectionScheme: 0
    # Home Network Public Key for protecting with SUCI Profile A
    homeNetworkPublicKey: '5a8d38864820197c3394b92613b20b91633cbd897119273bf8e4a6f4eec0a650'
    # Home Network Public Key ID for protecting with SUCI Profile A
    homeNetworkPublicKeyId: 1
    # Routing Indicator
    routingIndicator: '0000'

    # Permanent subscription key
    key: '8baf473f2f8fd09487cccbd7097c6862'
    # Operator code (OP or OPC) of the UE
    op: '8e27b6af0e692e750f32667a3b14605d'
    # This value specifies the OP type and it can be either 'OP' or 'OPC'
    opType: 'OP'
    # Authentication Management Field (AMF) value
    amf: '8000'
    # IMEI number of the device. It is used if no SUPI is provided
    imei: '356938035643804'
    # IMEISV number of the device. It is used if no SUPI and IMEI is provided
    imeiSv: '4370816125816152'

    # List of gNB IP addresses for Radio Link Simulation
    gnbSearchList:
      - 10.10.10.7 #127.0.0.1

    # UAC Access Identities Configuration
    uacAic:
      mps: false
      mcs: false

    # UAC Access Control Class
    uacAcc:
      normalClass: 0
      class11: false
      class12: false
      class13: false
      class14: false
      class15: false

    # Initial PDU sessions to be established
    sessions:
      - type: 'IPv4'
        apn: 'internet2'
        slice:
          sst: 0x01
          sd: 0x010203

    # Configured NSSAI for this UE by HPLMN
    configured-nssai:
      - sst: 0x01
        sd: 0x010203

    # Default Configured NSSAI for this UE
    default-nssai:
      - sst: 0x01
        sd: 0x010203

    # Supported integrity algorithms by this UE
    integrity:
      IA1: true
      IA2: true
      IA3: true

    # Supported encryption algorithms by this UE
    ciphering:
      EA1: true
      EA2: true
      EA3: true

    # Integrity protection maximum data rate for user plane
    integrityMaxRate:
      uplink: 'full'
      downlink: 'full'

</div>

IMPORTANT: UE2 configuration file has to specify
that DNN is internet2

You must register the second user using Webconsole.
Create a new profile with the following info (remmember to
change nssaid to "internet2")

![](images/165-7.png)

![](images/165-8.png)

![](images/165-9.png)

![](images/165-10.png)

![](images/165-11.png)

Once connected, remember that you have to configure the outgoing 
route for each UE though uesimtun0 interface.

NOTA.- If it does not work ping 8.8.8.8 from UE, check
the default route to UPF too, going through NAT... Check 
PING to other IP

Capture into 10.10.10.20 of UPF1 and do ping to 8.8.8.8
from UE1:

![](images/165-12.png)

Capture into 10.10.10.21 of UPF2 and do ping to
193.144.186.1 from UE2:

![](images/165-13.png)
