# ocp-service-mesh-foundations

## Installing the Service Mesh involves :
* Installing Elasticsearch, Jaeger, Kiali
* Installing the Service Mesh Operator
* Creating and managing a ServiceMeshControlPlane resource to deploy the Service Mesh control plane
* Creating a ServiceMeshMemberRoll resource to specify the namespaces associated with the Service Mesh.

## Verify Operator's installation 

```
oc get ClusterServiceVersion
NAME                                        DISPLAY                          VERSION              REPLACES                     PHASE
elasticsearch-operator.4.3.5-202003020549   Elasticsearch Operator           4.3.5-202003020549                                Succeeded
jaeger-operator.v1.13.1                     Jaeger Operator                  1.13.1                                            Succeeded
kiali-operator.v1.0.11                      Kiali Operator                   1.0.11               kiali-operator.v1.0.10       Succeeded
servicemeshoperator.v1.0.9                  Red Hat OpenShift Service Mesh   1.0.9                servicemeshoperator.v1.0.8   Succeeded
```

## View status after you install ServiceMeshControlPlane
```
oc get pods -n istio-system 
NAME                                     READY   STATUS    RESTARTS   AGE
grafana-65d9877595-ccmpv                 2/2     Running   0          5m44s
istio-citadel-b8cd4bb56-vs4f7            1/1     Running   0          7m39s
istio-egressgateway-769f47449-vdcsh      1/1     Running   0          6m25s
istio-galley-bb786f9f-xr5xw              1/1     Running   0          7m17s
istio-ingressgateway-6b84495b7c-kfclw    1/1     Running   0          6m25s
istio-pilot-7bfc869bc6-n56gt             2/2     Running   0          6m55s
istio-policy-77cbb47bb6-s6klc            2/2     Running   0          7m7s
istio-sidecar-injector-f4975d85d-99xhb   1/1     Running   0          5m57s
istio-telemetry-5c56bc5746-kmwhk         2/2     Running   0          7m6s
jaeger-645665785f-6vhxt                  2/2     Running   0          7m22s
kiali-86dc5bd4df-tkvvp                   1/1     Running   0          5m21s
prometheus-5856547667-wpzxw              2/2     Running   0          7m34s

```

## Rule Configuration
```
VirtualService		Defines rules that control how requests for service are routed within service mesh
DestinationRule		Configures set of policies to be applied to request after VirtualService routing occurs
ServiceEntry		Enables requests to services outside service mesh
Gateway			Configures load balancer operating at edge of mesh for:
			  	• HTTP/TCP ingress traffic to mesh application
			  	• Egress traffic to external services
Sidecar			Configures sidecar proxies attached to application workloads running inside mesh

```

# RedHat OpenShift Service Mesh Demo (Dynamic Routing and Blue/Green Deployments)

```
export OCP_TUTORIAL_PROJECT="service-mesh-demo"
oc new-project $OCP_TUTORIAL_PROJECT
```

## Create catalog service
```
oc create -f ./catalog/kubernetes/catalog-service-template.yml -n $OCP_TUTORIAL_PROJECT
oc create -f ./catalog/kubernetes/Service.yml -n $OCP_TUTORIAL_PROJECT
```

## Deploy Partner service version 1
```
oc create -f ./partner/kubernetes/partner-service-template.yml -n $OCP_TUTORIAL_PROJECT
```

## Create an OpenShift service entry for the partner service
```
oc create -f ./partner/kubernetes/Service.yml -n $OCP_TUTORIAL_PROJECT
```

## Deploy Gateway Service
```
oc create -f ./gateway/kubernetes/gateway-service-template.yml 
oc create -f ./gateway/kubernetes/Service.yml 
```

## Expose Gateway Service
```
oc apply -f gateway/kubernetes/service-mesh-gw.yaml -n $OCP_TUTORIAL_PROJECT
```

## Deploy the catalog-v2 service
```
oc create -f ./catalog-v2/kubernetes/catalog-service-template.yml -n $OCP_TUTORIAL_PROJECT
oc create -f ./catalog-v2/kubernetes/Service.yml -n $OCP_TUTORIAL_PROJECT
```

## Verify OpenShift round-robin load-balancing
```
export GATEWAY_URL=$(oc -n istio-system get route istio-ingressgateway -o jsonpath='{.spec.host}')
for i in $(1.10) ; do curl $GATEWAY_URL; sleep 1; done
```

## View catalog deployment
```
oc describe service catalog -n $OCP_TUTORIAL_PROJECT | grep Selector
oc get deploy catalog-v1 -o json -n $OCP_TUTORIAL_PROJECT | jq .spec.template.metadata.labels
```

## Route All Traffic to v2
oc apply -f   ./istiofiles/destination-rule-catalog-v1-v2.yml
oc apply -f   ./virtual-service-catalog-v2.yml -n $OCP_TUTORIAL_PROJECT

### Route 90%/10% of the traffic
oc apply -f istiofiles/virtual-service-catalog-v1_and_v2.yml
