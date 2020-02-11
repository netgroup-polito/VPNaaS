# HPA with Custom Metrics: the VPN-as-a-service use case.

Provision an OpenVPN installation on k8s that can autoscale against custom metrics.

## Architecture

This project contains a full OpenVPN deployment for k8s, which is coupled with an OpenVPN metrics exporter. The exporters harvests metrics from the OpenVPN instance and exposes them for Prometheus (note that the an instance of the [Prometheus Operator](https://github.com/coreos/prometheus-operator) needs to be running on the cluster).

These metrics are then fed to the [Prometheus Adapter](https://github.com/helm/charts/tree/master/stable/prometheus-adapter), which implements the k8s [Custom Metrics API](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#support-for-metrics-apis). The Adapter is reponsible for exposing the metrics through the k8s API, so that they can be queried by an HPA instance for autoscaling.




## Prerequisites

Everything was tested with:

* [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) version v1.12.0+.
* Kubernetes v1.6+ cluster.
* [helm](https://helm.sh/docs/intro/install/) v2.16+

## Installation

The Helm OpenVPN chart is derived from the [official one](https://github.com/helm/charts/tree/master/stable/openvpn). This fork includes new shared volumes that are used to share OpenVPN metrics, and a sidecar container that exports these metrics for Prometheus. 

To install from the chart directory, run 
```helm install --name <release_name> --tiller-namespace <tiller_namespace> .```

As an example, to install the chart in the `johndoe` namespace, you might do
```helm install --name openvpn_v01 --tiller-namespace johndoe .```


The metrics exporter, which is taken from [this project](https://github.com/kumina/openvpn_exporter), is deployed as a sidecar container in the OpenVPN pod, and it exposes metrics on port 9176. This is shown in the following code snippet, where the exporter image is used, and the commands for exporting the metrics are run.

```YAML
...

containers:
  - name: exporter
    image: kumina/openvpn-exporter
    command: ["/bin/openvpn_exporter"]
    args: ["-openvpn.status_paths", "/etc/openvpn-exporter/openvpn/openvpn-status.log"]
    volumeMounts:
    - name: openvpn-status
      mountPath: /etc/openvpn-exporter/openvpn
      
...
```        

Docs for the exporter are available [here](https://github.com/kumina/openvpn_exporter).

This chart also contains some minor tweaks that are used to make it compatible with the exporter, such as adding the `status-version 2` option in the OpenVPN configuration file.


After the chart is deployed and the pod is ready, an OpenVPN certificate for a new user can be generated. The certificate will allow a user to connect to the VPN using any OpenVPN client available.

Certificates can be generated using the following commands:

```bash
POD_NAME=$(kubectl get pods --namespace <namespace> -l "app=openvpn,release=<your_release>" -o jsonpath='{ .items[0].metadata.name }')
SERVICE_NAME=$(kubectl get svc --namespace <namespace>  -l "app=openvpn,release=<your_release>" -o jsonpath='{ .items[0].metadata.name }')
SERVICE_IP=$(kubectl get svc --namespace <namespace>  "$SERVICE_NAME" -o go-template='{{ range $k, $v := (index .status.loadBalancer.ingress 0)}}{{ $v }}{{end}}')
KEY_NAME=<key_name>
kubectl --namespace <namespace>  exec -it "$POD_NAME" -c openvpn /etc/openvpn/setup/newClientCert.sh "$KEY_NAME" "$SERVICE_IP"
kubectl --namespace <namespace>  exec -it "$POD_NAME" -c openvpn cat "/etc/openvpn/certs/pki/$KEY_NAME.ovpn" > "$KEY_NAME.ovpn"
```


Clients certificates can be revoked in this manner:

```bash
KEY_NAME=<key_name>
POD_NAME=$(kubectl get pods -n <namespace> -l "app=openvpn,release=<your_release>" -o jsonpath='{.items[0].metadata.name}')
kubectl -n <namespace> exec -it "$POD_NAME" /etc/openvpn/setup/revokeClientCert.sh $KEY_NAME
```

To take a look at the metrics, you can use port-forwarding.

Run `kubectl port-forward <pod_name> 9176:9176` and then connect to [http://localhost:9176/metrics](http://localhost:9176/metrics).

You should now be able to see some Prometheus metrics of your OpenVPN instance:

```
# HELP openvpn_openvpn_server_connected_clients Number Of Connected Clients
# TYPE openvpn_openvpn_server_connected_clients gauge
openvpn_openvpn_server_connected_clients{status_path="/etc/openvpn-exporter/openvpn/openvpn-status.log"} 1
# HELP openvpn_server_client_received_bytes_total Amount of data received over a connection on the VPN server, in bytes.
# TYPE openvpn_server_client_received_bytes_total counter
openvpn_server_client_received_bytes_total{common_name="CC2",connection_time="1576248156",real_address="10.244.0.0:25878",status_path="/etc/openvpn-exporter/openvpn/openvpn-status.log",username="UNDEF",virtual_address="10.240.0.6"} 17762
# HELP openvpn_server_client_sent_bytes_total Amount of data sent over a connection on the VPN server, in bytes.
# TYPE openvpn_server_client_sent_bytes_total counter
openvpn_server_client_sent_bytes_total{common_name="CC2",connection_time="1576248156",real_address="10.244.0.0:25878",status_path="/etc/openvpn-exporter/openvpn/openvpn-status.log",username="UNDEF",virtual_address="10.240.0.6"} 19047
```

At this point, you should have a working OpenVPN installation that runs on Kubernetes. The following steps will allow you to expose metrics through the [Custom Metrics API](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#support-for-metrics-apis), that allows us to autoscale against OpenVPN metrics.

## Exposing the metrics

We first need to expose the exporter through a service, so that the Prometheus operator can access it, by running `kubectl apply -f exporter_service.yaml`. This is a very simple service that sits in front of our OpenVPN pods, and that defines a port through which we expose the metrics.

Running `kubectl apply -f servicemonitor.yaml` will now deploy the service monitor that is used by Prometheus to harvest our metrics.
A service monitor is a Prometheus operator custom resource which declaratevly specifies how groups of services should be monitored.

Once everything is up and running, we are now ready to autoscale against our custom metrics.
The following YAML snippet shows a HPA that scales against the number of users currently connected to the VPN:

```YAML
kind: HorizontalPodAutoscaler
apiVersion: autoscaling/v2beta1
metadata:
  name: openvpn
spec:
  scaleTargetRef:
    # point the HPA at the sample application
    # you created above
    apiVersion: apps/v1
    kind: Deployment
    name: <your_openvpn_deployment>
  # autoscale between 1 and 10 replicas
  minReplicas: 1
  maxReplicas: 10
  metrics:
  # use a "Pods" metric, which takes the average of the
  # given metric across all pods controlled by the autoscaling target
  - type: Pods
    pods:
      metricName: openvpn_openvpn_server_connected_clients
      targetAverageValue: 3
```

Where `<your_openvpn_deployment>` should be replaced with the name of your OpenVPN deployment.

## Troubleshooting


### Internet traffic through VPN

You can avoid routing all traffic through the VPN by setting `redirectGateway: false`.

## TODO

* Manage certificate persistence across replicas.
