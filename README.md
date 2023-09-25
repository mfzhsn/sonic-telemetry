# SONiC Telemetry

[SONiC](https://github.com/sonic-net) has been implemented with gRPC for streaming telemetry. This lab will provide you steps to enable gnmi and its associated tools to stream metrics.

## Tools Used

| Functions    | Tools Used | 
| -------- | ------- |
| Router/Switch  | [SONiC](https://github.com/sonic-net) on [Nokia IXR-7215](https://www.nokia.com/networks/ip-networks/service-router-linux-NOS/)    | 
| gnmi collector  | [gnmic](https://gnmic.openconfig.net)    | 
| TSDB  | [Prometheus](https://prometheus.io/)    | 
| Dashboard/UI  | [Grafana](https://grafana.com/)    | 

## Getting Started

This guide explains enabling the streaming telemetry in 2 parts.

### 1. Enabling gnmi service at SONiC Device

1. Fixing the Telemetry Container

The SONiC device has the Telemetry container which might be in exited state, Lets fix this first.

```
admin@sonic:~$ docker ps --all
CONTAINER ID   IMAGE                             COMMAND                  CREATED        STATUS        PORTS     NAMES
ddedbed28641   docker-snmp:latest                "/usr/local/bin/supe…"   2 months ago   Up 42 hours             snmp
66da3f6d9754   docker-sonic-telemetry:latest     "/usr/local/bin/supe…"   2 months ago   Exited (0) 10 minutes ago             telemetry
3748af29ce98   2334c214e07f                      "/usr/bin/docker_ini…"   2 months ago   Up 42 hours             dhcp_relay
ef78a48ffdc2   docker-platform-monitor:latest    "/usr/bin/docker_ini…"   2 months ago   Up 42 hours             pmon
fb672d2ffd84   docker-lldp:latest                "/usr/bin/docker-lld…"   2 months ago   Up 42 hours             lldp
6c433abdc45b   docker-fpm-frr:latest             "/usr/bin/docker_ini…"   2 months ago   Up 42 hours             bgp
f870efdfce7a   docker-router-advertiser:latest   "/usr/bin/docker-ini…"   2 months ago   Up 42 hours             radv
a14167f26de8   docker-syncd-mrvl:latest          "/usr/local/bin/supe…"   2 months ago   Up 42 hours             syncd
abcbd9b80db5   docker-teamd:latest               "/usr/local/bin/supe…"   2 months ago   Up 42 hours             teamd
5539e083dd64   docker-orchagent:latest           "/usr/bin/docker-ini…"   2 months ago   Up 42 hours             swss
60e51e9afb20   docker-acms:latest                "/usr/local/bin/supe…"   2 months ago   Up 42 hours             acms
8ffd722b83d0   docker-database:latest            "/usr/local/bin/dock…"   2 months ago   Up 42 hours             database
```

You can check telemetry logs at `/var/log/telemetry.log`, you can will that it is expecting certificates.


a. Generate Certs

SONiC will have baseline configurations for Telemetry/gnmi, and it will check for certs in the below path. Configurations can be checked either by running `show runningconfiguration all` or `cat /etc/sonic/config_db.json`

```
    "TELEMETRY": {
        "certs": {
            "ca_crt": "/etc/sonic/telemetry/dsmsroot.cer",
            "server_crt": "/etc/sonic/telemetry/streamingtelemetryserver.cer",
            "server_key": "/etc/sonic/telemetry/streamingtelemetryserver.key"
        },
``` 
**Generate certs**

b. Create a directory called `telemetry`

```
mkdir /etc/sonic/telemetry
```
Certificate

```
sudo openssl req -x509 -newkey rsa:4096 -keyout /etc/sonic/telemetry/dsmsroot.key   -out /etc/sonic/telemetry/dsmsroot.cer -sha256 -days 365 -nodes -subj '/CN=sonic-lab'
```
CSR

```
sudo openssl req -new -newkey rsa:4096 -nodes   -keyout /etc/sonic/telemetry/streamingtelemetryserver.key -out /etc/sonic/telemetry/streamingtelemetryserver.csr   -subj "/CN=sonic-lab"
```
Key

```
sudo openssl x509 -req -in /etc/sonic/telemetry/streamingtelemetryserver.csr   -CA /etc/sonic/telemetry/dsmsroot.cer -CAkey /etc/sonic/telemetry/dsmsroot.key   -CAcreateserial -out /etc/sonic/telemetry/streamingtelemetryserver.cer   -days 365 -sha512
```

Restart the telemetry container

```
sudo docker container restart telemetry
```

Verify that telemetry container should be up and running

```
admin@sonic:/etc/sonic$ docker ps
CONTAINER ID   IMAGE                             COMMAND                  CREATED        STATUS        PORTS     NAMES
ddedbed28641   docker-snmp:latest                "/usr/local/bin/supe…"   2 months ago   Up 42 hours             snmp
66da3f6d9754   docker-sonic-telemetry:latest     "/usr/local/bin/supe…"   2 months ago   Up 42 hours             telemetry
3748af29ce98   2334c214e07f                      "/usr/bin/docker_ini…"   2 months ago   Up 42 hours             dhcp_relay
ef78a48ffdc2   docker-platform-monitor:latest    "/usr/bin/docker_ini…"   2 months ago   Up 42 hours             pmon
fb672d2ffd84   docker-lldp:latest                "/usr/bin/docker-lld…"   2 months ago   Up 42 hours             lldp
6c433abdc45b   docker-fpm-frr:latest             "/usr/bin/docker_ini…"   2 months ago   Up 42 hours             bgp
f870efdfce7a   docker-router-advertiser:latest   "/usr/bin/docker-ini…"   2 months ago   Up 42 hours             radv
a14167f26de8   docker-syncd-mrvl:latest          "/usr/local/bin/supe…"   2 months ago   Up 42 hours             syncd
abcbd9b80db5   docker-teamd:latest               "/usr/local/bin/supe…"   2 months ago   Up 42 hours             teamd
5539e083dd64   docker-orchagent:latest           "/usr/bin/docker-ini…"   2 months ago   Up 42 hours             swss
60e51e9afb20   docker-acms:latest                "/usr/local/bin/supe…"   2 months ago   Up 42 hours             acms
8ffd722b83d0   docker-database:latest            "/usr/local/bin/dock…"   2 months ago   Up 42 hours             database
``` 

**Note** 

This is optional, You can change the port number and authnetication options at `/etc/sonic/config_db.json`.

In my case, my gnmi collector should be dailing in on port 57400 with insecure connection.

```
 "gnmi": {
            "client_auth": "false",
            "log_level": "2",
            "port": "57400"
        }
```


### 2. Deploying the gnmi collector and prometheus/grafana


