# jambonz
[jambonz](https://jambonz.org) is an open-source CPaaS (Communication Platform as a Service) specifically designed for Communication Service Providers.  

To add this Helm chart repository to your Helm client:

```bash
helm repo add jambonz https://jambonz.github.io/helm-charts/
```

Having installed it, you can list its charts, of which there is just one:

```bash
$ helm search repo jambonz
NAME           	CHART VERSION	APP VERSION	DESCRIPTION
jambonz/jambonz	0.1.24       	0.8.1      	open source CPaaS for communication service pro...
```

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
By default, since 0.1.26 of this helm chart, everything is installed into a single namespace.  However, you can optionally install the components into three different namespaces if you like, e.g.:
- `db`: for the database workloads
- `monitoring`: for the monitoring workloads
- `jambonz`: for the core jambonz call, media, and api processing.

If you don't want to install everything into a single namespace, then set the `global.db.namespace` and `global.monitoring.namespace` in values.yaml to the namespaces that you want to use for these sub-charts, and also set `global.db.createNamespace` and `global.monitoring.createNamespace` to true.

## Installing the chart

Here is example of installing the main chart as well as the db and monitoring sub-charts into a single namespace named 'jambonz'.  The FQDNs for the ingress controllers created for the web portal, the API, the grafana and homer portals are also provided on the command line.  

The cloud provider where the Kubernetes cluster is running (google, in this case) is also provided as a command line argument.  (This is needed to properly configure the [drachtio](https://drachtio.org) SIP element)

```bash
 helm install --namespace=jambonz \
 --generate-name --create-namespace \
 --set "monitoring.grafana.hostname=grafana.example.com" \
 --set "monitoring.homer.hostname=homer.example.com" \
--set "webapp.hostname=portal.example.com" \
--set "api.hostname=api.example.com" \
--set cloud=gcp \
jambonz/jambonz
 ```

Note that all of the above command line values are required variables.

And here is an example of installing into 3 different namespaces:
```bash
helm install --namespace=jambonz \
--generate-name --create-namespace \
--set "global.db.createNamespace=true" \
--set "global.db.namespace=db" \
--set "global.monitoring.createNamespace=true" \
--set "global.monitoring.namespace=monitoring" \
--set "monitoring.grafana.hostname=grafana.example.com" \
--set "monitoring.homer.hostname=homer.example.com" \
--set "webapp.hostname=portal.example.com" \
--set "api.hostname=api.example.com" \
--set cloud=gcp \
jambonz/jambonz
 ```

Once you have installed the helm chart for the first, it will take a bit for all components to be downloaded, installed and transition to the running state.  This is because the mysql database schema will be created and seeded with initial data, any many of the Pods will wait for the database to become available (via an initContainer) before starting their containers.

Once the system is up and running, you will want to query the 4 ingress controllers to view the the IP addresses that have been assigned.

```bash
$ kubectl -n jambonz get ingress
NAME         CLASS    HOSTS                ADDRESS          PORTS   AGE
api-server   <none>   api.example.com      34.117.50.187    80      6m3s
webapp       <none>   portal.example.com   34.102.246.186   80      6m3s

$ kubectl -n monitoring  get ingress
NAME           CLASS    HOSTS                 ADDRESS         PORTS   AGE
grafana        <none>   grafana.example.com   35.201.68.117   80      6m10s
homer-webapp   <none>   homer.example.com     35.241.2.62     80      6m10s
```

Once you have done this you will want to create associated the DNS records in your DNS provider so that you can access these portals.

## Jaeger (opentelemetry)

By default, open telemetry spans are stored in cassandra and jaeger-collector and jaeger-query are used to save and retrieve the spans, respectively.  If you prefer to run a simpler jaeger-all-in-one deployment with spans stored in memory, set `global.cassandra.enabled` to false.  Please note that if you do so historical spans will be lost every time you restart jaeger.

## Using Kong API Gateway

Run:
```bash
helm repo add kong https://charts.konghq.com
helm install kong kong/kong -n <namespace> --set ingressController.installCRDs=false
```
Check pod status, and make sure the kong-kong-{ID} pod is running:
```bash
kubectl get pods -n <namespace> | grep kong
```

Update values.yaml
```bash
  kong:
    enabled: true
    useHostnames: true 
```
Set enabled to true to use Kong.


useHostnames set to true will use the hostnames already created for the different services.
Either create a wildcard for your hostnames

useHostnames set to false will us the Kong IP address with prefixes.
```bash
<IP>/api
<IP>/webapp
<IP>/grafana
<IP>/homer
```

## Enable Call Recording

There are two ways to record calls, and they can be simultaneously enabled or disabled.
- From the portal, you can enable call recordings and listen to the recordings. Recordings are saved to an AWS S3 bucket and this must be enabled at the account level for every account that wishes to record calls.  You may record all calls for the account, or only record calls for specified applications.  This is intended to be a user-facing record option where the user is providing their own AWS S3 bucket and credentials for storage.
- You can use rtpengine to globally record all calls handled by the system.

### Call recording in the portal
By default, call recording in the portal is enabled.  To disable it, set `webapp.disableRecording` to true.

### rtpengine recording

To enable the recording of calls, we need an image of `jambonz/rtpengine` with the `libpcap-dev` package installed.

#### Generate a new image

If none is available, then create one with a Dockerfile like so:

``` Dockerfile
FROM jambonz/rtpengine:0.1.3 as base

RUN touch /var/lib/dpkg/status
RUN mkdir /var/lib/dpkg/updates
RUN mkdir /var/lib/dpkg/info
RUN apt-get update
RUN apt-get --assume-yes install libpcap-dev
```

#### Configurations

The following parameters are available to configure:

|Name|Description|Default Value|
|----|-----------|-----|
|global.recordings.enabled|Enable the recordings on rtpengine|`false`|
|rtpengine.recordings.pvc|PVC for the recordings|`recordings-shared-volume`|
|rtpengine.recordings.dir|Mounted Directory in the pod for storing the recordings|`/recordings`|
|rtpengine.recordings.method|Method for recording|`pcap`|

Update `values.yaml` and set `global.recordings.enabled` to `true` to use the recordings. Ensure that the PVC is in the same namespace as the `jambonz-sbc-rtp` (defaults to `jambonz`).

``` yaml
global:
  recordings:
    enabled: true

rtpengine:
  image: # change the image of rtpengine to use one with the libpcap-dev package
```


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
|db.affinity|set the affinity structure for the db subchart resources||
|db.storageClassName|set the `storageClassName` for the db subchart StatefulSet resources||
|monitoring.enabled|if true, install the monitoring sub-chart|true|
|monitoring.affinity|set the affinity structure for the monitoring subchart resources||
|monitoring.storageClassName|set the `storageClassName` for the monitoring subchart StatefulSet resources||
|global.affinity|set the affinity structure for the base chart resources||
|global.recordings.enabled|Enable call audio recordings on SBC-RTP|`false`|
|cloud|(required) name of cloud provider, must be one of azure, aws, digitalocean, gcp, or none|""|
|jambonz.clusterId|a short identifier for the cluster|""|
|sbc.sip.nodeSelector.label|label name used to select sip node pool|"voip-environment"|
|sbc.sip.nodeSelector.value|label value used to select sip node pool|"sip"|
|sbc.sip.toleration|taint assigned to sip node pool|"sip"|
|sbc.rtp.nodeSelector.label|label name used to select sip node pool|"voip-environment"|
|sbc.rtp.nodeSelector.value|label value used to select sip node pool|"rtp"|
|sbc.rtp.toleration|taint assigned to sip node pool|"rtp"|
|sbc.rtp.storageClassName|set the `storageClassName` for the SBC-RTP StatefulSet ||
|sbc.rtp.serviceAccountName|service account name to be used|""|
|sbc.rtp.extraInitContainers|list of initContainers to add to the sbc-rtp definition of initContainers||
|sbc.rtp.extraContainers|list of sidecars to add to the sbc-rtp definition of containers||
|sbc.rtp.extraVolumes|list of volumes to add to the sbc-rtp definition of volumes||
|sbc.rtp.statefulset.enabled|use a StatefulSet instead of DaemonSet for the sbc-rtp|`false`|
|sbc.rtp.statefulset.replicas|number of replicas for the StatefulSet|`1`|
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
|rtpengine.recordings.pvc|PVC for the recordings|`recordings-shared-volume`|
|rtpengine.recordings.dir|Mounted Directory in the pod for storing the recordings|`/recordings`|
|rtpengine.recordings.method|Method for recording|`pcap`|
|rtpengine.recordings.storage|Amount of storage for the recordings volume|`10Gi`|
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
|sbcInbound.image|sbc-inbound image|jambonz/sbc-inbound:latest|
|sbcInbound.imagePullPolicy|sbc-inbound image pull policy|"Always"|
|sbcOutbound.drachtioPort|port to listen on for outbound connections from drachtio|"4000"|
|sbcOutbound.image|sbc-outbound image|jambonz/sbc-outbound:latest|
|sbcOutbound.imagePullPolicy|sbc-outbound image pull policy|"Always"|
|sbcOutbound.drachtioPort|port to listen on for outbound connections from drachtio|"4000"|
|sbcSipSidecar.image|sbc-sip-sidecar image|jambonz/sbc-sip-sidecar:latest|
|sbcSipSidecar.imagePullPolicy|sbc-sip-sidecar image pull policy|"Always"|
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


## Configuring jambonz to use sip over websockets

### Prerequisites
You need to have an  Certificate Issuer installed on your kubernetes cluster. If you don't, you can install [cert-manager](https://cert-manager.io) which allows you to obtain and renew automatically valid Let's Encrypt certificates.

### Request Certificate
To obtain certificates, you just need to create resources of type [Certificate](https://cert-manager.io/docs/usage/certificate/) in the namespace of your choice

```yaml
    apiVersion: cert-manager.io/v1
    kind: Certificate
    metadata:
      name: drachtio-certificate
      namespace: jambonz
    spec:
      # Secret names are always required.
      secretName: drachtio-certs
      duration: 2160h # 90d
      renewBefore: 360h # 15d
      usages:
        - server auth
        - client auth
      # At least one of a DNS Name, URI, or IP address is required.
      dnsNames:
        - example.com # Change this with your domain name
        - www.example.com # Change this with your domain name
      issuerRef:
        name: ca-issuer # name of your issuer as defined at step 1
        kind: Issuer #or ClusterIssuer
```
As soon as you apply this CRD, you will have a new CertificateRequest created and a secret (drachtio-certs) created in jambonz  namespace.

### Use your certs

To use the created certificates on drachtio server, set the config `sbc.sip.ssl.enabled` to `true`:

```yaml
sbc:
  sip:
    ssl:
      enabled: true
```

This will cause drachtio to add a listener for sip over wss on port 8443 and use the provided TLS certificate and key to authenticate requests.

> Note: you must be running drachtio server version 0.8.17-rc1 or later for support of the WSS_SIP env variable.

### Multi-Zone

A multi-zone deployment of jambonz is possible, but some changes need to be made in order to support that.

#### Change the default Storage Class

If your default Storage Class doesn't support a multi-zone claim by default (like it doesn't have the `volumeBindingMode: WaitForFirstConsumer` property), or you need to change it to other one, set the `storageClassName` value accordingly:

```yml
db:
  storageClassName: "regional-storage-class"

monitoring:
  storageClassName: "another-regional-storage-class"
```

#### Affinity

The Affinity is a kubernetes feature that expands the `nodeSelector` capabilities. To configure a jambonz cluster to be deployed to a set of zones, configure the `affinity` like so:

```yml
global:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
            - key: topology.kubernetes.io/zone
              operator: In
              values:
              - europe-west1-b
              - europe-west1-c
```

The `db` and `monitoring` subcharts have both a separated affinity value that can be set:

```yml
db:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
            - key: topology.kubernetes.io/zone
              operator: In
              values:
              - europe-west1-b

monitoring:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
            - key: topology.kubernetes.io/zone
              operator: In
              values:
              - europe-west1-d
```

### Add extra containers or volumes to the SBC-RTP

It may be required to do some processing on the data produced by the RTP service, like the recordings. For these scenarios, another container needs to run along the rtp and share a volume or a secret. Add the following configurations to your yaml file like you would populate a kubernetes deployment:

```yaml
sbc:
  rtp:
    #nodeSelector: 
    #  label: rtp
    #  value: "true"
    #toleration: rtp
    extraInitContainers:
    - name: init-processor
      image: my-service/init-processor:v1.0.0
    extraContainers:
    - name: recordings-processor
      env:
      - name: SERVER_PORT
        value: "8080"
      - name: GOOGLE_APPLICATION_CREDENTIALS
        value: /app/secrets/gcp.serviceaccount.json
      image: my-service/recordings-processor:v2.0.0
      ports:
      - containerPort: 8080
        hostPort: 8080
        protocol: TCP
      resources:
        requests:
          cpu: 50m
          memory: 300Mi
      volumeMounts:
      - mountPath: /app/secrets
        mountPropagation: None
        name: secrets-gcp
        readOnly: true
      - mountPath: /usr/src/app/cache
        mountPropagation: None
        name: cache
      - mountPath: /recordings # if enabled with global.recordings.enabled
        name: recordings
    - name: another-processor
      image: my-service/another-processor:v3.0.0
    extraVolumes:
    - name: secrets-gcp
      secret:
        defaultMode: 420
        optional: false
        secretName: gcp-secret
    - emptyDir: {}
      name: cache
```
