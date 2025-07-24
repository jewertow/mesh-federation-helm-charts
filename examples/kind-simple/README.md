## Mesh federation

### Setup clusters

```shell
curl -s https://raw.githubusercontent.com/istio/istio/refs/heads/release-1.26/samples/kind-lb/setupkind.sh | sh -s -- --cluster-name east --ip-space 254
curl -s https://raw.githubusercontent.com/istio/istio/refs/heads/release-1.26/samples/kind-lb/setupkind.sh | sh -s -- --cluster-name west --ip-space 255
```

```shell
kind get kubeconfig --name east > east.kubeconfig
alias keast="KUBECONFIG=$(pwd)/east.kubeconfig kubectl"
alias istioctl-east="KUBECONFIG=$(pwd)/east.kubeconfig istioctl"
alias helm-east="KUBECONFIG=$(pwd)/east.kubeconfig helm"
kind get kubeconfig --name west > west.kubeconfig
alias kwest="KUBECONFIG=$(pwd)/west.kubeconfig kubectl"
alias istioctl-west="KUBECONFIG=$(pwd)/west.kubeconfig istioctl"
alias helm-west="KUBECONFIG=$(pwd)/west.kubeconfig helm"
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
make -f common.mk clean
```

```shell
keast create namespace istio-system
keast create secret generic cacerts -n istio-system \
  --from-file=root-cert.pem=east/root-cert.pem \
  --from-file=ca-cert.pem=east/ca-cert.pem \
  --from-file=ca-key.pem=east/ca-key.pem \
  --from-file=cert-chain.pem=east/cert-chain.pem
kwest create namespace istio-system
kwest create secret generic cacerts -n istio-system \
  --from-file=root-cert.pem=west/root-cert.pem \
  --from-file=ca-cert.pem=west/ca-cert.pem \
  --from-file=ca-key.pem=west/ca-key.pem \
  --from-file=cert-chain.pem=west/cert-chain.pem
```

### Install Kubernetes Gateway API

```shell
keast apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.3.0/standard-install.yaml
kwest apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.3.0/standard-install.yaml
```

### Demo

1. Install control planes:

    ```shell
    istioctl-east install -y -f istio.yaml \
      --set values.global.meshID=east-mesh \
      --set values.global.network=east-network \
      --set values.global.multiCluster.clusterName=east-cluster
    istioctl-west install -y -f istio.yaml \
      --set values.global.meshID=west-mesh \
      --set values.global.network=west-network \
      --set values.global.multiCluster.clusterName=west-cluster
    ```

1. Enable mTLS, deploy `sleep` the east cluster and `httpbin` in the west cluster and export `httpbin`:

    ```shell
    # mTLS
    keast apply -f mtls.yaml -n istio-system
    kwest apply -f mtls.yaml -n istio-system
    # sleep
    keast create namespace sleep
    keast label namespace sleep istio-injection=enabled
    keast apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/sleep/sleep.yaml -n sleep
    # httpbin
    keast create namespace httpbin
    keast label namespace httpbin istio-injection=enabled
    keast apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/httpbin/httpbin.yaml -n httpbin
    kwest create namespace httpbin
    kwest label namespace httpbin istio-injection=enabled
    kwest apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/httpbin/httpbin.yaml -n httpbin
    ```

1. Deploy an east-west gateway in the west cluster and export httpbin service:

    ```shell
    helm-west install istio-eastwestgateway istio/gateway -n istio-system --version 1.26.2 -f west/ingress-gateway-values.yaml
    helm-west install mesh-federation ../../mesh-admin -n istio-system -f west/mesh-admin-values.yaml
    helm-west install export-httpbin ../../namespace-admin -n httpbin -f west/mesh-admin-values.yaml -f west/ns-admin-values.yaml
    ```

1. Import httpbin to the east cluster:

    ```shell
    WEST_INGRESS_IP=$(kwest get svc istio-eastwestgateway -n istio-system -o jsonpath='{.status.loadBalancer.ingress[].ip}')
    helm-east upgrade --install mesh-federation ../../mesh-admin -n istio-system \
      -f east/mesh-admin-values.yaml \
      --set "global.remote[0].addresses[0]=$WEST_INGRESS_IP"
    helm-east upgrade --install import-httpbin ../../namespace-admin -n httpbin \
      -f east/mesh-admin-values.yaml \
      -f east/ns-admin-values.yaml \
      --set "global.remote[0].addresses[0]=$WEST_INGRESS_IP"
    ```

1. Test connectivity:

    ```shell
    keast exec -n sleep deploy/sleep -c sleep -- curl -v httpbin.mesh.global:8000/headers
    ```

1. Enable the egress gateway in the east cluster and test connectivity again:

    ```shell
    helm-east install istio-egressgateway istio/gateway -n istio-system --version 1.26.2 -f east/egress-gateway-values.yaml
    WEST_INGRESS_IP=$(kwest get svc istio-eastwestgateway -n istio-system -o jsonpath='{.status.loadBalancer.ingress[].ip}')
    helm-east upgrade --install mesh-federation ../../mesh-admin -n istio-system \
      -f east/mesh-admin-values.yaml \
      --set "global.remote[0].addresses[0]=$WEST_INGRESS_IP" \
      --set global.gateways.egress.enabled=true
    helm-east upgrade --install import-httpbin ../../namespace-admin -n httpbin \
      -f east/mesh-admin-values.yaml \
      -f east/ns-admin-values.yaml \
      --set "global.remote[0].addresses[0]=$WEST_INGRESS_IP" \
      --set global.gateways.egress.enabled=true
    ```
