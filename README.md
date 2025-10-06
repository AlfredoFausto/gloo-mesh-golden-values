# gloo-mesh-golden-values

## How to install

The order matters when it comes to installing the new ambient mode:

1. Trust is set, cacerts need to exist before installing Ambient
2. Ambient Mesh (mgmt cluster and workload clusters)
3. Mgmt plane
4. Agents
5. Linking clusters

### Ambient

CRDs
```bash
helm upgrade -i istio-base oci://${HELM_REPO}/base                     \
--namespace istio-system                                          \
--create-namespace                                                \
--set defaultRevision=main                                        \
--version ${ISTIO_VER}-${SUFFIX_SOLO}                             \
--wait 
```

Istiod
```bash
helm upgrade -i istiod oci://${HELM_REPO}/istiod \
--namespace istio-system                         \
--version ${ISTIO_VER}-${SUFFIX_SOLO}            \
--wait --kube-context $cluster                   \
--values istiod.yaml
```

istio-cni

```bash
 helm upgrade -i istio-cni oci://${HELM_REPO}/cni \
--namespace istio-system                         \
--version ${ISTIO_VER}-${SUFFIX_SOLO}            \
--wait --kube-context $cluster                   \
--values istio-cni.yaml
```

ztunnel
```bash
helm upgrade -i ztunnel oci://${HELM_REPO}/ztunnel \
--namespace istio-system                           \
--version ${ISTIO_VER}-${SUFFIX_SOLO}              \
--wait --kube-context $cluster                     \
--values ztunnel.yaml
```

You need to label the namespace in each cluster with the same network provided in the helm values:

```bash
kubectl label namespace istio-system topology.istio.io/network=${MGMT} --overwrite --context ${MGMT} || true
kubectl label namespace istio-system topology.istio.io/network=${CLUSTER1} --overwrite --context ${CLUSTER1} || true
kubectl label namespace istio-system topology.istio.io/network=${CLUSTER2} --overwrite --context ${CLUSTER2} || true
```

### Gloo Mesh Enterprise

Create and label the ns with ambient mode
```bash
kubectl --context ${MGMT} create ns gloo-mesh
kubectl --context ${MGMT} label namespace gloo-mesh istio.io/dataplane-mode=ambient
```

Install Crds and mgmt plane

```bash
helm upgrade --install gloo-platform-crds gloo-platform-crds \
  --repo https://storage.googleapis.com/gloo-platform/helm-charts \
  --namespace gloo-mesh \
  --kube-context ${MGMT} \
  --set featureGates.insightsConfiguration=true \
  --set featureGates.ConfigDistribution=true \
  --set installEnterpriseCrds=true \
  --version ${GLOO_MESH_VERSION}
    
helm upgrade --install gloo-platform-mgmt gloo-platform \
  --repo https://storage.googleapis.com/gloo-platform/helm-charts \
  --namespace gloo-mesh \
  --set licensing.glooTrialLicenseKey=${GLOO_LICENSE_KEY} \
  --kube-context ${MGMT} \
  --version ${GLOO_MESH_VERSION} \
  -f mgmt.yaml
```

Label the telemetry gateway with the service-scope global

```bash
kubectl label svc gloo-telemetry-gateway solo.io/service-scope=global -n gloo-mesh --context ${MGMT}
```

Install agents in each cluster

```bash
kubectl --context ${CLUSTER1} create ns gloo-mesh
kubectl label ns gloo-mesh istio.io/dataplane-mode=ambient --context ${CLUSTER1}
```

Install Crds and agents

```bash
helm upgrade --install gloo-platform-crds gloo-platform-crds \
      --repo https://storage.googleapis.com/gloo-platform/helm-charts \
      --namespace gloo-mesh \
      --set installEnterpriseCrds=true \
      --kube-context ${CLUSTER1} \
      --version ${GLOO_MESH_VERSION}
    
    helm upgrade --install gloo-platform gloo-platform \
      --repo https://storage.googleapis.com/gloo-platform/helm-charts \
      --namespace gloo-mesh \
      --kube-context ${CLUSTER1} \
      --version ${GLOO_MESH_VERSION} \
      -f <(envsubst < agent.yaml)
```
Make sure you get the address of the mgmt plane before installing the agent and put into the values file.

### Linking clusters

You need to create the eastwest gateway, and automatically the mgmt plane will create all the remote gateways needed for multicluster communication.

Mgmt
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  labels:
    istio.io/expose-istiod: "15012"
    topology.istio.io/cluster: mgmt
    topology.istio.io/network: mgmt
  name: istio-eastwest
  namespace: istio-eastwest
spec:
  gatewayClassName: istio-eastwest
  listeners:
  - name: cross-network
    port: 15008
    protocol: HBONE
    tls:
      mode: Passthrough
    allowedRoutes:
      namespaces:
        from: All
  - name: xds-tls
    port: 15012
    protocol: TLS
    tls:
      mode: Passthrough
    allowedRoutes:
      namespaces:
        from: All
```

cluster1
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  labels:
    istio.io/expose-istiod: "15012"
    topology.istio.io/cluster: cluster1
    topology.istio.io/network: cluster1
  name: istio-eastwest
  namespace: istio-eastwest
spec:
  gatewayClassName: istio-eastwest
  listeners:
  - name: cross-network
    port: 15008
    protocol: HBONE
    tls:
      mode: Passthrough
    allowedRoutes:
      namespaces:
        from: All
  - name: xds-tls
    port: 15012
    protocol: TLS
    tls:
      mode: Passthrough
    allowedRoutes:
      namespaces:
        from: All
```
