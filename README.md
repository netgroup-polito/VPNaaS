# HPA with Custom Metrics: the VPN-as-a-service use case.

Provision an OpenVPN installation on k8s that can autoscale against custom metrics.

## Installation

The chart is forked from this [official openvpn chart](https://github.com/helm/charts/tree/master/stable/openvpn).

I have added a sidecar container in the deployment. The container is an openvpn exporter, which harvests openvpn metrics and exposes them on port 9176.

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


After the chart is deployed and the pod is read, an openvpn certificate can be generate using the following commands:

```
POD_NAME=$(kubectl get pods --namespace <namespace> -l "app=openvpn,release=<your_release>" -o jsonpath='{ .items[0].metadata.name }')
SERVICE_NAME=$(kubectl get svc --namespace <namespace>  -l "app=openvpn,release=<your_release>" -o jsonpath='{ .items[0].metadata.name }')
SERVICE_IP=$(kubectl get svc --namespace <namespace>  "$SERVICE_NAME" -o go-template='{{ range $k, $v := (index .status.loadBalancer.ingress 0)}}{{ $v }}{{end}}')
KEY_NAME=<key_name>
kubectl --namespace <namespace>  exec -it "$POD_NAME" -c openvpn /etc/openvpn/setup/newClientCert.sh "$KEY_NAME" "$SERVICE_IP"
kubectl --namespace <namespace>  exec -it "$POD_NAME" -c openvpn cat "/etc/openvpn/certs/pki/$KEY_NAME.ovpn" > "$KEY_NAME.ovpn"
```
