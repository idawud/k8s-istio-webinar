# Istio - Installation

Istio architecture:

![Istio Architecture](https://raw.githubusercontent.com/istio/istio.io/7bf371365e4a16a9a13c0e79355fe1eac7f8f99f/content/docs/concepts/what-is-istio/arch.svg?sanitize=true
"Istio Architecture")

## Install Istio

Either download Istio directly from [https://github.com/istio/istio/releases](https://github.com/istio/istio/releases)
or get the latest version by using curl:

```bash
export ISTIO_VERSION="1.1.0"
test -d tmp || mkdir tmp
cd tmp
curl -sL https://git.io/getLatestIstio | sh -
```

Output:

```shell
Downloading istio-1.1.0 from https://github.com/istio/istio/releases/download/1.1.0/istio-1.1.0-linux.tar.gz ...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   614    0   614    0     0    884      0 --:--:-- --:--:-- --:--:--   883
100 15.0M  100 15.0M    0     0  5252k      0  0:00:02  0:00:02 --:--:-- 12.4M
Downloaded into istio-1.1.0:
LICENSE  README.md  bin  install  istio.VERSION  samples  tools
Add /mnt/k8s-istio-webinar/k8s-istio-webinar/tmp/istio-1.1.0/bin to your path; e.g copy paste in your shell and/or ~/.profile:
export PATH="$PATH:/mnt/k8s-istio-webinar/k8s-istio-webinar/tmp/istio-1.1.0/bin"
```

Change the directory to the Istio installation files location:

```bash
cd istio*
```

Install `istioctl`:

```bash
test -x /usr/local/bin/istioctl || sudo mv bin/istioctl /usr/local/bin/
```

Install the `istio-init` chart to bootstrap all the Istio's CRDs:

```bash
helm install install/kubernetes/helm/istio-init --wait \
--name istio-init --namespace istio-system --set certmanager.enabled=true
sleep 30
```

Install [Istio](https://istio.io/) with add-ons ([Kiali](https://www.kiali.io/),
[Jaeger](https://www.jaegertracing.io/), [Grafana](https://grafana.com/),
[Prometheus](https://prometheus.io/), [cert-manager](https://github.com/jetstack/cert-manager)):

```bash
helm install install/kubernetes/helm/istio --wait --name istio --namespace istio-system \
  --set certmanager.enabled=true \
  --set certmanager.email=petr.ruzicka@gmail.com \
  --set gateways.istio-ingressgateway.sds.enabled=true \
  --set global.k8sIngress.enabled=true \
  --set global.k8sIngress.enableHttps=true \
  --set global.k8sIngress.gatewayName=ingressgateway \
  --set grafana.enabled=true \
  --set kiali.enabled=true \
  --set kiali.createDemoSecret=true \
  --set kiali.contextPath=/ \
  --set kiali.dashboard.grafanaURL=http://grafana.${MY_DOMAIN}/ \
  --set kiali.dashboard.jaegerURL=http://jaeger.${MY_DOMAIN}/ \
  --set servicegraph.enabled=true \
  --set tracing.enabled=true
```

## Create DNS records

Create DNS record `mylabs.dev` for the loadbalancer created by Istio:

```bash
export LOADBALANCER_HOSTNAME=$(kubectl get svc istio-ingressgateway -n istio-system -o jsonpath="{.status.loadBalancer.ingress[0].hostname}")
export CANONICAL_HOSTED_ZONE_NAME_ID=$(aws elb describe-load-balancers --query "LoadBalancerDescriptions[?DNSName==\`$LOADBALANCER_HOSTNAME\`].CanonicalHostedZoneNameID" --output text)
export HOSTED_ZONE_ID=$(aws route53 list-hosted-zones --query "HostedZones[?Name==\`${MY_DOMAIN}.\`].Id" --output text)

envsubst < ../../files/aws_route53-dns_change.json | aws route53 change-resource-record-sets --hosted-zone-id ${HOSTED_ZONE_ID} --change-batch=file:///dev/stdin
```

![Architecture](https://raw.githubusercontent.com/aws-samples/eks-workshop/65b766c494a5b4f5420b2912d8373c4957163541/static/images/crystal.svg?sanitize=true
"Architecture")

## Create SSL certificate using Let's Encrypt

Create `ClusterIssuer` and `Certificate` for Route53 used by cert-manager.
It will allow Let's encrypt to generate certificate. Route53 (DNS) method of
requesting certificate from Let's Encrypt must be used to create wildcard
certificate `*.mylabs.dev` (details [here](https://community.letsencrypt.org/t/wildcard-certificates-via-http-01/51223)).

```bash
export EKS_CERT_MANAGER_ROUTE53_AWS_SECRET_ACCESS_KEY_BASE64=$(echo -n "$EKS_CERT_MANAGER_ROUTE53_AWS_SECRET_ACCESS_KEY" | base64)
cat ../../files/cert-manager-letsencrypt-aws-route53-clusterissuer-certificate.yaml
envsubst < ../../files/cert-manager-letsencrypt-aws-route53-clusterissuer-certificate.yaml | kubectl apply -f -
```

![ACME DNS Challenge](https://b3n.org/wp-content/uploads/2016/09/acme_letsencrypt_dns-01-challenge.png
"ACME DNS Challenge")

([https://b3n.org/intranet-ssl-certificates-using-lets-encrypt-dns-01/](https://b3n.org/intranet-ssl-certificates-using-lets-encrypt-dns-01/))

Let `istio-ingressgateway` to use cert-manager generated certificate via
[SDS](https://www.envoyproxy.io/docs/envoy/v1.5.0/intro/arch_overview/service_discovery#arch-overview-service-discovery-types-sds).
Steps are taken from here [https://istio.io/docs/tasks/traffic-management/ingress/ingress-certmgr/](https://istio.io/docs/tasks/traffic-management/ingress/ingress-certmgr/).

```bash
kubectl -n istio-system patch gateway istio-autogenerated-k8s-ingress \
  --type=json \
  -p="[{"op": "replace", "path": "/spec/servers/1/tls", "value": {"credentialName": "ingress-cert-${LETSENCRYPT_ENVIRONMENT}", "mode": "SIMPLE", "privateKey": "sds", "serverCertificate": "sds"}}]"
```

## Check and configure Istio

Allow the `default` namespace to use Istio injection:

```bash
kubectl label namespace default istio-injection=enabled
```

Check namespaces:

```bash
kubectl get namespace -L istio-injection
```

Output:

```shell
NAME           STATUS   AGE   ISTIO-INJECTION
default        Active   19m   enabled
istio-system   Active   7m
kube-public    Active   19m
kube-system    Active   19m
```

See the Istio components:

```bash
kubectl get --namespace=istio-system svc,deployment,pods,job,horizontalpodautoscaler,destinationrule
```

Output:

```shell
NAME                             TYPE           CLUSTER-IP       EXTERNAL-IP                                                                  PORT(S)                                                                                                                                      AGE
service/grafana                  ClusterIP      10.100.84.93     <none>                                                                       3000/TCP                                                                                                                                     7m
service/istio-citadel            ClusterIP      10.100.203.5     <none>                                                                       8060/TCP,15014/TCP                                                                                                                           7m
service/istio-galley             ClusterIP      10.100.224.231   <none>                                                                       443/TCP,15014/TCP,9901/TCP                                                                                                                   7m
service/istio-ingressgateway     LoadBalancer   10.100.241.162   abd0be556520611e9ac0602dc9c152bf-2144127322.eu-central-1.elb.amazonaws.com   80:31380/TCP,443:31390/TCP,31400:31400/TCP,15029:31705/TCP,15030:30101/TCP,15031:30032/TCP,15032:32493/TCP,15443:31895/TCP,15020:31909/TCP   7m
service/istio-pilot              ClusterIP      10.100.68.4      <none>                                                                       15010/TCP,15011/TCP,8080/TCP,15014/TCP                                                                                                       7m
service/istio-policy             ClusterIP      10.100.24.13     <none>                                                                       9091/TCP,15004/TCP,15014/TCP                                                                                                                 7m
service/istio-sidecar-injector   ClusterIP      10.100.252.24    <none>                                                                       443/TCP                                                                                                                                      7m
service/istio-telemetry          ClusterIP      10.100.103.164   <none>                                                                       9091/TCP,15004/TCP,15014/TCP,42422/TCP                                                                                                       7m
service/jaeger-agent             ClusterIP      None             <none>                                                                       5775/UDP,6831/UDP,6832/UDP                                                                                                                   7m
service/jaeger-collector         ClusterIP      10.100.32.192    <none>                                                                       14267/TCP,14268/TCP                                                                                                                          7m
service/jaeger-query             ClusterIP      10.100.196.113   <none>                                                                       16686/TCP                                                                                                                                    7m
service/kiali                    ClusterIP      10.100.66.131    <none>                                                                       20001/TCP                                                                                                                                    7m
service/prometheus               ClusterIP      10.100.246.253   <none>                                                                       9090/TCP                                                                                                                                     7m
service/servicegraph             ClusterIP      10.100.163.157   <none>                                                                       8088/TCP                                                                                                                                     7m
service/tracing                  ClusterIP      10.100.90.197    <none>                                                                       80/TCP                                                                                                                                       7m
service/zipkin                   ClusterIP      10.100.8.55      <none>                                                                       9411/TCP                                                                                                                                     7m

NAME                                           DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/certmanager              1         1         1            1           7m
deployment.extensions/grafana                  1         1         1            1           7m
deployment.extensions/istio-citadel            1         1         1            1           7m
deployment.extensions/istio-galley             1         1         1            1           7m
deployment.extensions/istio-ingressgateway     1         1         1            1           7m
deployment.extensions/istio-pilot              1         1         1            1           7m
deployment.extensions/istio-policy             1         1         1            1           7m
deployment.extensions/istio-sidecar-injector   1         1         1            1           7m
deployment.extensions/istio-telemetry          1         1         1            1           7m
deployment.extensions/istio-tracing            1         1         1            1           7m
deployment.extensions/kiali                    1         1         1            1           7m
deployment.extensions/prometheus               1         1         1            1           7m
deployment.extensions/servicegraph             1         1         1            1           7m

NAME                                          READY   STATUS      RESTARTS   AGE
pod/certmanager-7478689867-6n8r7              1/1     Running     0          7m
pod/grafana-7b46bf6b7c-w7ms2                  1/1     Running     0          7m
pod/istio-citadel-75fdb679db-v8bqh            1/1     Running     0          7m
pod/istio-galley-c864b5c86-8xfpm              1/1     Running     0          7m
pod/istio-ingressgateway-6cb65d86cb-5ptgp     2/2     Running     0          7m
pod/istio-init-crd-10-stcw2                   0/1     Completed   0          7m
pod/istio-init-crd-11-fgdh9                   0/1     Completed   0          7m
pod/istio-init-crd-certmanager-10-rhmv9       0/1     Completed   0          7m
pod/istio-init-crd-certmanager-11-dv24d       0/1     Completed   0          7m
pod/istio-pilot-f4c98cfbf-pwp45               2/2     Running     0          7m
pod/istio-policy-6cbbd844dd-4dzbx             2/2     Running     2          7m
pod/istio-sidecar-injector-7b47cb4689-5x7ph   1/1     Running     0          7m
pod/istio-telemetry-ccc4df498-w77hk           2/2     Running     2          7m
pod/istio-tracing-75dd89b8b4-frg8w            1/1     Running     0          7m
pod/kiali-7787748c7d-lb454                    1/1     Running     0          7m
pod/prometheus-89bc5668c-54pdj                1/1     Running     0          7m
pod/servicegraph-5d4b49848-cscbp              1/1     Running     1          7m

NAME                                      DESIRED   SUCCESSFUL   AGE
job.batch/istio-init-crd-10               1         1            7m
job.batch/istio-init-crd-11               1         1            7m
job.batch/istio-init-crd-certmanager-10   1         1            7m
job.batch/istio-init-crd-certmanager-11   1         1            7m

NAME                                                       REFERENCE                         TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/istio-ingressgateway   Deployment/istio-ingressgateway   <unknown>/80%   1         5         1          7m
horizontalpodautoscaler.autoscaling/istio-pilot            Deployment/istio-pilot            <unknown>/80%   1         5         1          7m
horizontalpodautoscaler.autoscaling/istio-policy           Deployment/istio-policy           <unknown>/80%   1         5         1          7m
horizontalpodautoscaler.autoscaling/istio-telemetry        Deployment/istio-telemetry        <unknown>/80%   1         5         1          7m

NAME                                                  HOST                                             AGE
destinationrule.networking.istio.io/istio-policy      istio-policy.istio-system.svc.cluster.local      7m
destinationrule.networking.istio.io/istio-telemetry   istio-telemetry.istio-system.svc.cluster.local   7m
```

Configure the Istio services ([Jaeger](https://www.jaegertracing.io/),
[Prometheus](https://prometheus.io/), [Grafana](https://grafana.com/),
[Kiali](https://www.kiali.io/), Servicegraph) to be visible externally:

```bash
envsubst < ../../files/export_services_gateway.yaml | kubectl apply -f -
```

![Istio](../.vuepress/public/istio.svg "Istio")
