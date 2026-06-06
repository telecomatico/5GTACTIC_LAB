<div class="page">

# (Scenario 4) Scenario 3 + 2 slices

\

# (Scenario 4) 2 Slice - Free5GC(VM)+2UPF(docker)+UERANSIM GNB(docker)/UE(docker)

The topology is the same as in scenario 3, ya que la única
diferencia es que se van a definir dos slices diferentes, una en cada
UPF. In UPF1 la 010203 which provides access a “internet”, y en el UPF2 la
112233, which provides access a “internet2”.

![](images/167-1.png)

The modifications are minimal.

## PREVIOUS STEP: activar GTP5G en GNS3 VM HV

y activar FORWARDING

We must ensure to repeat the steps indicated in [(Scenario 3)
Free5GC(VM)+2UPF(docker)+UERANSIM
GNB(docker)/UE(docker)](5GTACTIC--GNS3--Training_-_Configuraciones_GNS3--(Scenario_3)_Free5GC(VM)+2UPF(docker)+UERANSIM_GNB(docker)-UE(docker)_165.html)

## Core: FREE5GC

![](images/167-2.png)

The CORE network is based on the template named Free5GC_GNS3.

We replicate the initial process described in [(Scenario2)
Free5GC(VM) + UPF(docker) + UERANSIM
GNB(docker)/UE(docker)](5GTACTIC--GNS3--Training_-_Configuraciones_GNS3--(Scenario2)_Free5GC(VM)_+_UPF(docker)_+_UERANSIM_GNB(docker)-UE(docker)_161.html),
until reaching the SMF configuration changes.

We can start del smfcfg.yaml del Scenario 3, modificando las partes
concretas del UPF2:

- We add the new sNssai: **<span style="color:#ff9d00;">-</span>**
  sNssai**<span style="color:#ff9d00;">:</span>**

<div class="codebox">

            sst: 1
            sd: 112233
          dnnInfos:
            - dnn: internet
              dns:
                ipv4: 8.8.8.8
            - dnn: internet2
              dns:
                ipv4: 8.8.8.8

</div>

- We modify UPF2 so that it only serves al nuevo slice:

<div class="codebox">

          UPF2:
            type: UPF
            nodeID: 10.10.10.21
            addr: 10.10.10.21
            sNssaiUpfInfos:
              - sNssai:
                  sst: 1
                  sd: 112233
                dnnUpfInfoList:
                  - dnn: internet2
                    pools:
                      - cidr: 10.61.0.0/16
                    staticPools: []
            interfaces:
              - interfaceType: N3
                endpoints:
                  - 10.10.10.21
                networkInstances:
                  - internet2

</div>

El fichero **smcfg.yaml** completo es el siguiente:

<div class="codebox">

    info:
      version: 1.0.7
      description: SMFConfigurationScenario3

    configuration:
      smfName: SMF

      sbi:
        scheme: http
        registerIPv4: 127.0.0.2
        bindingIPv4: 127.0.0.2
        port: 8000
        tls:
          key: cert/smf.key
          pem: cert/smf.pem

      serviceNameList:
        - nsmf-pdusession
        - nsmf-event-exposure
        - nsmf-oam

      snssaiInfos:
        - sNssai:
            sst: 1
            sd: 010203
          dnnInfos:
            - dnn: internet
              dns:
                ipv4: 8.8.8.8
            - dnn: internet2
              dns:
                ipv4: 8.8.8.8
        - sNssai:
            sst: 1
            sd: 112233
          dnnInfos:
            - dnn: internet
              dns:
                ipv4: 8.8.8.8
            - dnn: internet2
              dns:
                ipv4: 8.8.8.8

      plmnList:
        - mcc: 208
          mnc: 93

      locality: area1

      pfcp:
        nodeID: 10.10.10.155
        listenAddr: 10.10.10.155
        externalAddr: 10.10.10.155

      userplaneInformation:
        upNodes:
          gNB1:
            type: AN
            an_ip: 10.10.10.7

          UPF1:
            type: UPF
            nodeID: 10.10.10.20
            addr: 10.10.10.20
            sNssaiUpfInfos:
              - sNssai:
                  sst: 1
                  sd: 010203
                dnnUpfInfoList:
                  - dnn: internet
                    pools:
                      - cidr: 10.60.0.0/16
                    staticPools: []
            interfaces:
              - interfaceType: N3
                endpoints:
                  - 10.10.10.20
                networkInstances:
                  - internet

          UPF2:
            type: UPF
            nodeID: 10.10.10.21
            addr: 10.10.10.21
            sNssaiUpfInfos:
              - sNssai:
                  sst: 1
                  sd: 112233
                dnnUpfInfoList:
                  - dnn: internet2
                    pools:
                      - cidr: 10.61.0.0/16
                    staticPools: []
            interfaces:
              - interfaceType: N3
                endpoints:
                  - 10.10.10.21
                networkInstances:
                  - internet2

        links:
          - A: gNB1
            B: UPF1
          - A: gNB1
            B: UPF2

    #  ueRoutingFile: ./config/uerouting.yaml
    #  urrPeriod: 10
    #  urrThreshold: 1000000
    #  requestedUnit: 1000000
      ulcl: false

      t3591:
        enable: true
        expireTime: 16s
        maxRetryTimes: 3

      t3592:
        enable: true
        expireTime: 16s
        maxRetryTimes: 3

      nrfUri: http://127.0.0.10:8000
      nrfCertPem: cert/nrf.pem
      urrPeriod: 30
      urrThreshold: 500000
      requestedUnit: 1000

    logger:
      enable: true
      level: info
      reportCaller: false

</div>

## UERANSIM (GNB + UE1 + UE2)

![](images/167-3.png)![](images/167-4.png)![](images/167-5.png)

The docker containers we will use para el GNB y el UE son los que copian los
ficheros de configuración, es decir los de [Autoconfigurar Docker
UERANSIM](ning_-_Configuraciones_GNS3--(Scenario2)_Free5GC(VM)_+_UPF(docker)_+_UERANSIM_GNB(docker)-UE(docker)--Autoconfigurar_Docker_UERANSIM_164.html)

The configuration is identical, so the only remaining step is arrancar los Docker
y lanzar el GNB y el UE1.

For UE2 we must modify el fichero de configuración para que
utilice un nuevo usuario que utilice el slice 112233.

Si partimos del fichero del UE2 del escenario 3, basta con modificar los
slice:

<div class="codebox">

    # Initial PDU sessions to be established
    sessions:
      - type: 'IPv4'
        apn: 'internet2'
        slice:
          sst: 0x01
          sd: 0x112233

    # Configured NSSAI for this UE by HPLMN
    configured-nssai:
      - sst: 0x01
        sd: 0x112233

    # Default Configured NSSAI for this UE
    default-nssai:
      - sst: 0x01
        sd: 0x112233

</div>

Arrancamos el UE2 con este nuevo fichero "free5gc-UE3.yaml.

Hay que registrar a este tercer usuario, por lo que en el Webconsole
we create a new profile

al que we assign un nuevo IMEI, y we modify el S-NSSAI para que use el
slice SD 112233 y el DNN "internet2"

</div>
