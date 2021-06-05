## 1. Rancher Deployment
This HOWTO Assumes that you have a working Rancher deployment, the size of which is determined by what parts of the demo you will be hosting in the cluster.

The detailed steps required to install Rancher are outside of the scope of this demo but can be found on the [Rancher Website](https://rancher.com/docs/rancher/v2.x/en/quick-start-guide/deployment/quickstart-manual-setup/)

### NIGNX KIC, Filebeat, Application Only
This configuration assumes you are running your Elasticsearch/Kibana installation outside of the cluster. 

This configuration can be run on a one node rancher cluster with a minimum of 2 vCPU, 10GB RAM, and 60GB disk. 

### NGINX KIC, Filebeat, Application, Elasticsearch, Kibana
This configuration runs all the necessary components to run this demo including the EFK cluster. 

This configuration requires at least three nodes, each with a minimum of 2 vCPU, 10GB RAM, and 60GB disk. 

----

## 2. Load Balancer Deployment
If you are running on a cloud that provides external load balancer services you can skip this step. However, if you are running a self-hosted cluster you will need to provide LB access to the NGINX KIC unless you want to expose your NGINX KIC via a NodePort. 

One simple way is to provide this access via the [MetalLB Load Balancer](https://metallb.universe.tf/)

The following steps can be followed to install MetalLB from manifests:

```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.6/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.6/manifests/metallb.yaml
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```

The final step is to apply the configuration; be sure to adjust your IP range to match your network requirements.

Create the configuration manifest:
```
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.212.101-192.168.212.111
```

Apply the configuration manifest:
```
kubectl apply -f config.yaml
```

Alternatively, you can update the `config.yaml` and run the [`deploy-metallb.sh`](./metallb/deploy-metallb.sh) script which performs the steps above.

----

## 3. Certificate Manager (Optional)
This demo works with self-signed certificates, but it can also use the K8 [cert-manager](https://cert-manager.io/docs/) to generate certificates to be used with the examples.

This example assumes you have a domain name, a DNS provider that supports the DNS01 or HTTPS01 challenge protocols, and that there is a cert-manager plugin for that provider. Unless you have direct internet access from your K8 cluster that can be used for HTTPS01 authentication, it is recommended that you use DNS01 challenges. 

For the examples below, we will be using the`zathras.io` domain, which has it's DNS records managed by [DNSimple](https://dnsimple.com).

### Deploy Cert Manager
The first step is to deploy the cert manager; at the time of this writing the 1.3.1 release is being used:

```
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.3.1/cert-manager.yaml
```

### Deploy Webhook (DNSimple)
This step will add in the necessary webhook for updating and generating your certificate, along with the data that will be used for Let's Encrypt. You will need to deploy the appropriate webhook for your provider along with your token and email address.

Add the repo:
```
helm repo add neoskop https://charts.neoskop.dev
```

Deploy the webook:
```
helm install cert-manager-webhook-dnsimple \
--namespace cert-manager --set dnsimple.token='XXXXX'\
--set clusterIssuer.production.enabled=true \
--set clusterIssuer.staging.enabled=true  \
--set clusterIssuer.email=qdzlug@gmail.com  \
neoskop/cert-manager-webhook-dnsimple 
```

----

## 4. Deploy Storage (Required for EFK)
To deploy the EFK stack you will require storage for persistent volumes.

### Using Longhorn
The easiest way to provide this storage is to deploy the [Longhorn](https://longhorn.io) application via the Rancher GUI. 
- Select the default project in your cluster.
- Navigate to Applications
- Click on "Launch"
- Select "Longhorn"
- Click on "Launch"

This can also be done via manifest:

```
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v0.8.0/deploy/longhorn.yaml
```

### Using NFS 
To deploy an NFS provisioner for persistent storage using either an existing NFS server or an NFS server running in docker please see [this project](https://github.com/qdzlug/k8-nfs)

---

## 5. Add Bitnami and Elastic Catalogs
For this deployment we will be using the Bitnami packaging of Elasticsearch and Kibana, and the official Elastic Filebeat package. Note that this demo requires a version => 7.13 in order to work properly.

*Note*: If you are only installing Filebeat (using an external Elasticsearch/Kibana deployment) you only need to install the Elastic catalog.

These catalogs can be added via the GUI.
- Select the default project in your cluster.
- Navigate to applications
- Click on "Manage Catalogs"
- Click on "Add Catalog"
- Input the following information for each catalog.
- Click "Add" 

### Bitnami
Name: bitnami
Catalog URL: https://charts.bitnami.com/bitnami
Scope: Global
Helm Version: 3

### Elastic
Name: elastic
Catalog URL: https://helm.elastic.co
Scope: Global
Helm Version: 3

The catalogs will refresh in the background and will be available on the application launch screen.

----

## 6. Deploy Elasticsearch
This is done from within the Rancher GUI. 
- Select the default project in your cluster.
- Navigate to Applications
- Click on "Launch"
- Select "Elasticsearch" (Ensure you are selecting the "Bitnami" chart, not the "Elastic" chart)
- Add the following answers to the chart in the provided boxes:
	- global.kibanaenabled: true
	- ingest.enabled: true
- Click on "Launch"

The values can also be added to the YAML editor as 
```
---
  global: 
    kibanaEnabled: "true"
  ingest: 
    enabled: "true"
```

Alternatively, you can read in the [`elasticsearch-values.yaml`](./elasticsearch-values.yaml) file from this repository.

----

## 7. Deploy Filebeat
In order to interpret the data from the NGINX KIC, we are making use of the Filebeat "autodiscover" feature along with hints. This selects the appropriate ingest pipeline for data processing. We will be adding an annotation to the NGINX KIC deployment to provide the necessary hints for our logging to work properly.

This is done from within the Rancher GUI. 
- Select the default project in your cluster.
- Navigate to Applications
- Click on "Launch"
- Select "Filebeat" (Ensure you are selecting the "Elastic" chart)
- Click on "Edit as YAML"
- Add the answers to the chart for the Elasticsearch cluster you are writing to (either cloud or local) from the sections below.
- Click on "Launch"

### To Use Elastic Cloud
Note that this configuration can be used for either a "deployment" or a "daemonset" simply by changing the top level key. As provided, this deploys a "daemonset" (one per node), but by changing that to "deployment" it will deploy one pod by default which can be adjusted by adding replicas.

Add the following to the "values" entry field. Alternatively, you can load the [filebeat-values-cloud.yaml](./filebeat/filebeat-values-cloud.yaml) file.

```
---
daemonset:
  enabled: true
  filebeatConfig:
    filebeat.yml: |
      filebeat.autodiscover:
        providers:
          - type: kubernetes
            hints.enabled: true
            hints.default_config:
              type: container
              paths:
                - /var/lib/docker/containers/${data.kubernetes.container.id}/*.log
      cloud.id: "cloud-id-token-goes-here"
      cloud.auth: "cloud-auth-goes-here"
```

### To Use Elasticsearch Hosts Directly

Note that this configuration can be used for either a "deployment" or a "daemonset" simply by changing the top level key. As provided, this deploys a "daemonset" (one per node), but by changing that to "deployment" it will deploy one pod by default which can be adjusted by adding replicas.

Add the following to the "values" entry field. Alternatively, you can load the [filebeat-values-k8.yaml](./filebeat/filebeat-values-k8.yaml) file.

```
---
daemonset:
  enabled: true
  filebeatConfig:
    filebeat.yml: |
      filebeat.autodiscover:
        providers:
          - type: kubernetes
            hints.enabled: true
            hints.default_config:
              type: container
              paths:
                - /var/lib/docker/containers/${data.kubernetes.container.id}/*.log
      output.elasticsearch:
        host: '${NODE_NAME}'
        hosts: 'elasticsearch-coordinating-only.elasticsearch.svc.cluster.local:9200'
```

----

## 8. Deploy NGINX OSS KIC
This same process can be used for either the NGINX OSS KIC or the NGINX Plus KIC. 

There are two key changes being made from the stock configuration:
1. Add the annotation `"co.elastic.logs/module": "nginx"` to both the pods and the service. This provides the necessary hint to the filebeat process to use the `nginx` module to handle the logs.
2. Add new logging to approxmiate the upstream nginx-ingress controller log format so it can be parsed by the elasticsearch ingest node. This format is:
	```
	{"log-format": "$remote_addr - $remote_user [$time_local] \"$request\" $status $body_bytes_sent \"$http_referer\" \"$http_user_agent\" $upstream_response_time $upstream_status \"$uri\" $request_length $request_time [$proxy_host] [] $upstream_addr $upstream_bytes_sent $upstream_response_time $upstream_status $request_id"}
	```
Deployment is done from within the Rancher GUI. 
- Select the default project in your cluster.
- Navigate to Applications
- Click on "Launch"
- Select "NGINX Ingress Controller" (Ensure you are selecting the "NGINX, Inc" chart)
- Click on "Edit as YAML"
- Add the answers to the chart for the NGINX KIC (either OSS or Plus) from the sections below.
- Click on "Launch"

### To Install NGINX OSS KIC

Add the following to deploy the NGINX OSS KIC:

```
controller:
  kind: deployment
  nginxplus: false
  image:
    repository: nginx/nginx-ingress
    tag: "1.11.1"
    pullPolicy: IfNotPresent
  config:
    name: nginx-config
    annotations: {}
    entries: {"log-format": "$remote_addr - $remote_user [$time_local] \"$request\" $status $body_bytes_sent \"$http_referer\" \"$http_user_agent\" $upstream_response_time $upstream_status \"$uri\" $request_length $request_time [$proxy_host] [] $upstream_addr $upstream_bytes_sent $upstream_response_time $upstream_status $request_id"}
  service:
    create: true
    type: LoadBalancer
    annotations: {"co.elastic.logs/module": "nginx"}
  pod:
    annotations: {"co.elastic.logs/module": "nginx"}
prometheus:
  create: true
  port: 9113
```

Alternatively, use the [values.yaml](./nginx-helm/values.yaml).

### To Install NGINX Plus

Add the following to deploy the NGINX Plus KIC. Note that you will need to edit the configuration to add repository and secret pull credentials. You will also need to ensure you have a secret with the registry credentials defined in your K8 environment that is accessable from the namespace the controller is deployed in.

```
---
  controller: 
    kind: "deployment"
    nginxplus: true
    appprotect: 
      enable: false
    image: 
      repository: "register.virington.com/nginx-ingress"
      tag: "1.11.1"
      pullPolicy: "IfNotPresent"
    config:
      name: nginx-config
      annotations: {}
      entries: {"log-format": "$remote_addr - $remote_user [$time_local] \"$request\" $status $body_bytes_sent \"$http_referer\" \"$http_user_agent\" $upstream_response_time $upstream_status \"$uri\" $request_length $request_time [$proxy_host] [] $upstream_addr $upstream_bytes_sent $upstream_response_time $upstream_status $request_id"}
    serviceAccount: 
      imagePullSecretName: "register-secret"
    defaultTLS: 
      secret: ""
    ingressClass: "nginx"
    useIngressClassOnly: false
    enableCustomResources: true
    globalConfiguration: 
      create: true
    watchNamespace: ""
    service: 
      create: true
      type: "LoadBalancer"
      annotations: {"co.elastic.logs/module": "nginx"}
      httpPort: 
        enable: true
        port: "80"
      httpsPort: 
        enable: true
        port: "443"
    pod:
      annotations: {"co.elastic.logs/module": "nginx"}
  imageDefault: true
  prometheus: 
    create: true
    port: "9113"
```

Alternatively, use the [values.yaml](./nginx-helm/values.yaml).

## 9. Deploy Test App
For testing purposes, we will be using the standard NGINX Cafe demo. This is for convenience; any application can be used.

Note that if you are planning on using the cert-manager as part of the Ingress process you will need to use the correct syntax in the deployment. 

### Deployment and Service
Use the [`cafe.yaml`](./examples/cafe.yaml) manifest. This can be applyed with `kubectl create -f ./cafe.yaml`.

### Ingress
Note that you will need to change the hostname as required/desired. The example uses `rdemo.zathras.io` as it's hostname and generates a cert for the same. 

To deploy with standard Ingress, use the [`cafe-ingress.yaml`](./examples/cafe-ingress.yaml) manifest.

To deploy with TLS secured Ingress, use the [`cafe-ingress-certmgr.yaml`](./examples/cafe-ingress-certmgr.yaml) manifest.

## 10. Setup Kibana
Unlike a full deploy of the elasticstack, this version does not deploy all of the associated indexes and dashboards. To help address this issue, you can import an `ndjson` file that contains all of the necessary dashboards, visualizations, and index patterns required to view NGINX data.

As part of the standard setup, a ClusterIP is created for the Kibana application that can be accessed from within the Rancher interface. Alternatiely a LoadBalancer can can assigned to the Kibana service which will allow you to access the application via an LB address. (Note that by default Kibana listens on port 5601). 

To import this data:
- Log into the Kibana application GUI:
- Click on "Stack Management"
- Click on "Saved Objects"
- Click on "Import"
- Select the [`nginx-kic.ndjson`](./kibana/nginx-kic.ndjson) file and load it. 

Now make sure you have an Index pattern defined. To do this:
- Log into the Kibana application GUI:
- Click on "Visualizations"
- Follow the instructions to create an index pattern for `filebeat-7.13.1-`

## 11. Test the Application
You can now test the application. Using the test `rdemo.zathras.io` domain this would look like this:

```
curl https://rdemo.zathras.io/tea
curl https://rdemo.zathras.io/cofee
```

You can also test from a web browser.

## 12. Generate Load Against the Application
In order to generate a load against the server, you can use the [`seige`](https://github.com/JoeDog/siege) load tester. This can be setup to use a file that contains the two above URL's then run via:

test-urls
```
https://rdemo.zathras.io/tea
https://rdemo.zathras.io/cofee
```

command
```
siege -c 5 -d 2  -f ./test-urls
```

This will create enough load that you will be able to visualize the data in Kibana. This process can be stopped at any time via `^C`.

## 13. View Kibana Dashboards
From within the Kibana GUI, you can click on "Dashboards" and then search on "NGINX" to find the various related dashboards to view the data that is being sent back from the NGINX KIC during the test.