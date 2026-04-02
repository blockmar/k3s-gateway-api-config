# Gateway API Support in K3S / Traefik

This repo documents how to enable [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/) support in [K3S](https://k3s.io/), which uses [Traefik](https://traefik.io/) as its default ingress controller.

The Gateway API is the modern successor to the `Ingress` resource, offering more expressive routing, better multi-tenant support, and a richer set of features. Enabling it in K3S requires a small configuration override — it does not replace or break existing `Ingress` resources.

K3S's official documentation returns [no results](https://docs.k3s.io/search/?q=gateway+api) when searching for "Gateway API" (as of April 2026). The solution turns out to be simple — hopefully this saves you the time it took to find it.

---

## Background: How K3S manages Traefik

K3S deploys Traefik automatically and manages it via a Helm chart. From the [K3S networking docs](https://docs.k3s.io/networking/networking-services#traefik-ingress-controller):

> The default chart values can be found in `/var/lib/rancher/k3s/server/manifests/traefik.yaml`, but **this file should not be edited manually**, as K3s will replace the file with defaults at startup. Instead, you should customize Traefik by creating an additional `HelmChartConfig` manifest in `/var/lib/rancher/k3s/server/manifests`.

---

## Enabling Gateway API support

Create a `HelmChartConfig` that enables the Traefik Gateway API provider. This extends the default config without replacing it, so standard `Ingress` resources continue to work.

Create a new file `/var/lib/rancher/k3s/server/manifests/traefik-gateway-api.yaml`

```yaml
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: traefik
  namespace: kube-system
spec:
  valuesContent: |-
    providers:
      kubernetesGateway:
        enabled: true
    gateway:
      enabled: true
      listeners:
        web:
          namespacePolicy:
            from: All
```

The manifest is also available in this repo: [`traefik-gateway-api.yaml`](./traefik-gateway-api.yaml)

The `namespacePolicy: from: All` setting allows `HTTPRoute` resources from any namespace to attach to this gateway.

---

## Verifying the setup

After applying the config, K3S will re-run the Helm install. Check that it completes and that the gateway is up:

```bash
# Watch the Traefik Helm install job (excludes the CRD job)
kubectl get pods -A | grep 'helm-install-traefik-' | grep -v '\-crd-'
```

The pod name will look something like `helm-install-traefik-xxxxx`. A status of `Completed` means the Helm chart was applied successfully. If it shows `Error`, check the logs:

```bash
# Replace <pod-name> with the actual pod name from the grep above
kubectl logs <pod-name> -n kube-system
```

A successful run ends with output like:

```
Release "traefik" has been upgraded. Happy Helming!
NAME: traefik
LAST DEPLOYED: Thu Apr  2 09:43:44 2026
NAMESPACE: kube-system
STATUS: deployed
REVISION: 5
TEST SUITE: None
NOTES:
traefik with docker.io/rancher/mirrored-library-traefik:3.6.10 has been deployed successfully on kube-system namespace!
+ exit
```

If the helm apply looks good, continue by verifying that the gateway has been created.

```bash
# Check that the gateway was created
kubectl get gateway -n kube-system

# Inspect the gateway spec and status
kubectl describe gateway traefik-gateway -n kube-system
```

The gateway spec should reflect the namespace policy you configured:

```
Spec:
  Gateway Class Name:  traefik
  Listeners:
    Allowed Routes:
      Namespaces:
        From:  All
    Name:      web
    Port:      8000
    Protocol:  HTTP
```

---

## Demo

The file [`nginx-demo.yaml`](./nginx-demo.yaml) contains a  working example to verify that Gateway API routing is working. It deploys:

- A `demo` namespace
- An nginx pod serving a simple HTML page at `/demo`
- A `Service` exposing the pod on port 8080
- An `HTTPRoute` attached to the `traefik-gateway`, routing `/demo` to the nginx service

Apply it with:

```bash
kubectl apply -f nginx-demo.yaml
```

Then test it (replace `<node-ip>` with your K3S node's IP):

```bash
curl http://<node-ip>/demo
```

You should get back a simple HTML page with `Hello world 1!!`.
