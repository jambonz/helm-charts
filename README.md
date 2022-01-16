# jambonz
[jambonz](https://jambonz.org) is an open-source CPaaS (Communication Platform as a Service) specifically designed for Communication Service Providers.  

## Introduction
This chart bootstraps a jambonz deployment on a [Kubernetes](https://kubernetes.io) cluster using the [Helm](https://helm.sh) package manager.

### Special requirements for the Kubernetes cluster
You will need to have at least one (two are recommended) special node pools created in the Kubernetes cluster where the [SIP](https://datatracker.ietf.org/doc/html/rfc3261) and [RTP](https://datatracker.ietf.org/doc/html/rfc3550) processing components of the jambonz SBC will run.  The reason for this is that there are currently no Kubernetes ingress controllers that are capable of managing SIP traffic.

Two node pools are recommended so that the SIP processing can be handled in one, and RTP in the other.  For a production cluster it is desirable to allow the RTP processing nodes to scale independently of the SIP processing, because if we are continually adding or removing SIP processing nodes we will need to be updating our carriers with those addresses as they are added or removed.  Since the RTP processing is more computationally expensive than SIP, it is best to allow the RTP to scale independently, and this requires it to be in its own node pool.

Suffice it to say that two node pools are recommended to be created in addition to the default node pool, but this chart does allow you to deploy SIP and RTP elements into single node pool if desired.

#### SIP Node Pool configuration

Create the node pool for sip with the following labels and taints:
- label: voip-environment=sip
- taint: sip=true

> Note: if you choose to use a different label or taint, update the associated global values in the values.yaml file accordingly

Associate a firewall rule with this node pool allowing traffic into instances in this node pool as follows:
- 5060/udp
- 5060/tcp
- 5061/tls
- 8443/tcp

#### RTP Node Pool configuration
Create the node pool for rtp with the following labels and taints:
- label: voip-environment=rtp
- taint: rtp=true

> Note: as above, if you choose to use a different label or taint, update the associated global values in the values.yaml file accordingly

Associate a firewall rule with this node pool allowing traffic into instances in this node pool as follows:
- 40000-60000/udp

> Note: if you want to create a single additional node pool rather than two, update the global values in the values.yaml to reference the label and taint in the single node pool, and open all of the ports above.

### Sub-charts
The database (redis+mysql) and monitoring (grafana+influxdb+telegraf+homer) components of jambonz are provided as sub-charts, which allow them to be installed as standalone charts if nececessary.

### Namespaces
This chart provides the option of installing everything into a single namespace, or installing into three different namespaces; e.g.
- `db`: for the database workloads
- `monitoring`: for the monitoring workloads
- `jambonz`: for the core jambonz call, media, and api processing.

If you don't want to install everything into a single namespace, then set the `global.dbNamespace` and `global.monitoringNamespace`

## Installing the chart

Here is example of installing the main chart into the "jambonz" namespace, the db subchart into the "db" namespace, and the monitoring subchart into the "monitoring" namespace.  The FQDNs for the ingress controllers created for the web portal, the API, the grafana and homer portals are also provided on the command line.  

The cloud provider where the Kubernetes cluster is running (google, in this case) is also provided as a command line argument.  (This is needed to properly configure the [drachtio](https://drachtio.org) SIP element)

```bash
 helm install --namespace=jambonz \
 --generate-name --create-namespace \
 --set "global.db.namespace=db" \
 --set "global.monitoring.namespace=monitoring" \
 --set "monitoring.grafana.hostname=grafana.example.com" \
 --set "monitoring.homer.hostname=homer.example.com" \
--set "webapp.hostname=portal.example.com" \
--set "api.hostname=api.example.com" \
--set cloud=gcp .
 ```

 Installing the chart into a single namespace:

 ```bash
  helm install --namespace=jambonz \
 --generate-name --create-namespace \
 --set "monitoring.grafana.hostname=grafana.example.com" \
 --set "monitoring.homer.hostname=homer.example.com" \
--set "webapp.hostname=portal.example.com" \
--set "api.hostname=api.example.com" \
--set cloud=gcp .
```

Note that all of the above command line values are required variables.

## Uninstalling the chart

```bash
helm uninstall -n <namespace> <release-name>
```

## Parameters

### Global parameters

|Name|Description|Value|
|----|-----------|-----|
|dbNamespace|namespace where db components will be installed; if not provided these components will be installed into the Release namespace|""|
|monitoringNamespace|namespace where monitoring components will be installed; if not provided these components will be installed into the Release namespace|""|

### Other parameters
|Name|Description|Value|
|----|-----------|-----|
|db.enabled|if true, install the db sub-chart|true|
|monitoring.enabled|if true, install the monitoring sub-chart|true|
|cloud|(required) name of cloud provider, must be one of azure, aws, digitalocean, gcp, or none|""|
|jambonz.clusterId|a short identifier for the cluster|""|
|jambonz.loglevel|jambonz loglevel, should be either info or debug|"info"|
|sbc.sip.nodeSelector.label|label name used to select sip node pool|"voip-environment"|
|sbc.sip.nodeSelector.value|label value used to select sip node pool|"sip"|
|sbc.sip.toleration|taint assigned to sip node pool|"sip"|
|sbc.rtp.nodeSelector.label|label name used to select sip node pool|"voip-environment"|
|sbc.rtp.nodeSelector.value|label value used to select sip node pool|"rtp"|
|sbc.rtp.toleration|taint assigned to sip node pool|"rtp"|
|stats.enabled|if set, jambonz apps write stats to telegraf|"1"|
|stats.host|telegraf service name|"telegraf"|
|stats.port|telegraf service listening port|"8125"|
|stats.protocol|stats protocol|"tcp"|
|stats.sampleRate|stats sample rate|"1"|
|rtpengine.image|image for rtpengine rtp media proxy|jambonz/rtpengine:latest|
|rtpengine.imagePullPolicy|rtpengine image pull policy|"Always"|
|rtpengine.sidecarImage|image for rtpengine sidecar container|jambonz/rtpengine-sidecar:latest|
|rtpengine.sidecarImagePullPolicy|rtpengine sidecar image pull policy|"Always"|
|rtpengine.dtmfLogPort|udp port that rtpengine sends dtmf events to|"22223"|
|rtpengine.logLevel|rtpengine log level (1-7)|"5"|
|rtpengine.homerId|id to apply to HEP traffic sent to homer from rtpengine|"11"|
|drachtio.image|drachtio image|drachtio/drachtio-server:latest|
|drachtio.imagePullPolicy|drachtio image pull policy|"Always"|
|drachtio.host|host of drachtio server|"127.0.0.1"|
|drachtio.port|port to connect to drachtio server|"9022"|
|drachtio.homerId|id to apply to HEP traffic sent to homer from drachtio|"10"|
|freeswitch.image|freeswitch image|drachtio/drachtio-freeswitch-mrf:latest|
|freeswitch.imagePullPolicy|freeswitch image pull policy|"Always"|
|mysql.image|mysql image|mysql:5.7|
|mysql.host|mysql service name|"mysql"|
|mysql.database|jambonz database|"jambones"|
|mysql.user|mysql user|"jambones"|
|mysql.storage|amount of storage to allocate for mysql database|"10Gi"|
|redis.image|redis image|redis:alpine|
|redis.host|redis service name|"redis"|
|redis.port|redis listening port|"6379"|
|api.image|jambonz-api-server image|jambonz/jambonz-api-server:latest|
|api.imagePullPolicy|jambonz-api-server image pull policy|"Always"|
|api.httpPort|http port to listen on|"3000"|
|api.hostname|(required) FQDN for api ingress controller|""|
|webapp.image|jambonz webapp image|jambonz/webapp:latest|
|webapp.imagePullPolicy|jambonz webapp image pull policy|"Always"|
|webapp.hostname|(required ) FQDN for webapp ingress controller|""|
|sbcOptionsHandler.image|sbc-options-handler image|jambonz/sbc-options-handler:latest|
|sbcOptionsHandler.imagePullPolicy|sbc-options-handler image pull policy|"Always"|
|sbcInbound.image|sbc-inbound image|jambonz/sbc-inbound:latest|
|sbcInbound.imagePullPolicy|sbc-inbound image pull policy|"Always"|
|sbcOutbound.drachtioPort|port to listen on for outbound connections from drachtio|"4000"|
|sbcOutbound.image|sbc-outbound image|jambonz/sbc-outbound:latest|
|sbcOutbound.imagePullPolicy|sbc-outbound image pull policy|"Always"|
|sbcOutbound.drachtioPort|port to listen on for outbound connections from drachtio|"4000"|
|sbcRegistrar.image|sbc-registrar image|jambonz/sbc-registrar:latest|
|sbcRegistrar.imagePullPolicy|sbc-registrar image pull policy|"Always"|
|sbcRegistrar.drachtioPort|port to listen on for outbound connections from drachtio|"4000"|
|sbcCallRouter.image|sbc-call-router image|jambonz/sbc-call-router:latest|
|sbcCallRouter.imagePullPolicy|sbc-call-router image pull policy|"Always"|
|sbcCallRouter.drachtioPort|port to listen on for http connections from drachtio|"3000"|
|featureServer.image|feature-server image|jambonz/feature-server:latest|
|featureServer.imagePullPolicy|feature-server image pull policy|"Always"|
|featureServer.httpPort|port to listen on for api requests|"3000"|
|feature.drachtioConnection|drachtio connection information|"localhost:9022:cymru"|
|feature.freeswitchConnection|freeswitch connection information|"localhost:8021:JambonzR0ck$"|
|sbcSip.loglevel|drachtio loglevel in sbc sip container, should be info or debug|"info"|
|sbcSip.sofiaLogLevel|sofia-sip log level for drachtio in sbc sip container, should be 1-9|"3|
|dbWaiter.image|image of sidecar mysql client app to use for testing database readiness|d3fk/kubectl:v1.18|



