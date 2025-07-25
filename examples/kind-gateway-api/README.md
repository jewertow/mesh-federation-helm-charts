## Mesh federation on KinD

### Setup clusters

```shell
curl -s https://raw.githubusercontent.com/istio/istio/refs/heads/release-1.26/samples/kind-lb/setupkind.sh | sh -s -- --cluster-name east --ip-space 255
curl -s https://raw.githubusercontent.com/istio/istio/refs/heads/release-1.26/samples/kind-lb/setupkind.sh | sh -s -- --cluster-name west --ip-space 254
curl -s https://raw.githubusercontent.com/istio/istio/refs/heads/release-1.26/samples/kind-lb/setupkind.sh | sh -s -- --cluster-name central --ip-space 253
```

```shell
# east
kind get kubeconfig --name east > east.kubeconfig
alias keast="KUBECONFIG=$(pwd)/east.kubeconfig kubectl"
alias istioctl-east="KUBECONFIG=$(pwd)/east.kubeconfig istioctl"
alias helm-east="KUBECONFIG=$(pwd)/east.kubeconfig helm"
# west
kind get kubeconfig --name west > west.kubeconfig
alias kwest="KUBECONFIG=$(pwd)/west.kubeconfig kubectl"
alias istioctl-west="KUBECONFIG=$(pwd)/west.kubeconfig istioctl"
alias helm-west="KUBECONFIG=$(pwd)/west.kubeconfig helm"
# central
kind get kubeconfig --name central > central.kubeconfig
alias kcent="KUBECONFIG=$(pwd)/central.kubeconfig kubectl"
alias istioctl-cent="KUBECONFIG=$(pwd)/central.kubeconfig istioctl"
alias helm-cent="KUBECONFIG=$(pwd)/central.kubeconfig helm"
```

### Setup certificates

```shell
wget https://raw.githubusercontent.com/istio/istio/release-1.21/tools/certs/common.mk -O common.mk
wget https://raw.githubusercontent.com/istio/istio/release-1.21/tools/certs/Makefile.selfsigned.mk -O Makefile.selfsigned.mk
```

```shell
make -f Makefile.selfsigned.mk \
  ROOTCA_CN="East Root CA" \
  ROOTCA_ORG=my-company.org \
  root-ca
make -f Makefile.selfsigned.mk \
  INTERMEDIATE_CN="East Intermediate CA" \
  INTERMEDIATE_ORG=my-company.org \
  east-cacerts
make -f Makefile.selfsigned.mk \
  INTERMEDIATE_CN="West Intermediate CA" \
  INTERMEDIATE_ORG=my-company.org \
  west-cacerts
make -f Makefile.selfsigned.mk \
  INTERMEDIATE_CN="Central Intermediate CA" \
  INTERMEDIATE_ORG=my-company.org \
  central-cacerts
make -f common.mk clean
```

```shell
# east
keast create namespace istio-system
keast create secret generic cacerts -n istio-system \
  --from-file=root-cert.pem=east/root-cert.pem \
  --from-file=ca-cert.pem=east/ca-cert.pem \
  --from-file=ca-key.pem=east/ca-key.pem \
  --from-file=cert-chain.pem=east/cert-chain.pem
# west
kwest create namespace istio-system
kwest create secret generic cacerts -n istio-system \
  --from-file=root-cert.pem=west/root-cert.pem \
  --from-file=ca-cert.pem=west/ca-cert.pem \
  --from-file=ca-key.pem=west/ca-key.pem \
  --from-file=cert-chain.pem=west/cert-chain.pem
# central
kcent create namespace istio-system
kcent create secret generic cacerts -n istio-system \
  --from-file=root-cert.pem=central/root-cert.pem \
  --from-file=ca-cert.pem=central/ca-cert.pem \
  --from-file=ca-key.pem=central/ca-key.pem \
  --from-file=cert-chain.pem=central/cert-chain.pem
```

### Install Kubernetes Gateway API

```shell
keast apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.3.0/standard-install.yaml
kwest apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.3.0/standard-install.yaml
kcent apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.3.0/standard-install.yaml
```

### Install control planes

```shell
# east
istioctl-east install -y -f istio.yaml \
  --set values.global.meshID=east-mesh \
  --set values.global.network=east-network \
  --set values.global.multiCluster.clusterName=east-cluster
# west
istioctl-west install -y -f istio.yaml \
  --set values.global.meshID=west-mesh \
  --set values.global.network=west-network \
  --set values.global.multiCluster.clusterName=west-cluster
# central
istioctl-cent install -y -f istio.yaml \
  --set values.global.meshID=central-mesh \
  --set values.global.network=central-network \
  --set values.global.multiCluster.clusterName=central-cluster
```
```shell
keast apply -f mtls.yaml -n istio-system
kwest apply -f mtls.yaml -n istio-system
kcent apply -f mtls.yaml -n istio-system
```

### East cluster

1. Deploy and export `ratings` service:

   ```shell
   keast create namespace east-apps
   keast label namespace east-apps istio-injection=enabled
   keast apply -n east-apps -l account=ratings -f https://raw.githubusercontent.com/istio/istio/refs/heads/release-1.26/samples/bookinfo/platform/kube/bookinfo.yaml
   keast apply -n east-apps -l app=ratings -f https://raw.githubusercontent.com/istio/istio/refs/heads/release-1.26/samples/bookinfo/platform/kube/bookinfo.yaml
   ```
   ```shell
   cat <<EOF > east/istio-eastwestgateway-values.yaml
   service:
     ports:
     - name: cross-network-tls
       port: 15443
   env:
     ISTIO_META_REQUESTED_NETWORK_VIEW: east-network
   EOF
   helm-east upgrade --install istio-eastwestgateway istio/gateway -n istio-system --version 1.26.2 -f east/istio-eastwestgateway-values.yaml
   ```
   ```shell
   cat <<EOF > east/mesh-admin-values.yaml
   global:
     enableGatewayAPI: true
     defaultServicePorts:
     - number: 9080
       name: http
       protocol: HTTP
     local:
       export:
       - hostname: ratings.mesh.global
   EOF
   helm-east install mesh-federation ../../mesh-admin -n istio-system -f east/mesh-admin-values.yaml
   ```
   ```shell
   cat <<EOF > east/namespace-admin-values.yaml
   export:
   - hostname: ratings.mesh.global
     labelSelector:
       app: ratings
   EOF
   helm-east install bookinfo-federation ../../namespace-admin -n east-apps -f east/mesh-admin-values.yaml -f east/namespace-admin-values.yaml
   ```

### West cluster

1. Deploy and export `details` and `ratings` services:

   ```shell
   kwest create namespace west-apps
   kwest label namespace west-apps istio-injection=enabled
   kwest apply -n west-apps -l account=details -f https://raw.githubusercontent.com/istio/istio/refs/heads/release-1.26/samples/bookinfo/platform/kube/bookinfo.yaml
   kwest apply -n west-apps -l app=details -f https://raw.githubusercontent.com/istio/istio/refs/heads/release-1.26/samples/bookinfo/platform/kube/bookinfo.yaml
   kwest apply -n west-apps -l account=ratings -f https://raw.githubusercontent.com/istio/istio/refs/heads/release-1.26/samples/bookinfo/platform/kube/bookinfo.yaml
   kwest apply -n west-apps -l app=ratings -f https://raw.githubusercontent.com/istio/istio/refs/heads/release-1.26/samples/bookinfo/platform/kube/bookinfo.yaml
   ```
   ```shell
   cat <<EOF > west/istio-eastwestgateway-values.yaml
   service:
     ports:
     - name: cross-network-tls
       port: 15443
   env:
     ISTIO_META_REQUESTED_NETWORK_VIEW: west-network
   EOF
   helm-west upgrade --install istio-eastwestgateway istio/gateway -n istio-system --version 1.26.2 -f west/istio-eastwestgateway-values.yaml
   ```
   ```shell
   cat <<EOF > west/mesh-admin-values.yaml
   global:
     enableGatewayAPI: true
     defaultServicePorts:
     - number: 9080
       name: http
       protocol: HTTP
     local:
       export:
       - hostname: details.mesh.global
       - hostname: ratings.mesh.global
   EOF
   helm-west install mesh-federation ../../mesh-admin -n istio-system -f west/mesh-admin-values.yaml
   ```
   ```shell
   cat <<EOF > west/namespace-admin-values.yaml
   export:
   - hostname: details.mesh.global
     labelSelector:
       app: details
   - hostname: ratings.mesh.global
     labelSelector:
       app: ratings
   EOF
   helm-west install bookinfo-federation ../../namespace-admin -n west-apps -f west/mesh-admin-values.yaml -f west/namespace-admin-values.yaml
   ```

### Central cluster

This cluster will import services from east and west clusters.

1. Deploy ingress and egress gateways:

   ```shell
   cat <<EOF > central/istio-egressgateway-values.yaml
   service:
     type: ClusterIP
     ports:
     - name: cross-network-tls
       port: 15443
   EOF
   helm-cent upgrade --install istio-egressgateway istio/gateway -n istio-system --version 1.26.2 -f central/istio-egressgateway-values.yaml
   helm-cent upgrade --install istio-ingressgateway istio/gateway -n istio-system --version 1.26.2
   ```

1. Deploy productpage and reviews services:

   ```shell
   kcent create namespace central-apps
   kcent label namespace central-apps istio-injection=enabled
   kcent apply -l app=productpage -f https://raw.githubusercontent.com/istio/istio/refs/heads/release-1.26/samples/bookinfo/platform/kube/bookinfo.yaml -n central-apps
   kcent apply -l account=productpage -f https://raw.githubusercontent.com/istio/istio/refs/heads/release-1.26/samples/bookinfo/platform/kube/bookinfo.yaml -n central-apps
   kcent apply -l app=reviews -f https://raw.githubusercontent.com/istio/istio/refs/heads/release-1.26/samples/bookinfo/platform/kube/bookinfo.yaml -n central-apps
   kcent apply -l account=reviews -f https://raw.githubusercontent.com/istio/istio/refs/heads/release-1.26/samples/bookinfo/platform/kube/bookinfo.yaml -n central-apps
   kcent apply -l app=details -f https://raw.githubusercontent.com/istio/istio/refs/heads/release-1.26/samples/bookinfo/platform/kube/bookinfo.yaml -n central-apps
   kcent apply -l account=details -f https://raw.githubusercontent.com/istio/istio/refs/heads/release-1.26/samples/bookinfo/platform/kube/bookinfo.yaml -n central-apps
   ```
   ```shell
   kcent apply -f https://raw.githubusercontent.com/istio/istio/refs/heads/release-1.26/samples/bookinfo/networking/bookinfo-gateway.yaml -n central-apps
   kcent apply -n central-apps -f - <<EOF
   apiVersion: networking.istio.io/v1
   kind: Gateway
   metadata:
      name: bookinfo-gateway
   spec:
     selector:
      istio: ingressgateway
     servers:
     - port:
         number: 80
         name: http
         protocol: HTTP
       hosts:
       - "*"
   EOF
   ```

1. Enable mesh federation and import remote services:

   ```shell
   EAST_INGRESS_IP=$(keast get svc istio-eastwestgateway -n istio-system -o jsonpath='{.status.loadBalancer.ingress[].ip}')
   WEST_INGRESS_IP=$(kwest get svc istio-eastwestgateway -n istio-system -o jsonpath='{.status.loadBalancer.ingress[].ip}')
   cat <<EOF > central/mesh-admin-values.yaml
   global:
     defaultServicePorts:
     - number: 9080
       name: http
       protocol: HTTP
     enableGatewayAPI: true
     gateways:
       egress:
         enabled: true
     remote:
     - mesh: east-mesh
       addresses:
       - $EAST_INGRESS_IP
       network: east-network
       importedServices:
       - ratings.mesh.global
     - mesh: west-mesh
       addresses:
       - $WEST_INGRESS_IP
       network: west-network
       importedServices:
       - details.mesh.global
       - ratings.mesh.global
   EOF
   helm-cent install mesh-federation ../../mesh-admin -n istio-system -f central/mesh-admin-values.yaml
   ```
   ```shell
   cat <<EOF > central/namespace-admin-values.yaml
   import:
   - hostname: ratings.mesh.global
   - hostname: details.mesh.global
   export:
   - hostname: details.mesh.global
     labelSelector:
       app: details
   EOF
   helm-cent install bookinfo-federation ../../namespace-admin -n central-apps -f central/mesh-admin-values.yaml -f central/namespace-admin-values.yaml
   ```

1. Update productpage to consume imported ratings:

   ```shell
   kcent patch deployment productpage-v1 -n central-apps \
     --type='strategic' \
     -p='{
       "spec": {
         "template": {
           "spec": {
             "containers": [
               {
                 "name": "productpage",
                 "env": [
                   {
                     "name": "DETAILS_HOSTNAME",
                     "value": "details.mesh.global"
                   }
                 ]
               }
             ]
           }
         }
       }
     }'
   ```
   ```shell
   kcent patch deployment reviews-v2 -n central-apps \
     --type='strategic' \
     -p='{
       "spec": {
         "template": {
           "spec": {
             "containers": [
               {
                 "name": "reviews",
                 "env": [
                   {
                     "name": "RATINGS_HOSTNAME",
                     "value": "ratings.mesh.global"
                   }
                 ]
               }
             ]
           }
         }
       }
     }'
   ```
   ```shell
   kcent patch deployment reviews-v3 -n central-apps \
     --type='strategic' \
     -p='{
       "spec": {
         "template": {
           "spec": {
             "containers": [
               {
                 "name": "reviews",
                 "env": [
                   {
                     "name": "RATINGS_HOSTNAME",
                     "value": "ratings.mesh.global"
                   }
                 ]
               }
             ]
           }
         }
       }
     }'
   ```

1. Send requests in a loop:

   ```shell
   IP=$(kcent get svc istio-ingressgateway -n istio-system -o jsonpath='{.status.loadBalancer.ingress[].ip}')
   while true; do curl -v http://$IP/productpage > /dev/null; sleep 2; done
   ```