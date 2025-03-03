---
title: "Ingest Metrics into AMP"
date: 2021-01-21T22:27:17-05:00
draft: false
weight: 30
---
Amazon Managed Service for Prometheus does not directly scrape operational metrics from containerized workloads in a Kubernetes or ECS cluster. It requires users to deploy and manage a standard Prometheus server, Grafana Cloud Agent, or an OpenTelemetry agent such as the AWS Distro for OpenTelemetry Collector in their cluster to perform this task.

{{%notice info%}}
In this section we will walk through how to deploy a Prometheus server to scrape and ingest metrics into AMP. Take a look at [this blog post](https://aws.amazon.com/blogs/opensource/configuring-grafana-cloud-agent-for-amazon-managed-service-for-prometheus/) for instructions on how to use Grafana Cloud Agent as an alternate option.
{{% /notice%}}


{{% button href="https://www.youtube.com/watch?v=MZ-4HzOC_ac&t=25190s" icon="fab fa-youtube" icon-position="right"  %}} Watch {{% /button %}} our demo at Kubecon showing Prometheus metric collection from EKS to AMP and visualizing the metrics in AMG. _This video plays from a timeline where the relevant content starts._

### Deploy Prometheus server
1. In the AWS Management Console on the Services menu, click Cloud9.
2. Click Open IDE on the observabilityworkshop Cloud9 instance.

#### Install Helm
[Install helm using instructions from the public site](https://helm.sh/docs/intro/install/), or execute the steps below.

3. Execute the following commands in the terminal:

```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```
#### Add new Helm chart repositories

4. Execute the following commands in the terminal:

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add kube-state-metrics https://kubernetes.github.io/kube-state-metrics
helm repo update
```
#### Create a file called amp_ingest_override_values.yaml with the following content in it.

```
serviceAccounts:
  ## Disable alert manager roles
  ##
  server:
        name: "iamproxy-service-account"
  alertmanager:
    create: false

  ## Disable pushgateway
  ##
  pushgateway:
    create: false

server:
  remoteWrite:
    -
      queue_config:
        max_samples_per_send: 1000
        max_shards: 200
        capacity: 2500

  ## Use a statefulset instead of a deployment for resiliency
  ##
  statefulSet:
    enabled: true

  ## Store blocks locally for short time period only
  ##
  retention: 1h
  
## Disable alert manager
##
alertmanager:
  enabled: false

## Disable pushgateway
##
pushgateway:
  enabled: false
```


#### Set up the new server and start ingesting metrics 

5. Execute the following commands in the terminal:

>This script configures and installs Prometheus server on the cluster.

```
IAM_PROXY_PROMETHEUS_ROLE_ARN=$(aws iam get-role --role-name amp-iamproxy-ingest-role | jq .Role.Arn -r)
WORKSPACE_ID=$(aws amp list-workspaces --alias observability-workshop | jq .workspaces[0].workspaceId -r)
helm install amp-prometheus-chart prometheus-community/prometheus -n prometheus -f ./amp_ingest_override_values.yaml \
--set serviceAccounts.server.annotations."eks\.amazonaws\.com/role-arn"="${IAM_PROXY_PROMETHEUS_ROLE_ARN}" \
--set server.remoteWrite[0].url="https://aps-workspaces.${AWS_REGION}.amazonaws.com/workspaces/${WORKSPACE_ID}/api/v1/remote_write" \
--set server.remoteWrite[0].sigv4.region=${AWS_REGION}
```

#### Your setup should now be similar to the architecture diagram below

> The VPC Endpoint shown in the diagram is optional and is not used in the instructions here.

![AMP setup](/images/amp/amp5.png)

#### Validate the setup

6. Execute the following commands in the terminal:
>This script returns a list of Pods in the cluster

```
kubectl get pods -n prometheus
```

> Your results should look similar to the list shown below. You can see that the Prometheus server deployed as a Pod in the EKS cluster.

```
NAME                                                       READY   STATUS    RESTARTS   AGE
adot-collector-hxglz                                     1/1     Running   0          12s
adot-collector-w4f4b                                     1/1     Running   0          12s
amp-prometheus-chart-kube-state-metrics-579888d7-nf546   1/1     Running   0          11m
amp-prometheus-chart-node-exporter-mww2k                 1/1     Running   0          11m
amp-prometheus-chart-node-exporter-zmpk7                 1/1     Running   0          11m
amp-prometheus-chart-server-0                            2/2     Running   0          11m
```

The ADOT collector and the Prometheus server sign the requests when ingesting into the AMP workspace. 

7. Execute the following commands in the terminal:

> This command will allow you to connect to the Prometheus server container from localhost.

```
kubectl port-forward -n prometheus pods/amp-prometheus-chart-server-0 8080:9090
```

8. Navigate to [http://127.0.0.1:8080](http://127.0.0.1:8080) to see the Prometheus interface. 

> If you are on Cloud9, you can open the preview browser by clicking on `Preview Running Application` as shown below.

![Prom dashboard](/images/amp/amp9.png)

9. Paste the following PromQL in the query text box, click `Execute` and switch to the `Graph` tab to see the results as shown below.

```
rate(node_network_receive_bytes_total[5m])
```

![Prom dashboard](/images/amp/amp6.png)

10. Go to the `Configuration` page as shown below and search for the keyword `remote`.  

> You will find that the `remote_write` destination has been set to the AMP workspace as shown below. 

![Configuration Page](/images/amp/amp10.png)


![Prom dashboard](/images/amp/amp7.png)

This concludes this section. You may continue on to the next section.
