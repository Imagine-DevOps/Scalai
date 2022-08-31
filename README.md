# Scalai

Automate scalable services capacity up or down in Kubernetes to support dynamic traffic. 

-   Validating the installation of Metrics Server
-   Manually scaling an application
-   Autoscaling applications using Horizontal Pod Autoscaler

### Validating the installation of Metrics Server

The _Autoscaling applications using the Horizontal Pod Autoscaler_ recipe in this section also requires Metrics Server to be installed on your cluster. Metrics Server is a cluster-wide aggregator for core resource usage data. Follow these steps to validate the installation of Metrics Server:

1.  Confirm if you need to install Metrics Server by running the following command:

```markup
$ kubectl top nodeerror: metrics not available yet
```

2.  If it's been installed correctly, you should see the following node metrics:

```markup
$ kubectl top nodesNAME                          CPU(cores) CPU% MEMORY(bytes) MEMORY%ip-172-20-32-169.ec2.internal 259m       12%  1492Mi        19%ip-172-20-37-106.ec2.internal 190m       9%   1450Mi        18%ip-172-20-48-49.ec2.internal  262m       13%  2166Mi        27%ip-172-20-58-155.ec2.internal 745m       37%  1130Mi        14%
```

If you get an error message stating metrics not available yet, then you need to follow the steps provided in the next chapter in the _Adding metrics using the Kubernetes Metrics Server_ recipe to install Metrics Server.

### Manually scaling an application

When the usage of your application increases, it becomes necessary to scale the application up. Kubernetes is built to handle the orchestration of high-scale workloads.

Let's perform the following steps to understand how to manually scale an application:

1.  Change directories to /src/chapter7/charts/node, which is where the local clone of the example repository that you created in the _Getting ready_ section can be found:

```markup
$ cd /charts/node/
```

2.  Install the To-Do application example using the following command. This Helm chart will deploy two pods, including a Node.js service and a MongoDB service:

```markup
$ helm install . --name my-ch7-app
```

3.  Get the service IP of my-ch7-app-node to connect to the application. The following command will return an external address for the application:

```markup
$ export SERVICE_IP=$(kubectl get svc --namespace default my-ch7-app-node --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}")$ echo http://$SERVICE_IP/http://mytodoapp.us-east-1.elb.amazonaws.com/
```

4.  Open the address from _Step 3_ in a web browser. You will get a fully functional To-Do application:

![](https://static.packt-cdn.com/products/9781838828042/graphics/assets/1e02a226-7f1d-4804-98f7-6176144d9845.png)

5.  Check the status of the application using helm status. You will see the number of pods that have been deployed as part of the deployment in the Available column:

```markup
$ helm status my-ch7-appLAST DEPLOYED: Thu Oct 3 00:13:10 2019NAMESPACE: defaultSTATUS: DEPLOYEDRESOURCES:==> v1/DeploymentNAME               READY UP-TO-DATE AVAILABLE AGEmy-ch7-app-mongodb 1/1   1          1         9m9smy-ch7-app-node    1/1   1          1         9m9s...
```

6.  Scale the node pod to 3 replicas from the current scale of a single replica:

```markup
$ kubectl scale --replicas 3 deployment/my-ch7-app-nodedeployment.extensions/my-ch7-app-node scaled
```

7.  Check the status of the application again and confirm that, this time, the number of available replicas is 3 and that the number of my-ch7-app-node pods in the v1/Pod section has increased to 3:

```markup
$ helm status my-ch7-app...RESOURCES:==> v1/DeploymentNAME READY UP-TO-DATE AVAILABLE AGEmy-ch7-app-mongodb 1/1 1 1 26mmy-ch7-app-node 3/3 3 3 26m...==> v1/Pod(related)NAME READY STATUS RESTARTS AGEmy-ch7-app-mongodb-5499c954b8-lcw27 1/1 Running 0 26mmy-ch7-app-node-d8b94964f-94dsb 1/1 Running 0 91smy-ch7-app-node-d8b94964f-h9w4l 1/1 Running 3 26mmy-ch7-app-node-d8b94964f-qpm77 1/1 Running 0 91s
```

8.  To scale down your application, repeat _Step 5_, but this time with 2 replicas:

```markup
$ kubectl scale --replicas 2 deployment/my-ch7-app-nodedeployment.extensions/my-ch7-app-node scaled
```

With that, you've learned how to scale your application when needed. Of course, your Kubernetes cluster resources should be able to support growing workload capacities as well. You will use this knowledge to test the service healing functionality in the _Auto-healing pods in Kubernetes_ recipe.

The next recipe will show you how to autoscale workloads based on actual resource consumption instead of manual steps.

### Autoscaling applications using a Horizontal Pod Autoscaler

In this recipe, you will learn how to create a **Horizontal Pod Autoscaler** (**HPA**) to automate the process of scaling the application we created in the previous recipe. We will also test the HPA with a load generator to simulate a scenario of increased traffic hitting our services. Follow these steps:

1.  First, make sure you have the sample To-Do application deployed from the _Manually scaling an application_ recipe. When you run the following command, you should get both MongoDB and Node pods listed:

```markup
$ kubectl get pods | grep my-ch7-appmy-ch7-app-mongodb-5499c954b8-lcw27 1/1 Running 0 4h41mmy-ch7-app-node-d8b94964f-94dsb     1/1 Running 0 4h16mmy-ch7-app-node-d8b94964f-h9w4l     1/1 Running 3 4h41m
```

2.  Create an HPA declaratively using the following command. This will automate the process of scaling the application between 1 to 5 replicas when the targetCPUUtilizationPercentage threshold is reached. In our example, the mean of the pods' CPU utilization target is set to 50 percent usage. When the utilization goes over this threshold, your replicas will be increased:

```markup
cat <<EOF | kubectl apply -f -apiVersion: autoscaling/v1kind: HorizontalPodAutoscalermetadata:  name: my-ch7-app-autoscaler  namespace: defaultspec:  scaleTargetRef:    apiVersion: apps/v1    kind: Deployment    name: my-ch7-app-node  minReplicas: 1  maxReplicas: 5  targetCPUUtilizationPercentage: 50EOF
```

Although the results may be the same most of the time, a declarative configuration requires an understanding of the Kubernetes object configuration specs and file format. As an alternative, kubectl can be used for the imperative management of Kubernetes objects.

Note that you must have a request set in your deployment to use autoscaling. If you do not have a request for CPU in your deployment, the HPA will deploy but will not work correctly.  
You can also create the same HorizontalPodAutoscaler imperatively by running the $ kubectl autoscale deployment my-ch7-app-node --cpu-percent=50 --min=1 --max=5 command.

3.  Confirm the number of current replicas and the status of the HPA. When you run the following command, the number of replicas should be 1:

```markup
$ kubectl get hpaNAME                  REFERENCE                  TARGETS       MINPODS MAXPODS REPLICAS AGEmy-ch7-app-autoscaler Deployment/my-ch7-app-node 0%/50%        1       5       1        40s
```

4.  Get the service IP of my-ch7-app-node so that you can use it in the next step:

```markup
$ export SERVICE_IP=$(kubectl get svc --namespace default my-ch7-app-node --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}")$ echo http://$SERVICE_IP/http://mytodoapp.us-east-1.elb.amazonaws.com/
```

5.  Start a new Terminal window and create a load generator to test the HPA. Make sure that you replace YOUR\_SERVICE\_IP in the following code with the actual service IP from the output of _Step 4_. This command will generate traffic to your To-Do application:

```markup
$ kubectl run -i --tty load-generator --image=busybox /bin/shwhile true; do wget -q -O- YOUR_SERVICE_IP; done
```

6.  Wait a few minutes for the Autoscaler to respond to increasing traffic. While the load generator is running on one Terminal, run the following command on a separate Terminal window to monitor the increased CPU utilization. In our example, this is set to 210%:

```markup
$ kubectl get hpaNAME                  REFERENCE                  TARGETS       MINPODS MAXPODS REPLICAS AGEmy-ch7-app-autoscaler Deployment/my-ch7-app-node 210%/50%      1       5       1        23m
```

7.  Now, check the deployment size and confirm that the deployment has been resized to 5 replicas as a result of the increased workload:

```markup
$ kubectl get deployment my-ch7-app-nodeNAME            READY UP-TO-DATE AVAILABLE AGEmy-ch7-app-node 5/5   5          5         5h23m
```

8.  On the Terminal screen where you run the load generator, press _Ctrl_ + _C_ to terminate the load generator. This will stop the traffic coming to your application.
9.  Wait a few minutes for the Autoscaler to adjust and then verify the HPA status by running the following command. The current CPU utilization should be lower. In our example, it shows that it went down to 0%:

```markup
$ kubectl get hpaNAME                  REFERENCE                  TARGETS MINPODS MAXPODS REPLICAS AGEmy-ch7-app-autoscaler Deployment/my-ch7-app-node 0%/50%  1       5       1        34m
```

10.  Check the deployment size and confirm that the deployment has been scaled down to 1 replica as the result of stopping the traffic generator:

```markup
$ kubectl get deployment my-ch7-app-nodeNAME            READY UP-TO-DATE AVAILABLE AGEmy-ch7-app-node 1/1   1          1         5h35m
```

In this recipe, you learned how to automate how an application is scaled dynamically based on changing metrics. When applications are scaled up, they are dynamically scheduled on existing worker nodes.

## How it works...

This recipe showed you how to manually and automatically scale the number of pods in a deployment dynamically based on the Kubernetes metric.

In this recipe, in _Step 2_, we created an Autoscaler that adjusts the number of replicas between the defined minimum using minReplicas: 1 and maxReplicas: 5. As shown in the following example, the adjustment criteria are triggered by the targetCPUUtilizationPercentage: 50 metric:

```markup
spec:  scaleTargetRef:    apiVersion: apps/v1    kind: Deployment    name: my-ch7-app-node  minReplicas: 1  maxReplicas: 5  targetCPUUtilizationPercentage: 50
```

targetCPUUtilizationPercentage was used with the autoscaling/v1 APIs. You will soon see that targetCPUUtilizationPercentage will be replaced with an array called metrics.

To understand the new metrics and custom metrics, run the following command. This will return the manifest we created with V1 APIs into a new manifest using V2 APIs:

```markup
$ kubectl get hpa.v2beta2.autoscaling my-ch7-app-node -o yaml
```

This enables you to specify additional resource metrics. By default, CPU and memory are the only supported resource metrics. In addition to these resource metrics, v2 APIs enable two other types of metrics, both of which are considered custom metrics: per-pod custom metrics and object metrics. You can read more about this by going to the _Kubernetes HPA documentation_ link mentioned in the _See also_ section.

## See also

-   Kubernetes pod Autoscaler using custom metrics: [https://sysdig.com/blog/kubernetes-autoscaler/](https://sysdig.com/blog/kubernetes-autoscaler/)
-   Kubernetes HPA documentation: [https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
-   Declarative Management of Kubernetes Objects Using Configuration Files: [https://kubernetes.io/docs/tasks/manage-kubernetes-objects/declarative-config/](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/declarative-config/)
-   Imperative Management of Kubernetes Objects Using Configuration Files: [https://kubernetes.io/docs/tasks/manage-kubernetes-objects/imperative-config/](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/imperative-config/)

Bookmark

# Assigning applications to nodes

In this section, we will make sure that pods are not scheduled onto inappropriate nodes. You will learn how to schedule pods into Kubernetes nodes using node selectors, taints, toleration and by setting priorities.

## Getting ready

Make sure you have a Kubernetes cluster ready and kubectl and helm configured to manage the cluster resources.

## How to do it…

This section is further divided into the following subsections to make this process easier:

-   Labeling nodes
-   Assigning pods to nodes using nodeSelector
-   Assigning pods to nodes using node and inter-pod affinity

### Labeling nodes

Kubernetes labels are used for specifying the important attributes of resources that can be used to apply organizational structures onto system objects. In this recipe, we will learn about the common labels that are used for Kubernetes nodes and apply a custom label to be used when scheduling pods into nodes.

Let's perform the following steps to list some of the default labels that have been assigned to your nodes:

1.  List the labels that have been assigned to your nodes. In our example, we will use a kops cluster that's been deployed on AWS EC2, so you will also see the relevant AWS labels, such as availability zones:

```markup
$ kubectl get nodes --show-labelsNAME                          STATUS ROLES AGE VERSION LABELSip-172-20-49-12.ec2.internal  Ready   node  23h v1.14.6 kubernetes.io/arch=amd64,kubernetes.io/instance-type=t3.large,kubernetes.io/os=linux,failure-domain.beta.kubernetes.io/region=us-east-1,failure-domain.beta.kubernetes.io/zone=us-east-1a,kops.k8s.io/instancegroup=nodes,kubernetes.io/hostname=ip-172-20-49-12.ec2.internal,kubernetes.io/role=node,node-role.kubernetes.io/node=...
```

2.  Get the list of the nodes in your cluster. We will use node names to assign labels in the next step:

```markup
$ kubectl get nodesNAME                           STATUS ROLES  AGE VERSIONip-172-20-49-12.ec2.internal   Ready  node   23h v1.14.6ip-172-20-50-171.ec2.internal  Ready  node   23h v1.14.6ip-172-20-58-83.ec2.internal   Ready  node   23h v1.14.6ip-172-20-59-8.ec2.internal    Ready  master 23h v1.14.6
```

3.  Label two nodes as production and development. Run the following command using your worker node names from the output of _Step 2_:

```markup
$ kubectl label nodes ip-172-20-49-12.ec2.internal environment=production$ kubectl label nodes ip-172-20-50-171.ec2.internal environment=production$ kubectl label nodes ip-172-20-58-83.ec2.internal environment=development
```

4.  Verify that the new labels have been assigned to the nodes. This time, you should see environment labels on all the nodes except the node labeled role=master:

```markup
$ kubectl get nodes --show-labels
```

It is recommended to document labels for other people who will use your clusters. While they don't directly imply semantics to the core system, make sure they are still meaningful and relevant to all users.

### Assigning pods to nodes using nodeSelector

In this recipe, we will learn how to schedule a pod onto a selected node using the nodeSelector primitive:

1.  Create a copy of the Helm chart we used in the _Manually scaling an application_ recipe in a new directory called todo-dev. We will edit the templates later in order to specify nodeSelector:

```markup
$ cd src/chapter7/charts$ mkdir todo-dev$ cp -a node/* todo-dev/$ cd todo-dev
```

2.  Edit the deployment.yaml file in the templates directory:

```markup
$ vi templates/deployment.yaml
```

3.  Add nodeSelector: and environment: "{{ .Values.environment }}" right before the containers: parameter. This should look as follows:

```markup
...          mountPath: {{ .Values.persistence.path }}      {{- end }}# Start of the addition      nodeSelector:        environment: "{{ .Values.environment }}"# End of the addition      containers:      - name: {{ template "node.fullname" . }}...
```

The Helm installation uses templates to generate configuration files. As shown in the preceding example, to simplify how you customize the provided values, {{expr}} is used, and these values come from the values.yaml file names. The values.yaml file contains the default values for a chart.

Although it may not be practical on large clusters, instead of using nodeSelector and labels, you can also schedule a pod on one specific node using the nodeName setting. In that case, instead of the nodeSelector setting, you add nodeName: yournodename to your deployment manifest.

4.  Now that we've added the variable, edit the values.yaml file. This is where we will set the environment to the development label:

```markup
$ vi values.yaml
```

5.  Add the environment: development line to the end of the files. It should look as follows:

```markup
...## Affinity for pod assignment## Ref: https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#affinity-and-anti-affinity##affinity: {}environment: development
```

6.  Edit the Chart.yaml file and change the chart name to its folder name. In this recipe, it's called todo-dev. After these changes, the first two lines should look as follows:

```markup
apiVersion: v1name: todo-dev...
```

7.  Update the Helm dependencies and build them. The following commands will pull all the dependencies and build the Helm chart:

```markup
$ helm dep update & helm dep build
```

8.  Examine the chart for issues. If there are any issues with the chart's files, the linting process will bring them up; otherwise, no failures should be found:

```markup
$ helm lint .==> Linting .Lint OK1 chart(s) linted, no failures
```

9.  Install the To-Do application example using the following command. This Helm chart will deploy two pods, including a Node.js service and a MongoDB service, except this time the nodes are labeled as environment: development:

```markup
$ helm install . --name my-app7-dev --set serviceType=LoadBalancer
```

10.  Check that all the pods have been scheduled on the development nodes using the following command. You will find the my-app7-dev-todo-dev pod running on the node labeled environment: development:

```markup
$ for n in $(kubectl get nodes -l environment=development --no-headers | cut -d " " -f1); do kubectl get pods --all-namespaces --no-headers --field-selector spec.nodeName=${n} ; done
```

With that, you've learned how to schedule workload pods onto selected nodes using the nodeSelector primitive.

### Assigning pods to nodes using node and inter-pod Affinity

In this recipe, we will learn how to expand the constraints we expressed in the previous recipe, _Assigning pods to labeled nodes using nodeSelector_, using the affinity and anti-affinity features.

Let's use a scenario-based approach to simplify this recipe for different affinity selector options. We will take the previous example, but this time with complicated requirements:

-   todo-prod must be scheduled on a node with the environment:production label and should fail if it can't.
-   todo-prod should run on a node that is labeled with failure-domain.beta.kubernetes.io/zone=us-east-1a or us-east-1b but can run anywhere if the label requirement is not satisfied.
-   todo-prod must run on the same zone as mongodb, but should not run in the zone where todo-dev is running.

The requirements listed here are only examples in order to represent the use of some affinity definition functionality. This is not the ideal way to configure this specific application. The labels may be completely different in your environment.

The preceding scenario will cover both types of node affinity options (requiredDuringSchedulingIgnoredDuringExecution and preferredDuringSchedulingIgnoredDuringExecution). You will see these options later in our example. Let's get started:

1.  Create a copy of the Helm chart we used in the _Manually scaling an application_ recipe to a new directory called todo-prod. We will edit the templates later in order to specify nodeAffinity rules:

```markup
$ cd src/chapter7/charts$ mkdir todo-prod$ cp -a node/* todo-prod/$ cd todo-prod
```

2.  Edit the values.yaml file. To access it, use the following command:

```markup
$ vi values.yaml
```

3.  Replace the last line, affinity: {}, with the following code. This change will satisfy the first requirement we defined previously, meaning that a pod can only be placed on a node with an environment label and whose value is production:

```markup
## Affinity for pod assignment## Ref: https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#affinity-and-anti-affinity# affinity: {}# Start of the affinity addition #1affinity:  nodeAffinity:    requiredDuringSchedulingIgnoredDuringExecution:      nodeSelectorTerms:      - matchExpressions:        - key: environment          operator: In          values:          - production# End of the affinity addition #1
```

You can also specify more than one matchExpressions under the nodeSelectorTerms. In this case, the pod can only be scheduled onto a node where all matchExpressions are satisfied, which may limit your successful scheduling chances.

Although it may not be practical on large clusters, instead of using nodeSelector and labels, you can also schedule a pod on a specific node using the nodeName setting. In this case, instead of the nodeSelector setting, add nodeName: yournodename to your deployment manifest.

4.  Now, add the following lines right under the preceding code addition. This addition will satisfy the second requirement we defined, meaning that nodes with a label of failure-domain.beta.kubernetes.io/zone and whose value is us-east-1a or us-east-1b will be preferred:

```markup
          - production# End of the affinity addition #1# Start of the affinity addition #2    preferredDuringSchedulingIgnoredDuringExecution:    - weight: 1      preference:        matchExpressions:        - key: failure-domain.beta.kubernetes.io/zone          operator: In          values:          - us-east-1a          - us-east-1b# End of the affinity addition #2
```

5.  For the third requirement, we will use the inter-pod affinity and anti-affinity functionalities. They allow us to limit which nodes our pod is eligible to be scheduled based on the labels on pods that are already running on the node instead of taking labels on nodes for scheduling. The following podAffinity requiredDuringSchedulingIgnoredDuringExecution rule will look for nodes where app: mongodb exist and use failure-domain.beta.kubernetes.io/zone as a topology key to show us where the pod is allowed to be scheduled:

```markup
          - us-east-1b# End of the affinity addition #2# Start of the affinity addition #3a  podAffinity:    requiredDuringSchedulingIgnoredDuringExecution:    - labelSelector:        matchExpressions:        - key: app          operator: In          values:          - mongodb      topologyKey: failure-domain.beta.kubernetes.io/zone# End of the affinity addition #3a
```

6.  Add the following lines to complete the requirements. This time, the podAntiAffinity preferredDuringSchedulingIgnoredDuringExecution rule will look for nodes where app: todo-dev exists and use failure-domain.beta.kubernetes.io/zone as a topology key:

```markup
      topologyKey: failure-domain.beta.kubernetes.io/zone# End of the affinity addition #3a# Start of the affinity addition #3b  podAntiAffinity:    preferredDuringSchedulingIgnoredDuringExecution:    - weight: 100      podAffinityTerm:        labelSelector:          matchExpressions:          - key: app            operator: In            values:            - todo-dev        topologyKey: failure-domain.beta.kubernetes.io/zone# End of the affinity addition #3b
```

7.  Edit the Chart.yaml file and change the chart name to its folder name. In this recipe, it's called todo-prod. After making these changes, the first two lines should look as follows:

```markup
apiVersion: v1name: todo-prod...
```

8.  Update the Helm dependencies and build them. The following commands will pull all the dependencies and build the Helm chart:

```markup
$ helm dep update & helm dep build
```

9.  Examine the chart for issues. If there are any issues with the chart files, the linting process will bring them up; otherwise, no failures should be found:

```markup
$ helm lint .==> Linting .Lint OK1 chart(s) linted, no failures
```

10.  Install the To-Do application example using the following command. This Helm chart will deploy two pods, including a Node.js service and a MongoDB service, this time following the detailed requirements we defined at the beginning of this recipe:

```markup
$ helm install . --name my-app7-prod --set serviceType=LoadBalancer
```

11.  Check that all the pods that have been scheduled on the nodes are labeled as environment: production using the following command. You will find the my-app7-dev-todo-dev pod running on the nodes:

```markup
$ for n in $(kubectl get nodes -l environment=production --no-headers | cut -d " " -f1); do kubectl get pods --all-namespaces --no-headers --field-selector spec.nodeName=${n} ; done
```

In this recipe, you learned about advanced pod scheduling practices while using a number of primitives in Kubernetes, including nodeSelector, node affinity, and inter-pod affinity. Now, you will be able to configure a set of applications that are co-located in the same defined topology or scheduled in different zones so that you have better **service-level agreement** (**SLA**) times.

## How it works...

The recipes in this section showed you how to schedule pods on preferred locations, sometimes based on complex requirements.

In the _Labeling nodes_ recipe, in _Step 1_, you can see that some standard labels have been applied to your nodes already. Here is a short explanation of what they mean and where they are used:

-   kubernetes.io/arch: This comes from the runtime.GOARCH parameter and is applied to nodes to identify where to run different architecture container images, such as x86, arm, arm64, ppc64le, and s390x, in a mixed architecture cluster.
-   kubernetes.io/instance-type: This is only useful if your cluster is deployed on a cloud provider. Instance types tell us a lot about the platform, especially for AI and machine learning workloads where you need to run some pods on instances with GPUs or faster storage options.
-   kubernetes.io/os: This is applied to nodes and comes from runtime.GOOS. It is probably less useful unless you have Linux and Windows nodes in the same cluster.
-   failure-domain.beta.kubernetes.io/region and /zone: This is also more useful if your cluster is deployed on a cloud provider or your infrastructure is spread across a different failure-domain. In a data center, it can be used to define a rack solution so that you can schedule pods on separate racks for higher availability.
-   kops.k8s.io/instancegroup=nodes: This is the node label that's set to the name of the instance group. It is only used with kops clusters.
-   kubernetes.io/hostname: Shows the hostname of the worker.
-   kubernetes.io/role: This shows the role of the worker in the cluster. Some common values include node for representing worker nodes and master, which shows the node is the master node and is tainted as not schedulable for workloads by default.

In the _Assigning pods to nodes using node and inter-pod affinity_ recipe, in _Step 3_, the node affinity rule says that the pod can only be placed on a node with a label whose key is environment and whose value is production.

In _Step 4_, the affinity key: value requirement is preferred (preferredDuringSchedulingIgnoredDuringExecution). The weight field here can be a value between 1 and 100. For every node that meets these requirements, a Kubernetes scheduler computes a sum. The nodes with the highest total score are preferred.

Another detail that's used here is the In parameter. Node Affinity supports the following operators: In, NotIn, Exists, DoesNotExist, Gt, and Lt. You can read more about the operators by looking at the _Scheduler affinities through examples_ link mentioned in the _See also_ section.

If selector and affinity rules are not well planned, they can easily block pods getting scheduled on your nodes. Keep in mind that if you have specified both nodeSelector and nodeAffinity rules, both requirements must be met for the pod to be scheduled on the available nodes.

In _Step 5_, inter-pod affinity is used (podAffinity) to satisfy the requirement in PodSpec. In this recipe, podAffinity is requiredDuringSchedulingIgnoredDuringExecution. Here, matchExpressions says that a pod can only run on nodes where failure-domain.beta.kubernetes.io/zone matches the nodes where other pods with the app: mongodb label are running.

In _Step 6_, the requirement is satisfied with podAntiAffinity using preferredDuringSchedulingIgnoredDuringExecution. Here, matchExpressions says that a pod can't run on nodes where failure-domain.beta.kubernetes.io/zone matches the nodes where other pods with the app: todo-dev label are running. The weight is increased by setting it to 100.

## See also

-   List of known labels, annotations, and taints: [https://kubernetes.io/docs/reference/kubernetes-api/labels-annotations-taints/](https://kubernetes.io/docs/reference/kubernetes-api/labels-annotations-taints/)
-   Assigning Pods to Nodes in the Kubernetes documentation: [https://kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes/](https://kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes/)
-   More on labels and selectors in the Kubernetes documentation: [https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)
-   Scheduler affinities through examples: [https://banzaicloud.com/blog/k8s-affinities/](https://banzaicloud.com/blog/k8s-affinities/)
-   Node affinity and NodeSelector design document: [https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/nodeaffinity.md](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/nodeaffinity.md)
-   Interpod topological affinity and anti-affinity design document: [https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/podaffinity.md](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/podaffinity.md)

Bookmark

# Creating an external load balancer

The load balancer service type is a relatively simple service alternative to ingress that uses a cloud-based external load balancer. The external load balancer service type's support is limited to specific cloud providers but is supported by the most popular cloud providers, including AWS, GCP, Azure, Alibaba Cloud, and OpenStack.

In this section, we will expose our workload ports using a load balancer. We will learn how to create an external GCE/AWS load balancer for clusters on public clouds, as well as for your private cluster using inlet-operator.

## Getting ready

Make sure you have a Kubernetes cluster ready and kubectl and helm configured to manage the cluster resources. In this recipe, we are using a cluster that's been deployed on AWS using kops, as described in [Chapter 1](https://subscription.packtpub.com/book/cloud-and-networking/9781838828042/1), _Building Production-Ready Kubernetes Clusters_, in the _Amazon Web Services_ recipe. The same instructions will work on all major cloud providers.

To access the example files, clone the k8sdevopscookbook/src repository to your workstation to use the configuration files in the src/chapter7/lb directory, as follows:

```markup
$ git clone https://github.com/k8sdevopscookbook/src.git$ cd src/chapter7/lb/
```

After you've cloned the examples repository, you can move on to the recipes.

## How to do it…

This section is further divided into the following subsections to make this process easier:

-   Creating an external cloud load balancer
-   Finding the external address of the service

### Creating an external cloud load balancer

When you create an application and expose it as a Kubernetes service, you usually need the service to be reachable externally via an IP address or URL. In this recipe, you will learn how to create a load balancer, also referred to as a cloud load balancer.

In the previous chapters, we have seen a couple of examples that used the load balancer service type to expose IP addresses, including the _Configuring and managing S3 object storage using MinIO_ and _Application backup and recovery using Kasten_ recipes in the previous chapter, as well as the To-Do application that was provided in this chapter in the _Assigning applications to nodes_ recipe.

Let's use the MinIO application to learn how to create a load balancer. Follow these steps to create a service and expose it using an external load balancer service:

1.  Review the content of the minio.yaml file in the examples directory in src/chapter7/lb and deploy it using the following command. This will create a StatefulSet and a service where the MinIO port is exposed internally to the cluster via port number 9000. You can choose to apply the same steps and create a load balancer for your own application. In that case, skip to _Step 2_:

```markup
$ kubectl apply -f minio.yaml
```

2.  List the available services on Kubernetes. You will see that the MinIO service shows ClusterIP as the service type and none under the EXTERNAL-IP field:

```markup
$ kubectl get svcNAME       TYPE      CLUSTER-IP EXTERNAL-IP PORT(S)  AGEkubernetes ClusterIP 100.64.0.1 <none>      443/TCP  5dminio      ClusterIP None       <none>      9000/TCP 4m
```

3.  Create a new service with the TYPE set to LoadBalancer. The following command will expose port: 9000 of our MinIO application at targetPort: 9000 using the TCP protocol, as shown here:

```markup
cat <<EOF | kubectl apply -f -apiVersion: v1kind: Servicemetadata:  name: minio-servicespec:  type: LoadBalancer  ports:    - port: 9000      targetPort: 9000      protocol: TCP  selector:    app: minioEOF
```

The preceding command will immediately create the Service object, but the actual load balancer on the cloud provider side may take 30 seconds to a minute to be completely initialized. Although the object will state that it's ready, it will not function until the load balancer is initialized. This is one of the disadvantages of cloud load balancers compared to ingress controllers, which we will look at in the next recipe, _Creating an ingress service and service mesh using Istio_.

As an alternative to _Step 3_, you can also create the load balancer by using the following command:

```markup
$ kubectl expose rc example --port=9000 --target-port=9000 --name=minio-service --type=LoadBalancer
```

### Finding the external address of the service

Let's perform the following steps to get the externally reachable address of the service:

1.  List the services that use the LoadBalancer type. The EXTERNAL-IP column will show you the cloud vendor-provided address:

```markup
$ kubectl get svc |grep LoadBalancerNAME          TYPE         CLUSTER-IP    EXTERNAL-IP                                  PORT(S)        AGEminio-service LoadBalancer 100.69.15.120 containerized.me.us-east-1.elb.amazonaws.com 9000:30705/TCP 4h39m
```

2.  If you are running on a cloud provider service such as AWS, you can also use the following command to get the exact address. You can copy and paste this into a web browser:

```markup
$ SERVICE_IP=http://$(kubectl get svc minio-service \-o jsonpath='{.status.loadBalancer.ingress[0].hostname}:{.spec.ports[].targetPort}')$ echo $SERVICE_IP
```

3.  If you are running on a bare-metal server, then you probably won't have a hostname entry. As an example, if you are running MetalLB ([https://metallb.universe.tf/](https://metallb.universe.tf/)), a load balancer for bare-metal Kubernetes clusters, or SeeSaw ([https://github.com/google/seesaw](https://github.com/google/seesaw)), a **Linux Virtual Server** (**LVS**)-based load balancing platform, you need to look for the ip entry instead:

```markup
$ SERVICE_IP=http://$(kubectl get svc minio-service \-o jsonpath='{.status.loadBalancer.ingress[0].ip}:{.spec.ports[].targetPort}')$ echo $SERVICE_IP
```

The preceding command will return a link similar to https://containerized.me.us-east-1.elb.amazonaws.com:9000.

## How it works...

This recipe showed you how to quickly create a cloud load balancer to expose your services with an external address.

In the _Creating a cloud load balancer_ recipe, in _Step 3_, when a load balancer service is created in Kubernetes, a cloud provider load balancer is created on your behalf without you having to go through the cloud service provider APIs separately. This feature helps you easily manage the creation of load balancers outside of your Kubernetes cluster, but at the same takes a bit of time to complete and requires a separate load balancer for every service, so this might be costly and not very flexible.

To give load balancers flexibility and add more application-level functionality, you can use ingress controllers. Using ingress, traffic routing can be controlled by rules defined in the ingress resource. You will learn more about popular ingress gateways in the next two recipes, _Creating an ingress service and service mesh using Istio_ and _Creating an ingress service and service mesh using Linkerd_.

## See also

-   Kubernetes documentation on the load balancer service type: [https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer)
-   Using a load balancer on Amazon EKS: [https://docs.aws.amazon.com/eks/latest/userguide/load-balancing.html](https://docs.aws.amazon.com/eks/latest/userguide/load-balancing.html)
-   Using a load balancer on AKS: [https://docs.microsoft.com/en-us/azure/aks/load-balancer-standard](https://docs.microsoft.com/en-us/azure/aks/load-balancer-standard)
-   Using a load balancer on Alibaba Cloud: [https://www.alibabacloud.com/help/doc-detail/53759.htm](https://www.alibabacloud.com/help/doc-detail/53759.htm)
-   Load balancer for your private Kubernetes cluster: [https://blog.alexellis.io/ingress-for-your-local-kubernetes-cluster/](https://blog.alexellis.io/ingress-for-your-local-kubernetes-cluster/)

Bookmark

# Creating an ingress service and service mesh using Istio

Istio is a popular open source service mesh. In this section, we will get basic Istio service mesh functionality up and running. You will learn how to create a service mesh to secure, connect, and monitor microservices.

Service mesh is a very detailed concept and we don't intend to explain any detailed use cases. Instead, we will focus on getting our service up and running.

## Getting ready

Make sure you have a Kubernetes cluster ready and kubectl and helm configured to manage the cluster resources.

Clone the https://github.com/istio/istio repository to your workstation, as follows:

```markup
$ git clone https://github.com/istio/istio.git $ cd istio
```

We will use the examples in the preceding repository to install Istio on our Kubernetes cluster.

## How to do it…

This section is further divided into the following subsections to make this process easier:

-   Installing Istio using Helm
-   Verifying the installation
-   Creating an ingress gateway

### Installing Istio using Helm

Let's perform the following steps to install Istio:

1.  Create the Istio CRDs that are required before we can deploy Istio:

```markup
$ helm install install/kubernetes/helm/istio-init --name istio-init \--namespace istio-system
```

2.  Install Istio with the default configuration. This will deploy the Istio core components, that is, istio-citadel, istio-galley, istio-ingressgateway, istio-pilot, istio-policy, istio-sidecar-injector, and istio-telemetry:

```markup
$ helm install install/kubernetes/helm/istio --name istio \--namespace istio-system
```

3.  Enable automatic sidecar injection by labeling the namespace where you will run your applications. In this recipe, we will be using the default namespace:

```markup
$ kubectl label namespace default istio-injection=enabled
```

To be able to get Istio functionality for your application, the pods need to run an Istio sidecar proxy. The preceding command will automatically inject the Istio sidecar. As an alternative, you can find the instructions for manually adding Istio sidecars to your pods using the istioctl command in the _Installing the Istio sidecar instructions_ link provided in the _See also_ section.

### Verifying the installation

Let's perform the following steps to confirm that Istio has been installed successfully:

1.  Check the number of Istio CRDs that have been created. The following command should return 23, which is the number of CRDs that have been created by Istio:

```markup
$ kubectl get crds | grep 'istio.io' | wc -l23
```

2.  Run the following command and confirm that the list of Istio core component services have been created:

```markup
$ kubectl get svc -n istio-systemNAME                   TYPE         CLUSTER-IP     EXTERNAL-IP PORT(S)             AGEistio-citadel          ClusterIP    100.66.235.211 <none>      8060/TCP,...        2m10sistio-galley           ClusterIP    100.69.206.64  <none>      443/TCP,...         2m11sistio-ingressgateway   LoadBalancer 100.67.29.143  domain.com  15020:31452/TCP,... 2m11sistio-pilot            ClusterIP    100.70.130.148 <none>      15010/TCP,...       2m11sistio-policy           ClusterIP    100.64.243.176 <none>      9091/TCP,...        2m11sistio-sidecar-injector ClusterIP    100.69.244.156 <none>      443/TCP,...         2m10sistio-telemetry        ClusterIP    100.68.146.30  <none>      9091/TCP,...        2m11sprometheus             ClusterIP    100.71.172.191 <none>      9090/TCP            2m11s
```

3.  Make sure that all the pods listed are in the Running state:

```markup
$ kubectl get pods -n istio-system
```

4.  Confirm the Istio injection enabled namespaces. You should only see istio-injection for the default namespace:

```markup
$ kubectl get namespace -L istio-injectionNAME            STATUS AGE  ISTIO-INJECTIONdefault         Active 5d8h enabledistio-system    Active 40mkube-node-lease Active 5d8hkube-public     Active 5d8hkube-system     Active 5d8h
```

You can always enable injection for the other namespaces by adding the istio-injection=enabled label to a namespace.

### Creating an ingress gateway

Instead of using a controller to load balance traffic, Istio uses a gateway. Let's perform the following steps to create an Istio ingress gateway for our example application:

1.  Review the content of the minio.yaml file in the examples directory in src/chapter7/lb and deploy it using the following command. This will create a StatefulSet and a service where the MinIO port is exposed internally to the cluster via port number 9000. You can also choose to apply the same steps and create an ingress gateway for your own application. In that case, skip to _Step 2_:

```markup
$ kubectl apply -f minio.yaml
```

2.  Get the ingress IP and ports:

```markup
$ export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')$ export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')$ export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}')
```

3.  Create a new Istio gateway:

```markup
$ cat <<EOF | kubectl apply -f -apiVersion: networking.istio.io/v1alpha3kind: Gatewaymetadata:  name: minio-gatewayspec:  selector:    istio: ingressgateway   servers:  - port:      number: 80      name: http      protocol: HTTP    hosts:    - "*"EOF
```

4.  Create a new VirtualService to forward requests to the MinIO instance via the gateway. This helps specify routing for the gateway and binds the gateway to the VirtualService:

```markup
$ cat <<EOF | kubectl apply -f -apiVersion: networking.istio.io/v1alpha3kind: VirtualServicemetadata: name: miniospec: hosts: - "*" gateways: - minio-gateway.default http: - match: - uri: prefix: / route: - destination: port: number: 9000 host: minioEOF
```

This configuration will expose your services to external access using Istio, and you will have more control over rules.

## How it works...

This recipe showed you how to quickly configure the Istio service mesh and use custom Istio resources such as ingress gateway to open a service to external access.

For the service mesh to function correctly, each pod in the mesh needs to run an Envoy sidecar. In the _Installing Istio using Helm_ recipe, in _Step 3_, we enabled automatic injection for pods in the default namespace so that the pods that are deployed in that namespace will run the Envoy sidecar.

An ingress controller is a reverse-proxy that runs in the Kubernetes cluster and configures routing rules. In the _Creating an ingress gateway_ recipe, in _Step 2_, unlike traditional Kubernetes ingress objects, we used Istio CRDs such as Gateway, VirtualService, and DestinationRule to create the ingress.

We created a gateway rule for the ingress Gateway using the istio: ingressgateway selector in order to accept HTTP traffic on port number 80.

In _Step 4_, we created a VirtualService for the MinIO services we wanted to expose. Since the gateway may be in a different namespace, we used minio-gateway.default to set the gateway name.

With this, we have exposed our service using HTTP. You can read more about exposing the service using the HTTPS protocol by looking at the link in _See also_ section.

## There's more…

Although it is very popular, Istio is not the simplest ingress to deal with. We highly recommend that you look at all the options that are available for your use case and consider alternatives. Therefore, it is useful to know how to remove Istio.

### Deleting Istio

You can delete Istio by using the following commands:

```markup
$ helm delete istio$ helm delete istio-init
```

If you want to completely remove the deleted release records from the Helm records and free the release name to be used later, add the \--purge parameter to the preceding commands.

## See also

-   Istio documentation: [https://istio.io/docs/](https://istio.io/docs/)
-   Istio examples: [https://istio.io/docs/examples/bookinfo/](https://istio.io/docs/examples/bookinfo/)
-   Installing the Istio sidecar: [https://istio.io/docs/setup/additional-setup/sidecar-injection/](https://istio.io/docs/setup/additional-setup/sidecar-injection/)
-   Istio ingress tutorial from Kelsey Hightower: [https://github.com/kelseyhightower/istio-ingress-tutorial  
    ](https://github.com/kelseyhightower/istio-ingress-tutorial)
-   Traffic management with Istio: [https://istio.io/docs/tasks/traffic-management/](https://istio.io/docs/tasks/traffic-management/)
-   Security with Istio: [https://istio.io/docs/tasks/security/](https://istio.io/docs/tasks/security/)[](https://istio.io/docs/tasks/security/)
-   Policy enforcement with Istio: [https://istio.io/docs/tasks/policy-enforcement/](https://istio.io/docs/tasks/policy-enforcement/)
-   Collecting telemetry information with Istio: [https://istio.io/docs/tasks/telemetry/](https://istio.io/docs/tasks/telemetry/)
-   Creating Kubernetes ingress with Cert-Manager: [https://istio.io/docs/tasks/traffic-management/ingress/ingress-certmgr/](https://istio.io/docs/tasks/traffic-management/ingress/ingress-certmgr/)

Bookmark

# Creating an ingress service and service mesh using Linkerd

In this section, we will get basic Linkerd service mesh up and running. You will learn how to create a service mesh to secure, connect, and monitor microservices.

Service mesh is a very detailed concept in itself and we don't intend to explain any detailed use cases here. Instead, we will focus on getting our service up and running.

## Getting ready

Make sure you have a Kubernetes cluster ready and kubectl and helm configured to manage the cluster resources.

To access the example files for this recipe, clone the k8sdevopscookbook/src repository to your workstation to use the configuration files in the src/chapter7/linkerd directory, as follows:

```markup
$ git clone https://github.com/k8sdevopscookbook/src.git$ cd src/chapter7/linkerd/
```

After you've cloned the preceding repository, you can get started with the recipes.

## How to do it…

This section is further divided into the following subsections to make this process easier:

-   Installing the Linkerd CLI
-   Installing Linkerd
-   Verifying a Linkerd deployment
-   Viewing the Linkerd metrics

### Installing the Linkerd CLI

To interact with Linkerd, you need to install the linkerd CLI. Follow these steps:

1.  Install the linkerd CLI by running the following command:

```markup
$ curl -sL https://run.linkerd.io/install | sh
```

2.  Add the linkerd CLI to your path:

```markup
$ export PATH=$PATH:$HOME/.linkerd2/bin
```

3.  Verify that the linkerd CLI has been installed by running the following command. It should show the server as unavailable since we haven't installed it yet:

```markup
$ linkerd versionClient version: stable-2.5.0Server version: unavailable
```

4.  Validate that linkerd can be installed. This command will check the cluster and point to issues if they exist:

```markup
$ linkerd check --preStatus check results are √
```

If the status checks are looking good, you can move on to the next recipe.

### Installing Linkerd

Compared to the alternatives, Linkerd is much easier to get started with and manage, so it is my preferred service mesh.

Install the Linkerd control plane using the Linkerd CLI. This command will use the default options and install the linkerd components in the linkerd namespace:

```markup
$ linkerd install | kubectl apply -f -
```

Pulling all the container images may take a minute or so. After that, you can verify the health of the components by following the next recipe, _Verifying a Linkerd deployment._

### Verifying a Linkerd deployment

Verifying Linkerd's deployment is as easy as the installation process.

Run the following command to validate the installation. This will display a long summary of control plane components and APIs and will make sure you are running the latest version:

```markup
$ linkerd check...control-plane-version---------------------√ control plane is up-to-date√ control plane and cli versions matchStatus check results are √
```

If the status checks are good, you are ready to test Linkerd with a sample application.

### Adding Linkerd to a service

Follow these steps to add Linkerd to our demo application:

1.  Change directories to the linkerd folder:

```markup
$ cd /src/chapter7/linkerd
```

2.  Deploy the demo application, which uses a mix of gRPC and HTTP calls to service a voting application to the user:

```markup
$ kubectl apply -f emojivoto.yml
```

3.  Get the service IP of the demo application. This following command will return the externally accessible address of your application:

```markup
$ SERVICE_IP=http://$(kubectl get svc web-svc -n emojivoto \-o jsonpath='{.status.loadBalancer.ingress[0].hostname}:{.spec.ports[].targetPort}')$ echo $SERVICE_IP
```

4.  Open the external address from _Step 3_ in a web browser and confirm that the application is functional:

![](https://static.packt-cdn.com/products/9781838828042/graphics/assets/d32ad784-1d4e-49cb-a1d3-90a8e88004d6.png)

5.  Enable automatic sidecar injection by labeling the namespace where you will run your applications. In this recipe, we're using the emojivoto namespace:

```markup
$ kubectl label namespace emojivoto linkerd.io/inject=enabled
```

You can also manually inject a linkerd sidecar by patching the pods where you run your applications using the  
kubectl get -n emojivoto deploy -o yaml | linkerd inject - | kubectl apply -f - command. In this recipe, the emojivoto namespace is used.

## There's more…

This section is further divided into the following subsections to make this process easier:

-   Accessing the dashboard
-   Deleting Linkerd

### Accessing the dashboard

We can either use port forwarding or use ingress to access the dashboard. Let's start with the simple way of doing things, that is, by port forwarding to your local system:

1.  View the Linkerd dashboard by running the following command:

```markup
$ linkerd dashboard &
```

2.  Visit the following link in your browser to view the dashboard:

```markup
http://127.0.0.1:50750
```

The preceding commands will set up a port forward from your local system to the linkerd-web pod.

If you want to access the dashboard from an external IP, then follow these steps:

1.  Download the sample ingress definition:

```markup
$ wget https://raw.githubusercontent.com/k8sdevopscookbook/src/master/chapter7/linkerd/ingress-nginx.yaml
```

2.  Edit the ingress configuration in the ingress-nginx.yaml file in the src/chapter7/linkerd directory and change \- host: dashboard.example.com on line 27 to the URL where you want your dashboard to be exposed. Apply the configuration using the following command:

```markup
$ kubectl apply -f ingress-nginx.yaml
```

The preceding example file uses linkerddashboard.containerized.me as the dashboard address. It also protects access with basic auth using admin/admin credentials. It is highly suggested that you use your own credentials by changing the base64-encoded key pair defined in the auth section of the configuration using the username:password format.

### Deleting Linkerd

To remove the Linkerd control plane, run the following command:

```markup
$ linkerd install --ignore-cluster | kubectl delete -f -
```

This command will pull a list of all the configuration files for the Linkerd control plane, including namespaces, service accounts, and CRDs, and remove them.

## See also

-   Linkerd documentation: [https://linkerd.io/2/overview/](https://linkerd.io/2/overview/)
-   Common tasks with Linkerd: [https://linkerd.io/2/tasks/](https://linkerd.io/2/tasks/)
-   Frequently asked Linkerd questions and answers: [https://linkerd.io/2/faq/](https://linkerd.io/2/faq/)

Bookmark

# Auto-healing pods in Kubernetes

Kubernetes has self-healing capabilities at the cluster level. It restarts containers that fail, reschedules pods when nodes die, and even kills containers that don't respond to your user-defined health checks.

In this section, we will perform application and cluster scaling tasks. You will learn how to use liveness and readiness probes to monitor container health and trigger a restart action in case of failures.

## Getting ready

Make sure you have a Kubernetes cluster ready and kubectl and helm configured to manage the cluster resources.

## How to do it…

This section is further divided into the following subsections to make this process easier:

-   Testing self-healing pods
-   Adding liveness probes to pods

### Testing self-healing pods

In this recipe, we will manually remove pods in our deployment to show how Kubernetes replaces them. Later, we will learn how to automate this using a user-defined health check. Now, let's test Kubernetes' self-healing for destroyed pods:

1.  Create a deployment or StatefulSet with two or more replicas. As an example, we will use the MinIO application we used in the previous chapter, in the _Configuring and managing S3 object storage using MinIO_ recipe. This example has four replicas:

```markup
$ cd src/chapter7/autoheal/minio$ kubectl apply -f minio.yaml
```

2.  List the MinIO pods that were deployed as part of the StatefulSet. You will see four pods:

```markup
$ kubectl get pods |grep miniominio-0 1/1 Running 0 4m38msminio-1 1/1 Running 0 4m25sminio-2 1/1 Running 0 4m12sminio-3 1/1 Running 0 3m48s
```

3.  Delete a pod to test Kubernetes' auto-healing functionality and immediately list the pods again. You will see that the terminated pod will be quickly rescheduled and deployed:

```markup
$ kubectl delete pod minio-0pod "minio-0" deleted$ kubectl get pods |grep miniominio-0minio-0 0/1 ContainerCreating 0 2sminio-1 1/1 Running           0 8m9sminio-2 1/1 Running           0 7m56sminio-3 1/1 Running           0 7m32s
```

With this, you have tested Kubernetes' self-healing after manually destroying a pod in operation. Now, we will learn how to add a health status check to pods to let Kubernetes automatically kill non-responsive pods so that they're restarted.

### Adding liveness probes to pods

Kubernetes uses liveness probes to find out when to restart a container. Liveness can be checked by running a liveness probe command inside the container and validating that it returns 0 through TCP socket liveness probes or by sending an HTTP request to a specified path. In that case, if the path returns a success code, then kubelet will consider the container to be healthy. In this recipe, we will learn how to send an HTTP request method to the example application. Let's perform the following steps to add liveness probes:

1.  Edit the minio.yaml file in the src/chapter7/autoheal/minio directory and add the following livenessProbe section right under the volumeMounts section, before volumeClaimTemplates. Your YAML manifest should look similar to the following. This will send an HTTP request to the /minio/health/live location every 20 seconds to validate its health:

```markup
...        volumeMounts:        - name: data          mountPath: /data#### Starts here         livenessProbe:          httpGet:            path: /minio/health/live            port: 9000          initialDelaySeconds: 120          periodSeconds: 20#### Ends here   # These are converted to volume claims by the controller  # and mounted at the paths mentioned above.  volumeClaimTemplates:
```

For liveness probes that use HTTP requests to work, an application needs to expose unauthenticated health check endpoints. In our example, MinIO provides this through the /minio/health/live endpoint. If your workload doesn't have a similar endpoint, you may want to use liveness commands inside your pods to verify their health.

2.  Deploy the application. It will create four pods:

```markup
$ kubectl apply -f minio.yaml
```

3.  Confirm the liveness probe by describing one of the pods. You will see a Liveness description similar to the following:

```markup
$ kubectl describe pod minio-0...    Liveness: http-get http://:9000/minio/health/live delay=120s timeout=1s period=20s #success=1 #failure=3...
```

4.  To test the liveness probe, we need to edit the minio.yaml file again. This time, set the livenessProbe port to 8000, which is where the application will not able to respond to the HTTP request. Repeat _Steps 2_ and _3_, redeploy the application, and check the events in the pod description. You will see a minio failed liveness probe, will be restarted message in the events:

```markup
$ kubectl describe pod minio-0
```

5.  You can confirm the restarts by listing the pods. You will see that every MinIO pod is restarted multiple times due to it having a failing liveness status:

```markup
$ kubectl get podsNAME    READY STATUS  RESTARTS AGEminio-0 1/1   Running 4        12mminio-1 1/1   Running 4        12mminio-2 1/1   Running 3        11mminio-3 1/1   Running 3        11m
```

In this recipe, you learned how to implement the auto-healing functionality for applications that are running in Kubernetes clusters.

## How it works...

This recipe showed you how to use a liveness probe on your applications running on Kubernetes.

In the _Adding liveness probes to pods_ recipe, in _Step 1_, we added an HTTP request-based health check.

By adding the StatefulSet path and port, we let kubelet probe the defined endpoints. Here, the initialDelaySeconds field tells kubelet that it should wait 120 seconds before the first probe. If your application takes a while to get the endpoints ready, then make sure that you allow enough time before the first probe; otherwise, your pods will be restarted before the endpoints can respond to requests.

In _Step 3_, the periodSeconds field specifies that kubelet should perform a liveness probe every 20 seconds. Again, depending on the applications' expected availability, you should set a period that is right for your application.

## See also

-   Configuring liveness and readiness probes: [https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
-   Kubernetes Best Practices: Setting up health checks: [https://cloud.google.com/blog/products/gcp/kubernetes-best-practices-setting-up-health-checks-with-readiness-and-liveness-probes](https://cloud.google.com/blog/products/gcp/kubernetes-best-practices-setting-up-health-checks-with-readiness-and-liveness-probes)

Bookmark

# Managing upgrades through blue/green deployments

The blue-green deployment architecture is a method that's used to reduce downtime by running two identical production environments that can be switched between when needed. These two environments are identified as blue and green. In this section, we will perform rollover application upgrades. You will learn how to roll over a new version of your application with persistent storage by using blue/green deployment in Kubernetes.

## Getting ready

Make sure you have a Kubernetes cluster ready and kubectl and helm configured to manage the cluster resources.

For this recipe, we will need a persistent storage provider to take snapshots from one version of the application and use clones with the other version of the application to keep the persistent volume content. We will use OpenEBS as a persistent storage provider, but you can also use any CSI-compatible storage provider.

Make sure OpenEBS has been configured with the cStor storage engine by following the instructions in [Chapter 5](https://subscription.packtpub.com/book/cloud-and-networking/9781838828042/5), _Preparing for Stateful Workload_s, in the _Persistent storage using OpenEBS_ recipe.

## How to do it…

This section is further divided into the following subsections to make this process easier:

-   Creating the blue deployment
-   Creating the green deployment
-   Switching traffic from blue to green

### Creating the blue deployment

There are many traditional workloads that won't work with Kubernetes' way of rolling updates. If your workload needs to deploy a new version and cut over to it immediately, then you may need to perform blue/green deployment instead. Using the blue/green deployment approach, we will label the current production blue. In the next recipe, we will create an identical production environment called green before redirecting the services to green.

Let's perform the following steps to create the first application, which we will call blue:

1.  Change directory to where the examples for this recipe are located:

```markup
$ cd /src/chapter7/bluegreen
```

2.  Review the content of the blue-percona.yaml file and use that to create the blue version of your application:

```markup
$ kubectl create -f blue-percona.yamlpod "blue" createdpersistentvolumeclaim "demo-vol1-claim" created
```

3.  Review the content of the percona-svc.yaml file and use that to create the service. You will see that selector in the service is set to app: blue. This service will forward all the MySQL traffic to the blue pod:

```markup
$ kubectl create -f percona-svc.yaml
```

4.  Get the service IP for percona. In our example, the Cluster IP is 10.3.0.75:

```markup
$ kubectl get svc perconaNAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGEpercona ClusterIP 10.3.0.75 <none> 3306/TCP 1m
```

5.  Edit the sql-loadgen.yaml file and replace the target IP address with your percona service IP. In our example, it is 10.3.0.75:

```markup
      containers:      - name: sql-loadgen        image: openebs/tests-mysql-client        command: ["/bin/bash"]        args: ["-c", "timelimit -t 300 sh MySQLLoadGenerate.sh 10.3.0.75 > /dev/null 2>&1; exit 0"]        tty: true
```

6.  Start the load generator by running the sql-loadgen.yaml job:

```markup
$ kubectl create -f sql-loadgen.yaml
```

This job will generate a MySQL load targeting the IP of the service that was forwarded to the Percona workload (currently blue).

### Creating the green deployment

Let's perform the following steps to deploy the new version of the application as our green deployment. We will switch the service to green, take a snapshot of blue's persistent volume, and deploy the green workload in a new pod:

1.  Let's create a snapshot of the data from the blue application's PVC and use it to deploy the green application:

```markup
$ kubectl create -f snapshot.yamlvolumesnapshot.volumesnapshot.external-storage.k8s.io "snapshot-blue" created
```

2.  Review the content of the green-percona.yaml file and use that to create the green version of your application:

```markup
$ kubectl create -f green-percona.yamlpod "green" createdpersistentvolumeclaim "demo-snap-vol-claim" created
```

This pod will use a snapshot of the PVC from the blue application as its original PVC.

### Switching traffic from blue to green

Let's perform the following steps to switch traffic from blue to the new green deployment:

Edit the service using the following command and replace blue with green. Service traffic will be forwarded to the pod that is labeled green:

```markup
$ kubectl edit svc percona
```

In this recipe, you have learned how to upgrade your application with a stateful workload using the blue/green deployment strategy.

## See also

-   Zero Downtime Deployments in Kubernetes with Jenkins: [https://kubernetes.io/blog/2018/04/30/zero-downtime-deployment-kubernetes-jenkins/](https://kubernetes.io/blog/2018/04/30/zero-downtime-deployment-kubernetes-jenkins/)
-   A Simple Guide to blue/green Deployment: [https://codefresh.io/kubernetes-tutorial/blue-green-deploy/](https://codefresh.io/kubernetes-tutorial/blue-green-deploy/)
-   Kubernetes blue-green Deployment Examples: [https://github.com/ContainerSolutions/k8s-deployment-strategies/tree/master/blue-green](https://github.com/ContainerSolutions/k8s-deployment-strategies/tree/master/blue-green)
