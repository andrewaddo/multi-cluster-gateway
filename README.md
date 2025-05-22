# Multi-cluster gateway

## Use case 1: [**Deploy capacity-based load balancing**](https://cloud.google.com/kubernetes-engine/docs/how-to/deploying-multi-cluster-gateways#capacity-load-balancing)
### Set up and create multiple clusters in different regions, add them to a fleet.
1. Set up environment variables
```
PROJECT_ID=$(gcloud config get project)
PROJECT_NUMBER=$(gcloud projects describe $(gcloud config get-value project) --format="value(projectNumber)")
```
2. Run this command to enable the required APIs if they are not already enabled:
```
gcloud services enable \
  trafficdirector.googleapis.com \
  multiclusterservicediscovery.googleapis.com \
  multiclusteringress.googleapis.com \
  --project=$PROJECT_ID
```
3. Create a GKE cluster in us-central1-a named gke-uscentral1-1. There would be a prompt for enabling the GKE Hub API if it is not already.
```
gcloud container clusters create gke-uscentral1-1 \
    --gateway-api=standard \
    --location=us-central1-a \
    --workload-pool=$PROJECT_ID.svc.id.goog \
    --enable-fleet \
    --project=$PROJECT_ID
```
4. Create another GKE cluster in us-central1-a named gke-uscentral1-2:
```
gcloud container clusters create gke-uscentral1-2 \
    --gateway-api=standard \
    --location=us-central1-c \
    --workload-pool=$PROJECT_ID.svc.id.goog \
    --enable-fleet \
    --project=$PROJECT_ID
```
5. Create another GKE cluster in europe-west1-b named gke-euwest1-1:
```
gcloud container clusters create gke-euwest1-1 \
    --gateway-api=standard \
    --location=europe-west1-b \
    --workload-pool=$PROJECT_ID.svc.id.goog \
    --enable-fleet \
    --project=$PROJECT_ID
```
6. Confirm that the clusters have been successfully registered to the fleet:
```
gcloud container fleet memberships list --project=$PROJECT_ID
```
### Configure cluster credentials
This step configures cluster credentials with memorable names. This makes it easier to switch between clusters when deploying resources across several clusters.

1. Fetch the credentials for cluster gke-uscentral1-1, gke-uscentral1-2, and gke-euwest1-1:
```
gcloud container clusters get-credentials gke-uscentral1-1 --location=us-central1-a --project=$PROJECT_ID
gcloud container clusters get-credentials gke-uscentral1-2 --location=us-central1-c --project=$PROJECT_ID
gcloud container clusters get-credentials gke-euwest1-1 --location=europe-west1-b --project=$PROJECT_ID
```
This stores the credentials locally so that you can use your kubectl client to access the cluster API servers. By default an auto-generated name is created for the credential.

2. Rename the cluster contexts so they are easier to reference later. Replace the $PROJECT_ID with actual values before running the commands (**DO NOT COPY**)
```
kubectl config rename-context gke_$PROJECT_ID_us-central1-a_gke-uscentral1-1 gke-uscentral1-1
kubectl config rename-context gke_$PROJECT_ID_us-central1-c_gke-uscentral1-2 gke-uscentral1-2
kubectl config rename-context gke_$PROJECT_ID_europe-west1-b_gke-euwest1-1 gke-euwest1-1
```
For example,
```
kubectl config rename-context gke_addo-argolis-demo_us-central1-a_gke-uscentral1-1 gke-uscentral1-1
kubectl config rename-context gke_addo-argolis-demo_us-central1-c_gke-uscentral1-2 gke-uscentral1-2
kubectl config rename-context gke_addo-argolis-demo_europe-west1-b_gke-euwest1-1 gke-euwest1-1
```

### Enable multi-cluster Services in the fleet
1. Enable multi-cluster Services in your fleet for the registered clusters. This enables the MCS controller for the three clusters that are registered to your fleet so that it can start listening to and exporting Services.
```
gcloud container fleet multi-cluster-services enable \
    --project $PROJECT_ID
```
2. Grant Identity and Access Management (IAM) permissions required by the MCS controller. Choose option "None" for condition if prompted.
```
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member "serviceAccount:$PROJECT_ID.svc.id.goog[gke-mcs/gke-mcs-importer]" \
    --role "roles/compute.networkViewer" \
    --project=$PROJECT_ID
```
3. Confirm that MCS is enabled for the registered clusters. You will see the memberships for the three registered clusters. It may take several minutes for all of the clusters to show.
```
gcloud container fleet multi-cluster-services describe --project=$PROJECT_ID
```
### Enable multi-cluster Gateway in the fleet
The multi-cluster GKE Gateway controller governs the deployment of multi-cluster Gateways.
When enabling the multi-cluster Gateway controller, you must select your config cluster. The config cluster is the GKE cluster in which your Gateway resources (Gateway, Routes, Policies) are deployed. It is a central place that controls routing across your clusters. 
1. Enable multi-cluster Gateway and specify your config cluster in your fleet. Note that you can always update the config cluster at a later time. This example specifies gke-uscentral1-1 as the config cluster that will host the resources for multi-cluster Gateways.
```
gcloud container fleet ingress enable \
    --config-membership=projects/$PROJECT_ID/locations/us-central1/memberships/gke-uscentral1-1 \
    --project=$PROJECT_ID
```
2. Grant Identity and Access Management (IAM) permissions required by the multi-cluster Gateway controller. Choose "None" for condition if prompted.
```
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member "serviceAccount:service-$PROJECT_NUMBER@gcp-sa-multiclusteringress.iam.gserviceaccount.com" \
    --role "roles/container.admin" \
    --project=$PROJECT_ID
```
3. Confirm that the GKE Gateway controller is enabled for your fleet:
```
gcloud container fleet ingress describe --project=$PROJECT_ID
```
4. Confirm that the GatewayClasses exist in your config cluster:
```
kubectl get gatewayclasses --context=gke-uscentral1-1
```
5. Switch your kubectl context to the config cluster:
```
kubectl config use-context gke-uscentral1-1
```
### Deploy an application
1. Deploy the sample web application server to both clusters:
```
kubectl apply --context gke-uscentral1-2 -f https://raw.githubusercontent.com/GoogleCloudPlatform/gke-networking-recipes/master/gateway/docs/store-traffic-deploy.yaml
kubectl apply --context gke-euwest1-1 -f https://raw.githubusercontent.com/GoogleCloudPlatform/gke-networking-recipes/master/gateway/docs/store-traffic-deploy.yaml
```
### Deploy a Service, Gateway, and HTTPRoute
1. Apply the following Service manifest to both gke-uscentral1-2 and gke-euwest1-1 clusters:
```
cat << EOF | kubectl apply --context gke-uscentral1-2 -f -
apiVersion: v1
kind: Service
metadata:
  name: store
  namespace: traffic-test
  annotations:
    networking.gke.io/max-rate-per-endpoint: "10"
spec:
  ports:
  - port: 8080
    targetPort: 8080
    name: http
  selector:
    app: store
  type: ClusterIP
---
kind: ServiceExport
apiVersion: net.gke.io/v1
metadata:
  name: store
  namespace: traffic-test
EOF
```
```
cat << EOF | kubectl apply --context gke-euwest1-1 -f -
apiVersion: v1
kind: Service
metadata:
  name: store
  namespace: traffic-test
  annotations:
    networking.gke.io/max-rate-per-endpoint: "10"
spec:
  ports:
  - port: 8080
    targetPort: 8080
    name: http
  selector:
    app: store
  type: ClusterIP
---
kind: ServiceExport
apiVersion: net.gke.io/v1
metadata:
  name: store
  namespace: traffic-test
EOF
```
2. Apply the following Gateway manifest to the config cluster, gke-uscentral1-1 in this example:
Create a namespace in gke-uscentral1-1 cluster if needed:
```
kubectl --context gke-uscentral1-1 create namespace traffic-test
```
```
cat << EOF | kubectl apply --context gke-uscentral1-1 -f -
kind: Gateway
apiVersion: gateway.networking.k8s.io/v1beta1
metadata:
  name: store
  namespace: traffic-test
spec:
  gatewayClassName: gke-l7-global-external-managed-mc
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      kinds:
      - kind: HTTPRoute
EOF
```
3. Apply the following HTTPRoute manifest to the config cluster, gke-uscentral1-1 in this example:
```
cat << EOF | kubectl apply --context gke-uscentral1-1 -f -
kind: HTTPRoute
apiVersion: gateway.networking.k8s.io/v1beta1
metadata:
  name: store
  namespace: traffic-test
  labels:
    gateway: store
spec:
  parentRefs:
  - kind: Gateway
    namespace: traffic-test
    name: store
  rules:
  - backendRefs:
    - name: store
      group: net.gke.io
      kind: ServiceImport
      port: 8080
EOF
```
The manifest describes an HTTPRoute that configures the Gateway with a routing rule that directs all traffic to the store ServiceImport. The store ServiceImport groups the store Service Pods across both clusters and allows them to be addressed by the load balancer as a single Service.

```
curl $IP_ADDRESS
```
**Notes:** It might take several minutes (up to 10) for the Gateway to fully deploy and serve traffic. Please wait and try again if you didn't see the expected output (**DO NOT COPY**)
```
{"cluster_name":"gke-uscentral1-2","gce_instance_id":"1053164866380178954","gce_service_account":"addo-argolis-demo.svc.id.goog","host_header":"34.117.147.112","pod_name":"store-7c798f7497-9bjl5","pod_name_emoji":"\ud83e\udd38\ud83c\udffb","project_id":"addo-argolis-demo","timestamp":"2025-05-22T08:43:57","zone":"us-central1-c"}
```
## Verify traffic using load testing
To verify the load balancer is working, you can deploy a traffic generator in your gke-uscentral1-1 cluster
### Configure dashboard
1. Get the name of the underying URLmap for your Gateway:
```
kubectl get gateway store -n traffic-test --context=gke-uscentral1-1 -o=jsonpath="{.metadata.annotations.networking\.gke\.io/url-maps}"
```
The output is similar to the following:
```
/projects/PROJECT_NUMBER/global/urlMaps/gkemcg1-traffic-test-store-armvfyupay1t
```
2. In the Google Cloud console, go to the Metrics explorer page.
Go to Metrics explorer
3. Under Select a metric, click CODE: MQL.
4. Enter the following query to observe traffic metrics for the store Service across your two clusters. Replace $GATEWAY_URL_MAP with the URLmap name from the previous step (i.e. the value after "urlMaps/").
```
rate(loadbalancing_googleapis_com:https_backend_request_count{
  url_map_name="$GATEWAY_URL_MAP"
}[1m])
```
e.g.
```
rate(loadbalancing_googleapis_com:https_backend_request_count{
  url_map_name="gkemcg1-traffic-test-store-armvfyupay1t"
}[1m])
```
5. Click Run query. Wait at least 5 minutes after deploying the load generator in the next section for the metrics to display in the chart.
### Test with 10 RPS / 30 RPS / 60 RPS
#### Test with 10 RPS
1. Deploy a Pod to your gke-uscentral1-1 cluster:
```
kubectl run --context gke-uscentral1-1 -i --tty --rm loadgen  \
    --image=cyrilbkr/httperf  \
    --restart=Never  \
    -- /bin/sh -c 'httperf  \
    --server=$GATEWAY_IP_ADDRESS  \
    --hog --uri="/zone" --port 80  --wsess=100000,1,1 --rate 10'
```
Replace $GATEWAY_IP_ADDRESS with the Gateway IP address from the previous step.

The output is similar to the following, indicating that the traffic generator is sending traffic (DO NOT COPY)
```
If you don't see a command prompt, try pressing enter.
```
The load generator continuously sends 10 RPS to the Gateway. Even though traffic is coming from inside a Google Cloud region, the load balancer treats it as client traffic coming from the US Central. To simulate a realistic client diversity, the load generator sends each HTTP request as a new TCP connection, which means traffic is distributed across backend Pods more evenly.

The generator takes up to 5 minutes to generate traffic for the dashboard.

2. View your Metrics explorer dashboard. Two lines appear, indicating how much traffic is load balanced to each of the clusters:

You should see that us-central1-a is receiving approximately 10 RPS of traffic while europe-west1-b is not receiving any traffic. Because the traffic generator is running in us-central1, all traffic is sent to the Service in the gke-uscentral1-1 cluster.

3. Stop the load generator using Ctrl+C, then delete the pod:
```
kubectl delete pod loadgen --context gke-uscentral1-1
```
#### Test with 30 RPS
1. Deploy the load generator again, but configured to send 30 RPS:
```
kubectl run --context gke-uscentral1-1 -i --tty --rm loadgen  \
    --image=cyrilbkr/httperf  \
    --restart=Never  \
    -- /bin/sh -c 'httperf  \
    --server=$GATEWAY_IP_ADDRESS  \
    --hog --uri="/zone" --port 80  --wsess=100000,1,1 --rate 30'
```
The generator takes up to 5 minutes to generate traffic for the dashboard.

2. View your Cloud Ops dashboard.

You should see that approximately 20 RPS is being sent to us-central1-a and 10 RPS to europe-west1-b. This indicates that the Service in gke-uscentral1-1 is fully utilized and is overflowing 10 RPS of traffic to the Service in gke-euwest1-1.

3. Stop the load generator using Ctrl+C, then delete the Pod:
```
kubectl delete pod loadgen --context gke-uscentral1-1
```
#### Test with 60 RPS
1. Deploy the load generator configured to send 60 RPS:
```
kubectl run --context gke-uscentral1-1 -i --tty --rm loadgen  \
    --image=cyrilbkr/httperf  \
    --restart=Never  \
    -- /bin/sh -c 'httperf  \
    --server=$GATEWAY_IP_ADDRESS  \
    --hog --uri="/zone" --port 80  --wsess=100000,1,1 --rate 60'
```
Wait 5 minutes and view your Cloud Ops dashboard. It should now show that both clusters are receiving roughly 30 RPS. Since all Services are overutilized globally, there is no traffic spillover and Services absorb all the traffic they can.

2. Stop the load generator using Ctrl+C, then delete the Pod:

```
kubectl delete pod loadgen --context gke-uscentral1-1
```
## Clean up
After completing the exercises on this page, follow these steps to remove resources and prevent unwanted charges incurring on your account:
1. Delete the clusters.
2. Unregister the clusters from the fleet if they don't need to be registered for another purpose.
3. Disable the multiclusterservicediscovery feature:
```
gcloud container fleet multi-cluster-services disable
```
4. Disable Multi Cluster Ingress:
```
gcloud container fleet ingress disable
```
5. Disable the APIs:
```
gcloud services disable \
    multiclusterservicediscovery.googleapis.com \
    multiclusteringress.googleapis.com \
    trafficdirector.googleapis.com \
    --project=PROJECT_ID
```