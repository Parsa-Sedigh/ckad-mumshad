## 78 - Readiness and Liveness Probes
### Readiness probes
A few different stages in the lifecycle of a pod:

A pod has a status and some conditions. The pod status tells us where the pod is, in it's lifecycle. When a pod is first created,
it is in a `Pending` state. This is when the scheduler tries to figure out where to place the pod. If the scheduler cannot find
the node to place the pod, it remains in a pending state. To find out why it's stuck in a pending state, run: `k describe pod` command.

Once the pod is scheduled, it goes into a `ContainerCreating` status where the images required for the application are pulled and the
container starts.

Once all the containers in a pod starts, it goes into a `Running` state where it continues to be, until the program completes successfully
or it's terminated.

Conditions complement pod status. It is an array of true or false values that tell us the state of a pod and consist of:
- PodScheduled: when a pod is scheduled on a pod, this is set to true
- Initialized
- ContainerReady: We know a pod can have multiple containers. When all the containers in the pod are ready, this is set to true and finally
the pod itself is considered to be ready and this is reflected in the `Ready` condition
- Ready: Indicates that the application inside the pod is running and is ready to accept user traffic.

To see the state of pod conditions, run: `k describe pod` and look for the `Conditions` section. The Ready condition is also in the output of `k get pods`.

About the Ready condition:

What does that really mean?
The containers could be running different kinds of applications in them. It could be a simple script that performs a job. It could be a DB
or a large web server. The script maybe take a few milliseconds to get ready. The DB may take a few seconds to power up.
Some web servers may take several minutes to warm up. For example to run an instance of a jenkins server, it takes about 10 to 15 seconds for the server
to initialize, before a user can access the web UI. Even after the web UI is initialized, it takes a few seconds for the server to warm up and be ready
to serve users. During this wait period, if you look at the state of the pod, it continues to indicate that the pod is ready! which is not every true.
So why is that happening and how does k8s know whether the app inside the container is actually running or not? But before getting into that,
why does it matter if the state is reported incorrectly?

Remember that to expose the pod to users, we use a service. The service will route traffic to the pod immediately. The service relies on the pod's
Ready condition to route traffic. By default, k8s assumes that as soon as the container is created, it is ready to serve user traffic So it sets the value of
the Ready(ContainerReady?) condition for each container to true. But if the app within the container took longer to get ready, the service is unaware of it
and sends traffic through as the container is already in a Ready state, causing users to hit a pod that isn't running a live application.
What we need here is a way to tie the `Ready` condition to the actual state of the application inside the container.

As a developer of the app, you know better what it means for the app to be ready.

### Readiness probes
There are different ways that you can define if an app inside a container is actually ready. You can set up different kinds of tests or probes(which is
the appropriate term). In the case of a web app, it could be when the API server is up and running. So you can run an HTTP test to see if the API server
responds. In case of a DB, you may test to see if a particular TCP socket is listening. Or you may execute a command within the container to run a 
custom script that would exit successfully if the app is ready.

How do you configure that test? In the pod-definition file, add readinessProbe and use httpGet. Now when the container is created,
k8s does not immediately set the Ready condition on the container to true. Instead, it performs a test to see if the API responds positively. Until then,
the service does not forward any traffic to the pod, as it sees that the pod is not ready.

There are different ways a probe can be configured:
- http test
- tcp test
- exec command

If you know that your app will take a minimum of say 10 seconds to warmup, you can add an additional delay to the probe using `initialDelaySeconds` option.

**Note:** By default, if the app is not ready after 3 attempts, the probe will stop. If you like to make more attempts, use `failureThreshold`.

Q: How readiness probes are useful in a multi-pod setup?

A: Let's say we have a replica-set or deployment with multiple pods and a service serving traffic to all the pods. There are two pods already serving users.
Now we add another pod and let's say the pod takes a minute to warm up. Without the readiness probe configured correctly, the service would immediately
start routing traffic to the new pod. That will result in service disruption to at least some of the users. Instead, if the pods were configured with the correct
readiness probe, the service will continue to route traffic only to the older pods and wait until the new pod is ready. Once ready,
the traffic will be routed to the new pod as well, ensuring no users are affected.

## 79 - Liveness Probes
### Docker
When we run a container with docker using `docker run`, if the container dies, since docker is not an orchestration engine, the container 
continues to stay dead and deny services to users until you manually create a new container.

### Kubernetes
In k8s, when we run the same app, every time the app crashes, k8s makes an attempt to restart the container to restore service to users.
You can see the count of restarts increase in the output of `k get pods`. This works fine. However, what if the app is not really working, but the container
continues to stay alive? Say for example due to a bug in the code, the app is stuck in an infinite loop. As far as k8s is concerned, the container
is up, so the app is assumed to be up. But the users hitting the container, are not served.

In this case, the container needs to be restarted or destroyed and a new container is to be brought up. That is where the liveness probe can help us.
A liveness probe can be configured on the container to periodically test whether the application within the container is actually **healthy**. If the test
fails, the container is considered unhealthy and is destroyed and recreated.

As a developer, you get to define what it means for an application to be healthy.

### Liveness probes
- In case of a web application, it could be when the API server is up and running.
- in case of DB, you may test to see if a particular TCP socket is listening
- or you may execute a command to perform a test

The liveness probe is defined in the pod-definition file as you did with the readiness probe.

## 80 - Practice Test Readiness and Liveness Probes
Link to Practice Test: https://uklabs.kodekloud.com/topic/readiness-probes-2/

## 81 - Solution Readiness and Liveness Probes
## 82 - Container Logging
### Logs - docker
Logging in docker.

If you run `docker run` you can see the logs. But if you run the docker container in the background in a detached mode using `docker run -d`,
we wouldn't see the logs. If we want to view the logs, we can use `docker logs -f <container id>`. The -f option helps us see the live log trail.

### Logs - kubernetes
Once the pod is running, we can view the logs with `kubectl logs -f <pod name>`. Use -f to stream the logs live just like the previous docker command.

Now these logs are specific to the container running inside the pod. But k8s pods can have multiple docker containers in them.
If you run the `k logs -f <pod name>` command now(pod has multiple containers in it), which container's logs would it show?
You must specify the name of the container explicitly in the command, otherwise it would fail asking you to specify a name. So instead run:
`k logs -f <pod name> <container name>`.

This is the simple logging functionality implemented within k8s.

83 - Practice Test Container Logging
84 - Solution Logging optional
## 85 - Monitor and Debug Applications
Monitoring a k8s cluster.

How do you monitor resource consumption on k8s? Or more importantly, what would you like to monitor?

I'd like to know **node-level** metrics, such as the number of nodes in the cluster, how many of them are healthy, as well as performance metrics,
such as CPU, memory, network and disk utilization. As well as **pod-level** metrics such as the number of pods and the performance metrics of each
pod such as CPU and memory consumption on them. So we need a solution that will monitor these metrics, store them and provide analytics around this data.

As of this video, k8s does not come with a full-featured builtin monitoring solution. However, there are some OS solutions available today(oct 2018),
such as metrics server, prometheus, the elastic stack and proprietary solutions like datadog and dynatrace.

In this course, we discuss about metrics-server only.

Heapster was one of the original projects that enabled monitoring and analysis tools for k8s. But it's now deprecated and a slim down version was formed,
known as the metrics server. You can have one metrics server per k8s cluster.

The metrics server retrieves metrics from each of the k8s nodes and pods, aggregates them and stores them in **memory**. Note that the metrics server
is only an in-memory monitoring solution and does not store the metrics on the disk and as a result, you can't see historical performance data.
For that, you must rely on one of the advanced monitoring solutions.

Q: How are the metrics generated for the pods on these nodes?

A: K8s runs an agent on each node, known as the kubelet which is responsible for receiving instructions from the k8s API master server and running pods
on the nodes. The kubelet also contains a sub-component known as the cAdvisor or container advisor. cAdvisor is responsible for retrieving performance
metrics from pods and exposing them through the kubelet API to make the metrics available for the metrics server.

### Metrics server - getting started
If you're using minikube for your local cluster, run: `minikube addons enable metrics-server`. For all other environments, deploy the metrics-server by cloning the
metrics-server deployment files from github repo and then deploying the required components using `k create -f deploy/1.8+/`. This command,
deploys a set of pods, services and roles to enable metrics server, to poll for performance metrics from the nodes in the cluster.
Once deployed, give metrics server some time to collect and process data. Once processed, cluster performance can be viewed by running
`k top node`. This provides the CPU and memory consumption of each of the nodes.

Use `k top pods` to view performance metrics of pods in k8s.

86 - Practice Test Monitoring
87 - Solution Monitoring optional