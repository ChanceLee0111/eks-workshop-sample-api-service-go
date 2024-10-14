# Example End to end compute autoscaling with HPA


### Intro
This example is to show how Karpenter autoscales nodes by HPA scaling activities

### What we will do
1. Install the Metrics Server
2. Deploy an application with 1 replica to start with
3. Deploy an HPA autoscaling rule with a target CPU average utilization of 70%
4. Simulate sending some real traffic to the UI service to stress the application for 5 mins
5. See autoscaling in action, both for the workload with HPA and the nodes with Karpenter


### 1. Deploy Metrics Server
In order for HPA to collect metrics from your application, you need to deploy the metrics server:
```shell
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm repo update
helm install metrics-server metrics-server/metrics-server \
--set 'tolerations[0].key=CriticalAddonsOnly,tolerations[0].operator=Exists,tolerations[0].effect=NoSchedule' \
--set 'nodeSelector.karpenter\.sh/nodepool=system' \
    -n kube-system
```

Unless you already have nodes provisioned by system NodePool, Karpenter will launch a new node to run the metrics server pod on it. Notice that we’ve added a toleration to launch a node using the system NodePool as this is a critical addon, as well as a nodeSelector.

Wait around one minute for the node to be ready. Then, run this command to confirm the metrics server pod is Running:

```shell
kubectl get pods -n kube-system | grep metrics-server
```

### 2. Deploy an application
To see autoscaling in action, let’s deploy the following application:

```shell
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: retail-store-sample-ui
spec:
  replicas: 1
  selector:
    matchLabels:
      app: retail-store-sample-ui
  template:
    metadata:
      labels:
        app: retail-store-sample-ui
    spec:
      containers:
      - name: retail-store-sample-ui
        image: public.ecr.aws/aws-containers/retail-store-sample-ui:0.8.2
        ports:
        - name: http
          containerPort: 8080
        resources:
          requests:
            cpu: 500m
            memory: 256Mi
          limits:
            memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: retail-store-sample-ui
  labels:
    app: retail-store-sample-ui
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    app: retail-store-sample-ui
EOF
```

Notice that the pods are intentional on how much CPU and memory they request:

Once the application pods are running, run this command to confirm it’s actually working:
```shell
kubectl port-forward svc/retail-store-sample-ui 8080:80
```

Then, open a browser and go to the following URL:
```shell
http://localhost:8080/utility/stress/1000000
```

You should see a Pi number as this endpoint estimates Pi using the Monte Carlo simulation: 3.140612

### 3. Deploy HPA autoscaling rule 
Now that the application is running, and they’re being intention on how much requests they need, it’s time to configure an HPA rule to scale-out when the CPU usage goes above 70%. To do so, run this command:

```shell
cat <<EOF | kubectl apply -f -
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: retail-store-sample-ui-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: retail-store-sample-ui
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
    scaleDown:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
EOF
```
Notice that there’s a behavior section to speed up the scaling-out and scaling-in process. This is so that you don’t spend too much time running this lab, but you should tailor this configuration to your needs. Maybe you need to add/remove pods rapidly/slowly.

Run this command to confirm that the HPA rule is working:
```shell
kubectl get hpa
```
You’ll get an output like this:
```
NAME                         REFERENCE                           TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
retail-store-sample-ui-hpa   Deployment/retail-store-sample-ui   cpu: 0%/70%   1         10        3          67s
```
If you get an unknown instead of 0%, wait around 15 seconds while HPA collects data from the metrics server.

### 4. Load test the UI service 
Once you have everything in place, let’s send some load to the application by running this command:
```shell
kubectl run load-generator \
  --image=williamyeh/hey:latest \
  --restart=Never -- -c 800 -n 1000000 -z 5m http://retail-store-sample-ui.default.svc/utility/stress/1000000
```
A new pod will be created running the load test command. You won’t get any output.

### 5. Load test the UI service 
To see autoscaling in action, let’s open a few terminal tab/windows with the following commands.
In the first terminal, you will be watching all the events happening so you can see what Karpenter is doing:

```shell
kubectl get events -w --sort-by '.lastTimestamp'
```
In the second terminal, you’ll watch how much resources each application pod is consuming:
```shell
watch kubectl top pods -l app=retail-store-sample-ui
```

In the third one, you’ll watch how HPA scales out when the CPU goes above 70% usage:
```shell
watch kubectl get hpa
```

In the fourth one, you’ll watch how the number of replicas changes
```shell
watch kubectl get deployment retail-store-sample-ui
```

Finally, in a fifth one, you’ll watch the Karpenter NodeClaim objects to see how new nodes are coming due to the unscheduled pods that HPA ends up adding:
```shell
watch kubectl get nodeclaim
```

After five minutes have passed, you’ll see that the CPU usage of the application goes to 0%, then HPA will start removing replicas, and Karpenter will start to remove nodes as well.

### Cleanup
To remove all the resources created, run these commands:
```shell
kubectl delete deployment retail-store-sample-ui
kubectl delete service retail-store-sample-ui
helm uninstall metrics-server -n kube-system
kubectl delete hpa retail-store-sample-ui-hpa
kubectl delete pod --all
```