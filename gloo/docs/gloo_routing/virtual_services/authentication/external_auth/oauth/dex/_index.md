---
title: Dex and Gloo
weight: 2
description: Integrating Gloo and Dex IDP
---

We will show how to connect [Dex Identify Provider](https://github.com/dexidp/dex) with Gloo.
Dex is an OpenID Connect identity hub. Dex can be used to expose a consistent OpenID Connect interface to your applications
while allowing your users to use their existing identity from various back-ends, include LDAP, SAML, and other OIDC providers.

You can also use Dex for kubernetes itself, to allow for example LDAP logins to work with kubectl.  
this is outside the scope of this document, but you can read more about it [here](https://github.com/dexidp/dex/blob/master/Documentation/kubernetes.md).


This document will focus on deployment with a local cluster (like minikube, or kind). With small changes
these can be applied to a real cluster.

For simplicity we will use Dex with self sign certificate, as they are auto-generated. The same flow
will work if you provide the certificates.

## Install Gloo

That's easy!

```
glooctl install gateway --license-key=$GLOO_KEY
```

See more info [here](/installation/enterprise).

## Install Dex
We will install Dex into the `gloo-system` namespace, and setup the alt-name in the Dex certificate to the 
correct service DNS name (so that later Gloo will trust the Dex service).
```
cat > /tmp/dex-values.yaml <<EOF

https: true
config:
  issuer: https://dex.gloo-system.svc.cluster.local:32000

  staticClients:
  - id: gloo
    redirectURIs:
    - 'http://localhost:8080/callback'
    name: 'GlooApp'
    secret: secretvalue
  
  staticPasswords:
  - email: "admin@example.com"
    # bcrypt hash of the string "password"
    hash: "\$2a\$10\$2b2cU8CPhOTaGrs1HRQuAueS7JTT5ZHsHSzYiFPm1leZck7Mc8T4W"
    username: "admin"
    userID: "08a8684b-db88-4b73-90a9-3cd1661f5466"

certs:
  web:
    altNames:
    - https://dex.gloo-system.svc.cluster.local:32000
    - dex.gloo-system.svc.cluster.local

EOF

helm install --name dex --namespace gloo-system stable/dex -f /tmp/dex-values.yaml
```

## Setup Dex CA in Gloo
Let's setup Gloo's external auth container to trust the Dex Certificate Authority.
We will add an init container that adds the Dex CA cert to the trusted CA certificates.

{{% notice note %}}
You may need to modify the command below to match your version of Gloo.
You can edit your deployment and copy the highlighted parts.
{{% /notice %}}

{{< highlight shell "hl_lines=24-45 48-50" >}}
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: gloo
    gloo: extauth
  name: extauth
  namespace: gloo-system
spec:
  replicas: 1
  selector:
    matchLabels:
      gloo: extauth
  template:
    metadata:
      labels:
        gloo: extauth
      annotations:
        prometheus.io/path: /metrics
        prometheus.io/port: "9091"
        prometheus.io/scrape: "true"
    spec:
      volumes:
      - name: certs
        emptyDir: {}
      - name: ca-certs
        secret:
          secretName: dex-web-server-ca
          items:
          - key: tls.crt
            path: ca.crt
      initContainers:
      - name: add-ca-cert
        image: quay.io/solo-io/extauth-ee:0.18.13
        command:
          - sh
        args:
          - "-c"
          - "cp -r /etc/ssl/certs/* /certs; cat /etc/ssl/certs/ca-certificates.crt /ca-certs/ca.crt > /certs/ca-certificates.crt"
        volumeMounts:
          - name: certs
            mountPath: /certs
          - name: ca-certs
            mountPath: /ca-certs
      containers:
      - image: quay.io/solo-io/extauth-ee:0.18.13
        volumeMounts:
          - name: certs
            mountPath: /etc/ssl/certs/
        imagePullPolicy: Always
        name: extauth
        env:
          - name: POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
          - name: GLOO_ADDRESS
            value: gloo:9977
          - name: SIGNING_KEY
            valueFrom:
              secretKeyRef:
                name: extauth-signing-key
                key: signing-key
          - name: SERVER_PORT
            value: "8083"
          - name: USER_ID_HEADER
            value: "x-user-id"
          - name: START_STATS_SERVER
            value: "true"
      imagePullSecrets:
        - name: solo-io-readerbot-pull-secret
      affinity:
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    gloo: gateway-proxy
                topologyKey: kubernetes.io/hostname
EOF
{{< /highlight >}}

## Test!

Deploy the pet clinic demo app

```shell
kubectl --namespace default apply -f https://raw.githubusercontent.com/solo-io/gloo/v0.8.4/example/petclinic/petclinic.yaml
```

Create Gloo OIDC config with settings matching the ones configured in Dex's config.
```
glooctl create  secret oauth --client-secret secretvalue oauth
glooctl create virtualservice --oidc-auth-app-url http://localhost:8080/ --oidc-auth-callback-path /callback --oidc-auth-client-id gloo --oidc-auth-client-secret-name oauth --oidc-auth-client-secret-namespace gloo-system --oidc-auth-issuer-url https://dex.gloo-system.svc.cluster.local:32000/ oidc-test --namespace gloo-system --enable-oidc-auth
```
Add a route to the pet clinic demo app.
```
glooctl add route --name default --namespace gloo-system --path-prefix / --dest-name default-petclinic-80 --dest-namespace gloo-system
```

As we are testing in a local cluster, add the following to your `/etc/hosts` file:
```
127.0.0.1 dex.gloo-system.svc.cluster.local
```
The OIDC flow redirects the browser to a login page hosts by dex. This line in the hosts file will allow this flow to work, with 
Dex hosted inside our cluster (using `kubectl port-forward`).

{{% notice note %}}
The browser will display a warning when redirecting to the login page, as the Dex CA cert is not trusted
by the browser. We can ignore the warning in this setup. This should be properly address in your 
production setup.
{{% /notice %}}

Port forward to Gloo and Dex:
```
kubectl -n gloo-system port-forward svc/dex 32000:32000 &
kubectl -n gloo-system port-forward svc/gateway-proxy-v2 8080:80 &
```

And finally, open http://localhost:8080 in your browser.
You should see a login page. You can use the user `admin@example.com` and the password `password` to
login successfully to the pet clinic demo app.

## Cleanup
```
helm delete --purge dex

kubectl delete -n gloo-system secret  dex-grpc-ca  dex-grpc-client-tls  dex-grpc-server-tls  dex-web-server-ca  dex-web-server-tls
kubectl delete -n gloo-system vs oidc-test
```