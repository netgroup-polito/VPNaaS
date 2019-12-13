# HPA with Custom Metrics: the VPN-as-a-service use case.

Provision an OpenVPN installation on k8s that can autoscale against custom metrics.

## Installation

The chart is forked from this [official openvpn chart](https://github.com/helm/charts/tree/master/stable/openvpn).

I have added a sidecar container to the deployment. The container is an openvpn exporter, which harvests openvpn metrics and exposes them on port 9176.

```
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


After the chart is deployed and the pod is ready, an openvpn certificate can be generate using the following commands:

```
POD_NAME=$(kubectl get pods --namespace <namespace> -l "app=openvpn,release=<your_release>" -o jsonpath='{ .items[0].metadata.name }')
SERVICE_NAME=$(kubectl get svc --namespace <namespace>  -l "app=openvpn,release=<your_release>" -o jsonpath='{ .items[0].metadata.name }')
SERVICE_IP=$(kubectl get svc --namespace <namespace>  "$SERVICE_NAME" -o go-template='{{ range $k, $v := (index .status.loadBalancer.ingress 0)}}{{ $v }}{{end}}')
KEY_NAME=<key_name>
kubectl --namespace <namespace>  exec -it "$POD_NAME" -c openvpn /etc/openvpn/setup/newClientCert.sh "$KEY_NAME" "$SERVICE_IP"
kubectl --namespace <namespace>  exec -it "$POD_NAME" -c openvpn cat "/etc/openvpn/certs/pki/$KEY_NAME.ovpn" > "$KEY_NAME.ovpn"
```

Clients certificates can be revoked in this manner:

```
KEY_NAME=<key_name>
POD_NAME=$(kubectl get pods -n <namespace> -l "app=openvpn,release=<your_release>" -o jsonpath='{.items[0].metadata.name}')
kubectl -n <namespace> exec -it "$POD_NAME" /etc/openvpn/setup/revokeClientCert.sh $KEY_NAME
```

To take a look at the metrics, you can use port-forwarding.

Run `kubectl port-forward <pod_name> 9176:9176` and then connect to [http://localhost:9176/metrics](http://localhost:9176/metrics).

You should now be able to see some OpenVPN metrics:

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
