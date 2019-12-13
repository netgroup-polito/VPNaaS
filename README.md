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
