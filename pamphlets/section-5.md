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


80 - Practice Test Readiness and Liveness Probes
81 - Solution Readiness and Liveness Probes
82 - Container Logging
83 - Practice Test Container Logging
84 - Solution Logging optional
85 - Monitor and Debug Applications
86 - Practice Test Monitoring
87 - Solution Monitoring optional