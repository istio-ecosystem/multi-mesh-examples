---
title: Perform mesh federation for TCP services.
subtitle: Access TCP services in remote meshes and perform load balancing between services in different clusters
description: Access TCP services in remote meshes and perform load balancing between services in different clusters.
keywords: [traffic-management,multicluster,security,gateway]
---

### Prerequisites for three clusters

See [Prerequisites for three clusters](/docs/examples/multimesh/multimesh-common-setup/#prerequisites-for-three-clusters)
in [common multi-mesh setup](/docs/examples/multimesh/multimesh-common-setup).

## Initial setup

In the following sections you deploy `tcp-echo` service in the second cluster and the third cluster to demonstrate load
balancing between remote clusters, and also `tcp-hello-echo` service in the third cluster to demonstrate accessing
multiple services in a remote cluster. All the three services are echo services that return the input they receive,
prepending it with a prefix string. The services differ only by their prefixes, the prefixes are `one` and `two` for the
`tco-echo` services and `hello` for `tcp-hello-echo`.

### Deploy a TCP service in the second cluster

1.  Create a namespace for a TCP service in the second cluster and label it for Istio sidecar injection:

    ```bash
    $ kubectl create --context=$CTX_CLUSTER2 namespace sample
    $ kubectl label --context=$CTX_CLUSTER2 namespace sample istio-injection=enabled
    namespace/sample created
    namespace/sample labeled
    ```

1.  Deploy the `tcp-echo` sample service, version `v1`:

    ```bash
    $ kubectl apply -n sample -l app=tcp-echo,version=v1 --context=$CTX_CLUSTER2 -f https://raw.githubusercontent.com/istio/istio/533221ef3369834ae44eaa4abcddf67c2d3dc549/samples/tcp-echo/tcp-echo.yaml
    $ kubectl apply -n sample -l service=tcp-echo --context=$CTX_CLUSTER2 -f https://raw.githubusercontent.com/istio/istio/533221ef3369834ae44eaa4abcddf67c2d3dc549/samples/tcp-echo/tcp-echo.yaml
    deployment.apps/tcp-echo-v1 created
    service/tcp-echo created
    ```

1.  Create a destination rule for `tcp-echo`:

    ```bash
    $ kubectl apply --context=$CTX_CLUSTER2 -n sample -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: DestinationRule
    metadata:
      name: tcp-echo
    spec:
      host: tcp-echo
      trafficPolicy:
        tls:
          mode: ISTIO_MUTUAL
    EOF
    ```

### Deploy two TCP services in the third cluster

1.  Create a namespace for a TCP service in the second cluster and label it for Istio sidecar injection:

    ```bash
    $ kubectl create --context=$CTX_CLUSTER3 namespace sample
    $ kubectl label --context=$CTX_CLUSTER3 namespace sample istio-injection=enabled
    namespace/sample created
    namespace/sample labeled
    ```

1.  Deploy the `tcp-echo` sample service, version `v2`:

    ```bash
    $ kubectl apply -n sample -l app=tcp-echo,version=v2 --context=$CTX_CLUSTER3 -f https://raw.githubusercontent.com/istio/istio/533221ef3369834ae44eaa4abcddf67c2d3dc549/samples/tcp-echo/tcp-echo.yaml
    $ kubectl apply -n sample -l service=tcp-echo --context=$CTX_CLUSTER3 -f https://raw.githubusercontent.com/istio/istio/533221ef3369834ae44eaa4abcddf67c2d3dc549/samples/tcp-echo/tcp-echo.yaml
    deployment.apps/tcp-echo-v2 created
    service/tcp-echo created
    ```

1.  Create a destination rule for `tcp-echo`:

    ```bash
    $ kubectl apply --context=$CTX_CLUSTER3 -n sample -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: DestinationRule
    metadata:
      name: tcp-echo
    spec:
      host: tcp-echo
      trafficPolicy:
        tls:
          mode: ISTIO_MUTUAL
    EOF
    ```

1.  Deploy the `tcp-hello-echo` sample service:

    ```bash
    $ kubectl apply -n sample --context=$CTX_CLUSTER3 -f https://raw.githubusercontent.com/istio/istio/533221ef3369834ae44eaa4abcddf67c2d3dc549/samples/tcp-echo/tcp-hello-echo.yaml
    deployment.apps/tcp-hello-echo created
    service/tcp-hello-echo created
    ```

1.  Create a destination rule for `tcp-hello-echo`:

    ```bash
    $ kubectl apply --context=$CTX_CLUSTER3 -n sample -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: DestinationRule
    metadata:
      name: tcp-hello-echo
    spec:
      host: tcp-hello-echo
      trafficPolicy:
        tls:
          mode: ISTIO_MUTUAL
    EOF
    ```

### Deploy sleep samples in all the clusters and test that the services are accessed in each cluster, locally

1.  In each of the clusters, deploy the [sleep]({{< github_tree >}}/samples/sleep) sample app to use as a test source
    for sending requests. (If you already have the sleep app deployed, no need to delete it)

    ```bash
    $ kubectl apply -f samples/sleep/sleep.yaml --context=$CTX_CLUSTER1
    $ kubectl apply -f samples/sleep/sleep.yaml --context=$CTX_CLUSTER2
    $ kubectl apply -f samples/sleep/sleep.yaml --context=$CTX_CLUSTER3
    ```

1.  Wait until the `sleep` apps are running and the previous versions, if any, are terminated:

    ```bash
    $ kubectl get pod -l app=sleep --context=$CTX_CLUSTER1
    $ kubectl get pod -l app=sleep --context=$CTX_CLUSTER2
    $ kubectl get pod -l app=sleep --context=$CTX_CLUSTER3
    NAME                     READY   STATUS    RESTARTS   AGE
    sleep-666475687f-f42ft   2/2     Running   0          4m8s
    NAME                     READY   STATUS    RESTARTS   AGE
    sleep-666475687f-hsnzx   2/2     Running   0          4m6s
    NAME                     READY   STATUS    RESTARTS   AGE
    sleep-666475687f-h6t7d   2/2     Running   0          4m6s
    ```

1.  Test accessing the `tcp-echo` service in the second cluster locally:

    ```bash
    $ kubectl exec -it $(kubectl get pod -l app=sleep -o jsonpath='{.items..metadata.name}' --context=$CTX_CLUSTER2) -c sleep --context=$CTX_CLUSTER2 -- sh -c 'echo world | nc tcp-echo.sample.svc.cluster.local 9000'
    one world
    ```

1.  Test accessing the `tcp-echo` service in the third cluster locally:

    ```bash
    $ kubectl exec -it $(kubectl get pod -l app=sleep -o jsonpath='{.items..metadata.name}' --context=$CTX_CLUSTER3) -c sleep --context=$CTX_CLUSTER3 -- sh -c 'echo world | nc tcp-echo.sample.svc.cluster.local 9000'
    two world
    ```

1.  Test accessing the `tcp-hello-echo` service in the third cluster locally:

    ```bash
    $ kubectl exec -it $(kubectl get pod -l app=sleep -o jsonpath='{.items..metadata.name}' --context=$CTX_CLUSTER3) -c sleep --context=$CTX_CLUSTER3 -- sh -c 'echo world | nc tcp-hello-echo.sample.svc.cluster.local 9000'
    hello world
    ```

After completing the steps until now, you get the following setting (the `sleep` containers in the second and third
clusters omitted):

{{< image width="100%" link="./MeshFederationTCP_1.svg" caption="The three clusters with the deployed services" >}}

## Perform one-time setup of private gateways

Follow the instructions in the [Setup](/docs/examples/multimesh/multimesh-common-setup/#setup) section of [common multi-mesh setup](/docs/examples/multimesh/multimesh-common-setup).

Once you finish the instructions above, you get the following setting:

{{< image width="100%" link="./MeshFederationTCP_2.svg" caption="The three clusters with the deployed services and gateways" >}}

## Expose and consume services (on a per-service basis)

In the following sections you expose the service on two pre-defined ports, 31400 and 31401. The ports were defined in the
private ingress gateway's configurations in the
[Deploy a private ingress gateway in the second cluster](/docs/examples/multimesh/multimesh-common-setup/#deploy-a-private-ingress-gateway-in-the-second-cluster)
and in the
[Deploy a private ingress gateway in the third cluster](/docs/examples/multimesh/multimesh-common-setup/#deploy-a-private-ingress-gateway-in-the-third-cluster)
sections, step 3. You need to expose more services, add more ports to the definitions of the ingress gateways.

### Expose the tcp service in the second cluster

1.  Define an ingress `Gateway`:

    ```bash
    $ kubectl apply --context=$CTX_CLUSTER2 -n istio-private-gateways -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: Gateway
    metadata:
      name: istio-private-ingressgateway
    spec:
      selector:
        istio: private-ingressgateway
      servers:
      - port:
          number: 31400
          name: tls-1
          protocol: TLS
        tls:
          mode: MUTUAL
          serverCertificate: /etc/istio/c2.example.com/certs/tls.crt
          privateKey: /etc/istio/c2.example.com/certs/tls.key
          caCertificates: /etc/istio/example.com/certs/example.com.crt
        hosts:
        - "*"
    EOF
    ```

1.  Configure routing to `tcp-echo`:

    ```bash
    $ kubectl apply --context=$CTX_CLUSTER2 -n istio-private-gateways -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: privately-exposed-services
    spec:
      hosts:
      - "*"
      gateways:
      - istio-private-ingressgateway
      tcp:
      - match:
        - port: 31400
        route:
        - destination:
            host: tcp-echo.sample.svc.cluster.local
            port:
              number: 9000
    EOF
    ```

1.  Test your configuration by accessing the exposed service. The `openssl` command below uses the certificate and the
    private key of `cluster1`:

    ```bash
    $ echo world | openssl s_client -host $CLUSTER2_INGRESS_HOST -port 31400 -cert c1.example.com.crt -key c1.example.com.key -CAfile example.com.crt -quiet
    one world
    ```

    Kill the command above by pressing `Ctrl-C`.

### Consume TCP echo in the first cluster

Bind `tcp-echo` exposed from `cluster2` as `echo.default.svc.cluster.local` in `cluster1`. Note that the name of the
service in `cluster1` is different from the name of the service in `cluster2`.

1.  Create a Kubernetes service for `c2.example.com` since it is not an existing hostname. In the real life, you
    would use the real hostname of your cluster.

    ```bash
    $ kubectl apply --context=$CTX_CLUSTER1 -n istio-private-gateways -f - <<EOF
    kind: Service
    apiVersion: v1
    metadata:
      name: c2-example-com-31400
    spec:
      type: ExternalName
      externalName: $CLUSTER2_INGRESS_HOST
      ports:
      - name: tls-1
        protocol: TCP
        port: 31400
    EOF
    ```

1.  Create a destination rule for `c2.example.com`:

    ```bash
    $ kubectl apply --context=$CTX_CLUSTER1 -n istio-private-gateways -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: DestinationRule
    metadata:
      name: c2-example-com-31400
    spec:
      host: c2-example-com-31400
      exportTo:
      - "."
      trafficPolicy:
        loadBalancer:
          simple: ROUND_ROBIN
        portLevelSettings:
        - port:
            number: 31400
          tls:
            mode: MUTUAL
            clientCertificate: /etc/istio/c1.example.com/certs/tls.crt
            privateKey: /etc/istio/c1.example.com/certs/tls.key
            caCertificates: /etc/istio/example.com/certs/example.com.crt
            sni: c2.example.com
    EOF
    ```

1.  To handle DNS, create a Kubernetes service for `echo.default.svc.cluster.local`, port 9001. Note that the port in
    in the consuming cluster can be different from the port in the exposing cluster.

    ```bash
    $ kubectl apply --context=$CTX_CLUSTER1 -f - <<EOF
    kind: Service
    apiVersion: v1
    metadata:
      name: echo
    spec:
      ports:
      - name: tcp
        protocol: TCP
        port: 9001
    EOF
    ```

1.  Create a Kubernetes service for `echo-c2.default.svc.cluster.local`, to be used by the egress gateway:

    ```bash
    $ kubectl apply --context=$CTX_CLUSTER1 -f - <<EOF
    kind: Service
    apiVersion: v1
    metadata:
      name: echo-c2
    spec:
      ports:
      - name: tcp
        protocol: TCP
        port: 31400
    EOF
    ```

1.  Create an egress `Gateway` for `echo-c2.default.svc.cluster.local`, port 31400, and a destination rule for
    traffic directed to the egress gateway.

    ```bash
    $ kubectl apply --context=$CTX_CLUSTER1 -n istio-private-gateways -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: Gateway
    metadata:
      name: istio-private-egressgateway-echo-c2
    spec:
      selector:
        istio: private-egressgateway
      servers:
      - port:
          number: 31400
          name: tls
          protocol: TLS
        hosts:
        - echo-c2.default.svc.cluster.local
        tls:
          mode: MUTUAL
          serverCertificate: /etc/certs/cert-chain.pem
          privateKey: /etc/certs/key.pem
          caCertificates: /etc/certs/root-cert.pem
    ---
    apiVersion: networking.istio.io/v1alpha3
    kind: DestinationRule
    metadata:
      name: istio-private-egressgateway
    spec:
      host: istio-private-egressgateway.istio-private-gateways.svc.cluster.local
      subsets:
      - name: echo-c2
        trafficPolicy:
          loadBalancer:
            simple: ROUND_ROBIN
          portLevelSettings:
          - port:
              number: 31400
            tls:
              mode: ISTIO_MUTUAL
              sni: echo-c2.default.svc.cluster.local
    EOF
    ```

1.  Define a virtual service to direct traffic from the egress gateway to the external service:

    ```bash
    $ kubectl apply --context=$CTX_CLUSTER1 -n istio-private-gateways -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: echo-c2
    spec:
      hosts:
      - echo-c2.default.svc.cluster.local
      gateways:
      - istio-private-egressgateway-echo-c2
      tcp:
      - match:
          - port: 31400
        route:
        - destination:
            host: c2-example-com-31400.istio-private-gateways.svc.cluster.local
            port:
              number: 31400
          weight: 100
    EOF
    ```

1.  Direct the traffic destined to `echo.default.svc.cluster.local` to the private egress gateway:

    ```bash
    $ kubectl apply --context=$CTX_CLUSTER1 -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: echo
    spec:
      hosts:
      - echo
      tcp:
      - match:
        - port: 9001
        route:
        - destination:
            host: istio-private-egressgateway.istio-private-gateways.svc.cluster.local
            subset: echo-c2
            port:
              number: 31400
          weight: 100
    EOF
    ```

1.  Test your configuration by accessing the exposed service by the `nc` command:

    ```bash
    $ kubectl exec -it $(kubectl get pod -l app=sleep -o jsonpath='{.items..metadata.name}' --context=$CTX_CLUSTER1) -c sleep --context=$CTX_CLUSTER1 -- sh -c 'echo world | nc echo 9001'
    one world
    ```

### Expose the tcp services in the third cluster

1.  Define an ingress `Gateway`:

    ```bash
    $ kubectl apply --context=$CTX_CLUSTER3 -n istio-private-gateways -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: Gateway
    metadata:
      name: istio-private-ingressgateway
    spec:
      selector:
        istio: private-ingressgateway
      servers:
      - port:
          number: 31400
          name: tls-1
          protocol: TLS
        tls:
          mode: MUTUAL
          serverCertificate: /etc/istio/c3.example.com/certs/tls.crt
          privateKey: /etc/istio/c3.example.com/certs/tls.key
          caCertificates: /etc/istio/example.com/certs/example.com.crt
        hosts:
        - "*"
      - port:
          number: 31401
          name: tls-2
          protocol: TLS
        tls:
          mode: MUTUAL
          serverCertificate: /etc/istio/c3.example.com/certs/tls.crt
          privateKey: /etc/istio/c3.example.com/certs/tls.key
          caCertificates: /etc/istio/example.com/certs/example.com.crt
        hosts:
        - "*"
    EOF
    ```

1.  Configure routing to `tcp-echo` and `tcp-hello-echo`:

    ```bash
    $ kubectl apply --context=$CTX_CLUSTER3 -n istio-private-gateways -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: privately-exposed-services
    spec:
      hosts:
      - "*"
      gateways:
      - istio-private-ingressgateway
      tcp:
      - match:
        - port: 31400
        route:
        - destination:
            host: tcp-echo.sample.svc.cluster.local
            port:
              number: 9000
      - match:
        - port: 31401
        route:
        - destination:
            host: tcp-hello-echo.sample.svc.cluster.local
            port:
              number: 9000
    EOF
    ```

1.  Test your configuration by accessing `tcp-echo` The `openssl` command below uses the certificate and the
    private key of `cluster1`:

    ```bash
    $ echo world | openssl s_client -host $CLUSTER3_INGRESS_HOST -port 31400 -cert c1.example.com.crt -key c1.example.com.key -CAfile example.com.crt -quiet
    two world
    ```

    Kill the command above by pressing `Ctrl-C`.

1.  Test your configuration by accessing `tcp-hello-echo`. The `openssl` command below uses the certificate and the
    private key of `cluster1`:

    ```bash
    $ echo world | openssl s_client -host $CLUSTER3_INGRESS_HOST -port 31401 -cert c1.example.com.crt -key c1.example.com.key -CAfile example.com.crt -quiet
    hello world
    ```

    Kill the command above by pressing `Ctrl-C`.

### Consume TCP echo from the third cluster in the first cluster

Bind `tcp-echo` exposed from `cluster3` as `echo.default.svc.cluster.local` in `cluster1`.

1.  Create a Kubernetes service for `c3.example.com` since it is not an existing hostname. In the real life, you
    would use the real hostname of your cluster. You have to create a service per port, otherwise Istio will route
    the requests to all the ports of the service.

    ```bash
    $ kubectl apply --context=$CTX_CLUSTER1 -n istio-private-gateways -f - <<EOF
    kind: Service
    apiVersion: v1
    metadata:
      name: c3-example-com-31400
    spec:
      type: ExternalName
      externalName: $CLUSTER3_INGRESS_HOST
      ports:
      - name: tls-1
        protocol: TCP
        port: 31400
    EOF
    ```

1.  Create a destination rule for `c3.example.com`:

    ```bash
    $ kubectl apply --context=$CTX_CLUSTER1 -n istio-private-gateways -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: DestinationRule
    metadata:
      name: c3-example-com-31400
    spec:
      host: c3-example-com-31400
      exportTo:
      - "."
      trafficPolicy:
        loadBalancer:
          simple: ROUND_ROBIN
        portLevelSettings:
        - port:
            number: 31400
          tls:
            mode: MUTUAL
            clientCertificate: /etc/istio/c1.example.com/certs/tls.crt
            privateKey: /etc/istio/c1.example.com/certs/tls.key
            caCertificates: /etc/istio/example.com/certs/example.com.crt
            sni: c3.example.com
    EOF
    ```

1.  Create a Kubernetes service for `echo-c3.default.svc.cluster.local`, to be used by the egress gateway:

    ```bash
    $ kubectl apply --context=$CTX_CLUSTER1 -f - <<EOF
    kind: Service
    apiVersion: v1
    metadata:
      name: echo-c3
    spec:
      ports:
      - name: tcp
        protocol: TCP
        port: 31400
    EOF
    ```

1.  Create an egress `Gateway` to handle `echo-c3.default.svc.cluster.local`, port 31400 and update the destination rule
    you created previously to handle traffic to it.

    ```bash
    $ kubectl apply --context=$CTX_CLUSTER1 -n istio-private-gateways -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: Gateway
    metadata:
      name: istio-private-egressgateway-echo-c3
    spec:
      selector:
        istio: private-egressgateway
      servers:
      - port:
          number: 31400
          name: tls
          protocol: TLS
        hosts:
        - echo-c3.default.svc.cluster.local
        tls:
          mode: MUTUAL
          serverCertificate: /etc/certs/cert-chain.pem
          privateKey: /etc/certs/key.pem
          caCertificates: /etc/certs/root-cert.pem
    ---
    apiVersion: networking.istio.io/v1alpha3
    kind: DestinationRule
    metadata:
      name: istio-private-egressgateway
    spec:
      host: istio-private-egressgateway.istio-private-gateways.svc.cluster.local
      subsets:
      - name: echo-c2
        trafficPolicy:
          loadBalancer:
            simple: ROUND_ROBIN
          portLevelSettings:
          - port:
              number: 31400
            tls:
              mode: ISTIO_MUTUAL
              sni: echo-c2.default.svc.cluster.local
      - name: echo-c3
        trafficPolicy:
          loadBalancer:
            simple: ROUND_ROBIN
          portLevelSettings:
          - port:
              number: 31400
            tls:
              mode: ISTIO_MUTUAL
              sni: echo-c3.default.svc.cluster.local
    EOF
    ```

1.  Define a virtual service to direct traffic from the egress gateway to the external service:

    ```bash
    $ kubectl apply --context=$CTX_CLUSTER1 -n istio-private-gateways -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: echo-c3
    spec:
      hosts:
      - echo-c3.default.svc.cluster.local
      gateways:
      - istio-private-egressgateway-echo-c3
      tcp:
      - match:
          - port: 31400
        route:
        - destination:
            host: c3-example-com-31400.istio-private-gateways.svc.cluster.local
            port:
              number: 31400
          weight: 100
    EOF
    ```

1.  Direct the traffic destined to `echo.default.svc.cluster.local` to the private egress gateway, while load balancing
    50:50 between `tcp-echo` in the second and third clusters:

    ```bash
    $ kubectl apply --context=$CTX_CLUSTER1 -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: echo
    spec:
      hosts:
      - echo
      tcp:
      - match:
        - port: 9001
        route:
        - destination:
            host: istio-private-egressgateway.istio-private-gateways.svc.cluster.local
            subset: echo-c2
            port:
              number: 31400
          weight: 50
        - destination:
            host: istio-private-egressgateway.istio-private-gateways.svc.cluster.local
            subset: echo-c3
            port:
              number: 31400
          weight: 50
    EOF
    ```

1.  Test your configuration by accessing the exposed service by the `nc` command:

    ```bash
    $ kubectl exec -it $(kubectl get pod -l app=sleep -o jsonpath='{.items..metadata.name}' --context=$CTX_CLUSTER1) -c sleep --context=$CTX_CLUSTER1 -- sh -c 'for i in `seq 1 10`; do date | nc echo 9001; done'
    one Mon Nov  4 04:20:21 UTC 2019
    two Mon Nov  4 04:20:21 UTC 2019
    two Mon Nov  4 04:20:22 UTC 2019
    two Mon Nov  4 04:20:22 UTC 2019
    one Mon Nov  4 04:20:23 UTC 2019
    two Mon Nov  4 04:20:23 UTC 2019
    one Mon Nov  4 04:20:23 UTC 2019
    two Mon Nov  4 04:20:24 UTC 2019
    one Mon Nov  4 04:20:24 UTC 2019
    one Mon Nov  4 04:20:24 UTC 2019
    ```

### Consume TCP hello echo from the third cluster in the first cluster

Bind `tcp-hello-echo` exposed from the third cluster as `tcp-hello-echo.default.svc.cluster.local` in `cluster1`.

1.  Create a Kubernetes service for `c3.example.com`, port 31401:

    ```bash
    $ kubectl apply --context=$CTX_CLUSTER1 -n istio-private-gateways -f - <<EOF
    kind: Service
    apiVersion: v1
    metadata:
      name: c3-example-com-31401
    spec:
      type: ExternalName
      externalName: $CLUSTER3_INGRESS_HOST
      ports:
      - name: tls-2
        protocol: TCP
        port: 31401
    EOF
    ```

1.  Create a destination rule for `c3.example.com`, port 31401:

    ```bash
    $ kubectl apply --context=$CTX_CLUSTER1 -n istio-private-gateways -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: DestinationRule
    metadata:
      name: c3-example-com-31401
    spec:
      host: c3-example-com-31401
      exportTo:
      - "."
      trafficPolicy:
        loadBalancer:
          simple: ROUND_ROBIN
        portLevelSettings:
        - port:
            number: 31401
          tls:
            mode: MUTUAL
            clientCertificate: /etc/istio/c1.example.com/certs/tls.crt
            privateKey: /etc/istio/c1.example.com/certs/tls.key
            caCertificates: /etc/istio/example.com/certs/example.com.crt
            sni: c3.example.com
    EOF
    ```

1.  To handle DNS, create a Kubernetes service for `tcp-hello-echo.default.svc.cluster.local`, port 9001.

    ```bash
    $ kubectl apply --context=$CTX_CLUSTER1 -f - <<EOF
    kind: Service
    apiVersion: v1
    metadata:
      name: tcp-hello-echo
    spec:
      ports:
      - name: tcp
        protocol: TCP
        port: 9001
    EOF
    ```

1.  Create a Kubernetes service for `tcp-hello-echo-c3.default.svc.cluster.local`, to be used by the egress gateway:

    ```bash
    $ kubectl apply --context=$CTX_CLUSTER1 -f - <<EOF
    kind: Service
    apiVersion: v1
    metadata:
      name: tcp-hello-echo-c3
    spec:
      ports:
      - name: tcp
        protocol: TCP
        port: 31401
    EOF
    ```

1.  Create the egress `Gateway` items to handle `tcp-hello-echo-c3.default.svc.cluster.local`, port 31401, and update
    the destination rule you created previously to handle the traffic to the service.

    ```bash
    $ kubectl apply --context=$CTX_CLUSTER1 -n istio-private-gateways -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: Gateway
    metadata:
      name: istio-private-egressgateway-tcp-hello-echo-c3
    spec:
      selector:
        istio: private-egressgateway
      servers:
      - port:
          number: 31401
          name: tls
          protocol: TLS
        hosts:
        - tcp-hello-echo-c3.default.svc.cluster.local
        tls:
          mode: MUTUAL
          serverCertificate: /etc/certs/cert-chain.pem
          privateKey: /etc/certs/key.pem
          caCertificates: /etc/certs/root-cert.pem
    ---
    apiVersion: networking.istio.io/v1alpha3
    kind: DestinationRule
    metadata:
      name: istio-private-egressgateway
    spec:
      host: istio-private-egressgateway.istio-private-gateways.svc.cluster.local
      subsets:
      - name: echo-c2
        trafficPolicy:
          loadBalancer:
            simple: ROUND_ROBIN
          portLevelSettings:
          - port:
              number: 31400
            tls:
              mode: ISTIO_MUTUAL
              sni: echo-c2.default.svc.cluster.local
      - name: echo-c3
        trafficPolicy:
          loadBalancer:
            simple: ROUND_ROBIN
          portLevelSettings:
          - port:
              number: 31400
            tls:
              mode: ISTIO_MUTUAL
              sni: echo-c3.default.svc.cluster.local
      - name: tcp-hello-echo-c3
        trafficPolicy:
          loadBalancer:
            simple: ROUND_ROBIN
          portLevelSettings:
          - port:
              number: 31401
            tls:
              mode: ISTIO_MUTUAL
              sni: tcp-hello-echo-c3.default.svc.cluster.local
    EOF
    ```

1.  Define a virtual service to direct traffic from the egress gateway to the external service:

    ```bash
    $ kubectl apply --context=$CTX_CLUSTER1 -n istio-private-gateways -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: tcp-hello-echo-c3
    spec:
      hosts:
      - tcp-hello-echo-c3.default.svc.cluster.local
      gateways:
      - istio-private-egressgateway-tcp-hello-echo-c3
      tcp:
      - match:
          - port: 31401
        route:
        - destination:
            host: c3-example-com-31401.istio-private-gateways.svc.cluster.local
            port:
              number: 31401
          weight: 100
    EOF
    ```

1.  Direct the traffic destined to `tcp-hello-echo.default.svc.cluster.local` to the private egress gateway:

    ```bash
    $ kubectl apply --context=$CTX_CLUSTER1 -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: tcp-hello-echo
    spec:
      hosts:
      - tcp-hello-echo
      tcp:
      - match:
        - port: 9001
        route:
        - destination:
            host: istio-private-egressgateway.istio-private-gateways.svc.cluster.local
            subset: tcp-hello-echo-c3
            port:
              number: 31401
          weight: 100
    EOF
    ```

1.  Test your configuration by accessing the exposed service by the `nc` command:

    ```bash
    $ kubectl exec -it $(kubectl get pod -l app=sleep -o jsonpath='{.items..metadata.name}' --context=$CTX_CLUSTER1) -c sleep --context=$CTX_CLUSTER1 -- sh -c 'echo world | nc tcp-hello-echo 9001'
    hello world
    ```

1.  Check that the `tcp-echo` services are still accessible:

    ```bash
    $ kubectl exec -it $(kubectl get pod -l app=sleep -o jsonpath='{.items..metadata.name}' --context=$CTX_CLUSTER1) -c sleep --context=$CTX_CLUSTER1 -- sh -c 'for i in `seq 1 10`; do date | nc echo 9001; done'
    one Mon Nov  4 04:20:21 UTC 2019
    two Mon Nov  4 04:20:21 UTC 2019
    two Mon Nov  4 04:20:22 UTC 2019
    two Mon Nov  4 04:20:22 UTC 2019
    one Mon Nov  4 04:20:23 UTC 2019
    two Mon Nov  4 04:20:23 UTC 2019
    one Mon Nov  4 04:20:23 UTC 2019
    two Mon Nov  4 04:20:24 UTC 2019
    one Mon Nov  4 04:20:24 UTC 2019
    one Mon Nov  4 04:20:24 UTC 2019
    ```

You have now the setting as shown in the diagram below:

{{< image width="100%" link="./MeshFederationTCP_3.svg" caption="The three clusters with configured exposure and consumption" >}}

## Apply Istio RBAC on the third cluster

In this section you harden the security of your third cluster by applying
[Istio RBAC](/docs/concepts/security/#authorization) on the `sample` namespace and on the namespace of the private
ingress gateway. The goal is to control which service is allowed to call which specific service, which services are
allowed to be accessed from the outside through the private ingress gateway and which external clusters are allowed to
access which specific services. The goal is to reduce the possible attack vector in case some of the internal services
or the external clusters is compromised.

Access control is enforced at the entrance to the cluster and also inside the cluster, following the
[Defense-in-depth principle](https://en.wikipedia.org/wiki/Defense_in_depth_(computing)) and implementing
[boundary protection](https://insights.sei.cmu.edu/insider-threat/2018/09/cybersecurity-architecture-part-2-system-boundary-and-boundary-protection.html).

The security is hardened in two phases in the next subsections:

1. You enable Istio RBAC on the `sample` namespace, and declare
that only `tcp-echo` and `tcp-hello-echo` are allowed to be accessed from the outside through the private ingress gateway.
You also allow `tcp-hello-echo` to be called by the `sleep` sample.

1. You enable Istio RBAC on the `istio-private-gateways` namespace and declare that only cluster `c1` is allowed to
access `tcp-echo` and only cluster `c4` is allowed to access `tcp-hello-echo` through the private ingress gateway.

Istio will deny all the unspecified access.

### Apply Istio RBAC on the `sample` namespace

1.   Create Istio service roles for read access to `helloworld` and `httpbin`.

    ```bash
    $ kubectl apply  --context=$CTX_CLUSTER3 -n sample -f - <<EOF
    apiVersion: rbac.istio.io/v1alpha1
    kind: ServiceRole
    metadata:
      name: tcp-echo-accessor
    spec:
      rules:
      - services: ["tcp-echo.sample.svc.cluster.local"]
        constraints:
        - key: "destination.port"
          values: ["9000"]
    ---
    apiVersion: rbac.istio.io/v1alpha1
    kind: ServiceRole
    metadata:
      name: tcp-hello-echo-accessor
    spec:
      rules:
      - services: ["tcp-hello-echo.sample.svc.cluster.local"]
        constraints:
        - key: "destination.port"
          values: ["9000"]
    EOF
    ```

1.  Create role bindings to enable read access to the services according to the requirements of the application.
    `tcp-echo` may be called from the private ingress gateway, `tcp-hello-echo` can also be called from the private ingress
    gateway and also from `sleep`. These role bindings forbid, for example, calls from `tcp-echo` to `tcp-hello-echo`.

    ```bash
    $ kubectl apply --context=$CTX_CLUSTER3 -n sample -f - <<EOF
    apiVersion: rbac.istio.io/v1alpha1
    kind: ServiceRoleBinding
    metadata:
      name: tcp-echo-accessor
    spec:
      subjects:
      - user: "cluster.local/ns/istio-private-gateways/sa/istio-private-ingressgateway-service-account"
      roleRef:
        kind: ServiceRole
        name: tcp-echo-accessor
    ---
    apiVersion: rbac.istio.io/v1alpha1
    kind: ServiceRoleBinding
    metadata:
      name: tcp-hello-echo-accessor
    spec:
      subjects:
      - user: "cluster.local/ns/default/sa/sleep"
      - user: "cluster.local/ns/istio-private-gateways/sa/istio-private-ingressgateway-service-account"
      roleRef:
        kind: ServiceRole
        name: tcp-hello-echo-accessor
    EOF
    ```

1.  Enable [Istio RBAC](/docs/concepts/security/#authorization) on the `sample` namespace.

    {{< warning >}}
    If you already enabled Istio RBAC on some of your namespaces, add `sample` to the list of the included namespaces.
    The command below assumes that `sample` is the only namespace you enabled RBAC on.
    {{< /warning >}}

    ```bash
    $ kubectl apply --context=$CTX_CLUSTER3 -f - <<EOF
    apiVersion: "rbac.istio.io/v1alpha1"
    kind: ClusterRbacConfig
    metadata:
      name: default
      namespace: istio-system
    spec:
      mode: ON_WITH_INCLUSION
      inclusion:
        namespaces: [ sample ]
    EOF
    ```

1.  Check that unauthorized access is denied. Send a request from `sleep` to `tcp-echo`:

    ```bash
    $ kubectl exec -it $(kubectl get pod -l app=sleep -o jsonpath='{.items..metadata.name}' --context=$CTX_CLUSTER3) -c sleep --context=$CTX_CLUSTER3 -- sh -c 'echo world | nc tcp-echo.sample.svc.cluster.local 9000'
    ```

    This time no output is produced.

1.  Check that `sleep` can call `tcp-hello-echo`, since it is allowed by the policy:

    ```bash
    $ kubectl exec -it $(kubectl get pod -l app=sleep -o jsonpath='{.items..metadata.name}' --context=$CTX_CLUSTER3) -c sleep --context=$CTX_CLUSTER3 -- sh -c 'echo world | nc tcp-hello-echo.sample.svc.cluster.local 9000'
    hello world
    ```

1.  Check the services can be called from the first cluster as previously. Test accessing the
    `tcp-hello-echo.sample.svc.cluster.local` service from the first cluster:

    ```bash
    $ kubectl exec -it $(kubectl get pod -l app=sleep -o jsonpath='{.items..metadata.name}' --context=$CTX_CLUSTER1) -c sleep --context=$CTX_CLUSTER1 -- sh -c 'echo world | nc tcp-hello-echo 9001'
    hello world
    ```

1.  Send ten requests to `echo.sample.svc.cluster.local` service from the first cluster, they should be accessed as
    previously:

    ```bash
    $ kubectl exec -it $(kubectl get pod -l app=sleep -o jsonpath='{.items..metadata.name}' --context=$CTX_CLUSTER1) -c sleep --context=$CTX_CLUSTER1 -- sh -c 'for i in `seq 1 10`; do date | nc echo 9001; done'
    one Mon Nov  4 04:20:21 UTC 2019
    two Mon Nov  4 04:20:21 UTC 2019
    two Mon Nov  4 04:20:22 UTC 2019
    two Mon Nov  4 04:20:22 UTC 2019
    one Mon Nov  4 04:20:23 UTC 2019
    two Mon Nov  4 04:20:23 UTC 2019
    one Mon Nov  4 04:20:23 UTC 2019
    two Mon Nov  4 04:20:24 UTC 2019
    one Mon Nov  4 04:20:24 UTC 2019
    one Mon Nov  4 04:20:24 UTC 2019
    ```

### Enable RBAC on the ingress gateway

1.  Currently, the only identities Istio RBAC can handle are the identities of Istio mutual TLS. In this case, you want
    to control the traffic for non-Istio mutual TLS from external clusters. To do that, you can use Envoy's filter [`envoy.filters.http.rbac`](https://www.envoyproxy.io/docs/envoy/latest/configuration/http_filters/rbac_filter).
    To instruct Istio to use that filter, create an
    [`EnvoyFilter`](/docs/reference/config/networking/v1alpha3/envoy-filter/).

    Specify the RBAC policy that allows the `c1` cluster to access port 31400 of the istio-ingress gateway and the `c4`
    cluster to access port 31401. Here access specified is per port, in this case port uniquely identifies an exposed
    TCP service.

    ```bash
    $ kubectl apply  --context=$CTX_CLUSTER3 -n istio-private-gateways -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: EnvoyFilter
    metadata:
      name: private-ingress-rbac
    spec:
      workloadSelector:
        labels:
          app: istio-private-ingressgateway
      configPatches:
      - applyTo: NETWORK_FILTER
        match:
          context: GATEWAY
          listener:
            portNumber: 31400
            filterChain:
              filter:
                name: mixer
        patch:
          operation: INSERT_BEFORE
          value:
             name: envoy.filters.network.rbac
             config:
               rules:
                 policies:
                   ingress-accessor:
                     permissions:
                     - and_rules:
                         rules:
                         - or_rules:
                             rules:
                             - destination_port: 31400
                     principals:
                     - and_ids:
                         ids:
                         - authenticated:
                             principal_name:
                               exact: spiffe://c1.example.com/istio-private-egressgateway
               stat_prefix: "tcp."
      - applyTo: NETWORK_FILTER
        match:
          context: GATEWAY
          listener:
            portNumber: 31401
            filterChain:
              filter:
                name: mixer
        patch:
          operation: INSERT_BEFORE
          value:
             name: envoy.filters.network.rbac
             config:
               rules:
                 policies:
                   ingress-accessor:
                     permissions:
                     - and_rules:
                         rules:
                         - or_rules:
                             rules:
                             - destination_port: 31401
                     principals:
                     - and_ids:
                         ids:
                         - authenticated:
                             principal_name:
                               exact: spiffe://c4.example.com/istio-private-egressgateway
               stat_prefix: "tcp."
    EOF
    ```

1.  Perform the following tests by the `curl` command using identities of the `c1` and the `c4` clusters:

    1.  Verify that `tcp-echo` is allowed for the `c1` cluster:

    ```bash
    $ echo world | openssl s_client -host $CLUSTER3_INGRESS_HOST -port 31400 -cert c1.example.com.crt -key c1.example.com.key -CAfile example.com.crt -quiet
    depth=1 O = example Inc., CN = example.com
    verify return:1
    depth=0 O = "example Inc., department 3", CN = c3.example.com
    verify return:1
    two world
    ```

    Kill the command above by pressing `Ctrl-C`.

    1.  Verify that `tcp-echo` is denied for the `c4` cluster:

    ```bash
    $ echo world | openssl s_client -host $CLUSTER3_INGRESS_HOST -port 31400 -cert c4.example.com.crt -key c4.example.com.key -CAfile example.com.crt -quiet
    depth=1 O = example Inc., CN = example.com
    verify return:1
    depth=0 O = "example Inc., department 3", CN = c3.example.com
    verify return:1
    ```

    Note that the `two world` message is not printed.

    1.  Verify that `tcp-hello-echo` is denied for the `c1` cluster:

    ```bash
    $ echo world | openssl s_client -host $CLUSTER3_INGRESS_HOST -port 31401 -cert c1.example.com.crt -key c1.example.com.key -CAfile example.com.crt -quiet
    depth=1 O = example Inc., CN = example.com
    verify return:1
    depth=0 O = "example Inc., department 3", CN = c3.example.com
    verify return:1
    ```

    Note that the `hello world` message is not printed.

    1.  Verify that `tcp-hello-echo` is allowed for the `c4` cluster:

    ```bash
    $ echo world | openssl s_client -host $CLUSTER3_INGRESS_HOST -port 31401 -cert c4.example.com.crt -key c4.example.com.key -CAfile example.com.crt -quiet
    depth=1 O = example Inc., CN = example.com
    verify return:1
    depth=0 O = "example Inc., department 3", CN = c3.example.com
    verify return:1
    hello world
    ```

    Kill the command above by pressing `Ctrl-C`.

1.  Check the services can be called from the first cluster as previously. Test accessing the
    `tcp-hello-echo.sample.svc.cluster.local` service from the first cluster:

    ```bash
    $ kubectl exec -it $(kubectl get pod -l app=sleep -o jsonpath='{.items..metadata.name}' --context=$CTX_CLUSTER1) -c sleep --context=$CTX_CLUSTER1 -- sh -c 'echo world | nc tcp-hello-echo 9001'
    hello world
    ```

1.  Send ten requests to `echo.sample.svc.cluster.local` service from the first cluster, they should be accessed as
    previously:

    ```bash
    $ kubectl exec -it $(kubectl get pod -l app=sleep -o jsonpath='{.items..metadata.name}' --context=$CTX_CLUSTER1) -c sleep --context=$CTX_CLUSTER1 -- sh -c 'for i in `seq 1 10`; do date | nc echo 9001; done'
    one Mon Nov  4 04:20:21 UTC 2019
    two Mon Nov  4 04:20:21 UTC 2019
    two Mon Nov  4 04:20:22 UTC 2019
    two Mon Nov  4 04:20:22 UTC 2019
    one Mon Nov  4 04:20:23 UTC 2019
    two Mon Nov  4 04:20:23 UTC 2019
    one Mon Nov  4 04:20:23 UTC 2019
    two Mon Nov  4 04:20:24 UTC 2019
    one Mon Nov  4 04:20:24 UTC 2019
    one Mon Nov  4 04:20:24 UTC 2019
    ```

## Cleanup

### Disable Istio RBAC in the third cluster

1.  Disable [Istio RBAC](/docs/concepts/security/#authorization) on the `sample` and `istio-private-gateways`
    namespaces.

    {{< warning >}}
    If you enabled Istio RBAC on some of your namespaces, remove `sample`
    from the list of the included namespaces. The command below assumes that you enabled Istio RBAC only on the
    `sample` namespace.
    {{< /warning >}}

    ```bash
    $ kubectl delete --context=$CTX_CLUSTER3 -n istio-system clusterrbacconfig default
    ```

1.  Delete the service roles and service role bindings:

    ```bash
    $ kubectl delete --context=$CTX_CLUSTER3 -n sample servicerolebinding tcp-echo-accessor tcp-hello-echo-accessor
    $ kubectl delete --context=$CTX_CLUSTER3 -n sample servicerole tcp-echo-accessor tcp-hello-echo-accessor
    ```

1.  Delete the Envoy's filter:

    ```bash
    $ kubectl delete envoyfilter private-ingress-rbac -n istio-private-gateways --context=$CTX_CLUSTER3
    ```

### Delete consumption of the services in the first cluster

```bash
$ kubectl delete --context=$CTX_CLUSTER1 virtualservice echo-c2 echo-c3 tcp-hello-echo-c3 -n istio-private-gateways
$ kubectl delete --context=$CTX_CLUSTER1 virtualservice echo tcp-hello-echo
$ kubectl delete --context=$CTX_CLUSTER1 destinationrule istio-private-egressgateway -n istio-private-gateways
$ kubectl delete --context=$CTX_CLUSTER1 gateway istio-private-egressgateway-echo-c2 istio-private-egressgateway-echo-c3 istio-private-egressgateway-tcp-hello-echo-c3 -n istio-private-gateways
$ kubectl delete --context=$CTX_CLUSTER1 destinationrule c2-example-com-31400 c3-example-com-31400 -n istio-private-gateways
$ kubectl delete --context=$CTX_CLUSTER1 service c2-example-com-31400 c3-example-com-31400 -n istio-private-gateways
$ kubectl delete --context=$CTX_CLUSTER1 service echo echo-c2 echo-c3 tcp-hello-echo tcp-hello-echo-c3
```

### Delete exposure of the services in the second cluster

```bash
$ kubectl delete --context=$CTX_CLUSTER2 virtualservice privately-exposed-services -n istio-private-gateways
$ kubectl delete --context=$CTX_CLUSTER2 gateway istio-private-ingressgateway -n istio-private-gateways
```

### Delete exposure of the services in the third cluster

```bash
$ kubectl delete --context=$CTX_CLUSTER3 virtualservice privately-exposed-services -n istio-private-gateways
$ kubectl delete --context=$CTX_CLUSTER3 gateway istio-private-ingressgateway -n istio-private-gateways
```

### Delete the sleep apps:

Undeploy the services:

```bash
$ kubectl delete -f @samples/sleep/sleep.yaml@ --context=$CTX_CLUSTER1
$ kubectl delete -f @samples/sleep/sleep.yaml@ --context=$CTX_CLUSTER2
$ kubectl delete -f @samples/sleep/sleep.yaml@ --context=$CTX_CLUSTER3
```

### Undeploy the TCP service in the second cluster

1.  Undeploy the service:

    ```bash
    $ kubectl delete -n sample -l service=tcp-echo --context=$CTX_CLUSTER2 -f https://raw.githubusercontent.com/istio/istio/533221ef3369834ae44eaa4abcddf67c2d3dc549/samples/tcp-echo/tcp-echo.yaml
    $ kubectl delete -n sample -l app=tcp-echo,version=v1 --context=$CTX_CLUSTER2 -f https://raw.githubusercontent.com/istio/istio/533221ef3369834ae44eaa4abcddf67c2d3dc549/samples/tcp-echo/tcp-echo.yaml
    ```

1.  Delete the destination rule:

    ```bash
    $ kubectl delete destinationrule tcp-echo --context=$CTX_CLUSTER2 -n sample
    ```

1.  Delete the `sample` namespace:

    ```bash
    $ kubectl delete namespace sample --context=$CTX_CLUSTER2
    ```

### Undeploy the TCP services in the third cluster

1.  Undeploy the services:

    ```bash
    $ kubectl delete -n sample -l service=tcp-echo --context=$CTX_CLUSTER3 -f https://raw.githubusercontent.com/istio/istio/533221ef3369834ae44eaa4abcddf67c2d3dc549/samples/tcp-echo/tcp-echo.yaml
    $ kubectl delete -n sample -l app=tcp-echo,version=v2 --context=$CTX_CLUSTER3 -f https://raw.githubusercontent.com/istio/istio/533221ef3369834ae44eaa4abcddf67c2d3dc549/samples/tcp-echo/tcp-echo.yaml
    $ kubectl apply -n sample --context=$CTX_CLUSTER3 -f https://raw.githubusercontent.com/istio/istio/533221ef3369834ae44eaa4abcddf67c2d3dc549/samples/tcp-echo/tcp-hello-echo.yaml
    ```

1.  Delete the destination rules:

    ```bash
    $ kubectl delete destinationrule tcp-echo tcp-hello-echo --context=$CTX_CLUSTER3 -n sample
    ```

1.  Delete the `sample` namespace:

    ```bash
    $ kubectl delete namespace sample --context=$CTX_CLUSTER3
    ```

### Delete the private gateways

Follow the instructions in the [Cleenup](/docs/examples/multimesh/multimesh-common-setup/#cleanup) section of [common multi-mesh setup](/docs/examples/multimesh/multimesh-common-setup).
