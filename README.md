# HPA with Custom Metrics: the VPN-as-a-service use case.

Provision an OpenVPN installation on k8s that can autoscale against custom metrics.

## Installation

The chart is forked from this [official OpenVPN chart](https://github.com/helm/charts/tree/master/stable/openvpn).

To install from the chart directory, run `helm install --name <release_name> --tiller-namespace <tiller_namespace> .`

I have added an OpenVPN exporter as a sidecar container in the deployment, which harvests OpenVPN metrics and exposes as Prometheus metrics on port 9176.

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

My chart also contains some minor tweaks that are used to make it compatible with the exporter, such as adding the `status-version 2` option in the OpenVPN configuration file.


After the chart is deployed and the pod is ready, an OpenVPN certificate can be generated using the following commands:

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

We also need to expose the exporter (no pun intended) through a service, so that the Prometheus operator can access it, by running `kubectl apply -f exporter_service.yaml`.

`kubectl apply -f servicemonitor.yaml` will now deploy the service monitor that is used by the Prometheus operator to harvest our metrics.

Once everything is up and running, we are now ready to autoscale against our custom metrics! 
The following shows a HPA that scales against the number of users currently connected to the VPN:

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

## Troubleshooting

### Load Balancer issues

If the client is not able to connect through a LoadBalancer, is it possible to switch to using the NodePort, by changing the IP and port on the client certificate.

### Internet connection

IP forwarding needs to be set on the server machines for internet connectivity to work.

