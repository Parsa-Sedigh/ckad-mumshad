## 1 - Volumes Quiz
There are different types of volumes supported by Kubernetes. A number of cloud providers and storage vendors have built their
storage plugins to support the Kubernetes Infrastructure. Explore the different types of volumes listed
here https://kubernetes.io/docs/concepts/storage in this quiz.

## 115 - Volumes

## 116 - Persistent Volumes

## 117 - Persistent Volume Claims

## 118 - Using PVCs in PODs
## 119 - Practice Test Persistent Volumes
## 120 - Solution Persistent Volume and Persistent Volume Claims Optional
## 121 - Note on optional topics

## 122 - Storage Classes
## 123 - Practice Test Storage Class
## 124 - Why Stateful Sets
Why we need stateful sets?

We are tasked to deploy a DB server. So we install and setup mysql on the server and create a DB.

Now to withstand failures, we are tasked to deploy a high availability solution. So we deploy **additional servers** and install mysql on those as well(we 
wanna do DB replication).
But now we have a blank DB on the new servers. So how do we replicate the data from the original DB to the new DBs on the new servers?
There are many ways of doing this.

There are different topologies available. The most straightforward one being a **single master/multi salve** topology where all writes come into the
master server(the original server) and reads can be served by either the master or any of the slave servers.
So the master server should be set up first, before deploying the slaves. Once the slaves are deployed, perform an initial clone of the DB from the master server
to the first slave. After the initial copy, enable continuous replication from the master to that slave, so that the DB on the slave node is always
in sync with the DB on the master. We then proceed to set up the second slave. We could do it the same way where we clone data from the master,
but every time you do that, it's going to impact the resources on the master, especially the network interface. Since we have a copy of the master's
data on the first slave, it is better to just copy the data from it instead of the master. So we wait for slave 1 to be ready and then clone data from
slave 1 to slave 2. And finally, we enable continuous replication on slave 2 from the master.

Note: Both these slaves are configured with the address of the master host. When replication is initialized, you point the slave to the master, with the
master's host name or address. That way, the slaves know where the master is.

Summary: We want the master to come up first and then the slaves. We want to clone data from master to slave 1 first and then enable continuous replication from
master to slave 1 and then wait for slave 1 to be ready and then clone data from slave 1 to slave 2 and then enable continuous replication from master
to slave 2 and make sure that the slaves are configured with the address of the master node.

So this is the high level plan for deploying a mysql cluster.
![](../img/124-1.png)

Now let's see in the k8s and containers world how to deploy this setup.

We know in k8s, such kind of a deployment in each of the instances including master and slaves, are a pod, part of a deployment. That way we can
easily scale it up or down as required.

In step one, we want the master to come up first and then the slaves and in case of the slaves, we want the slave 1 to come up first, perform an initial
clone of data from the master and in step 4, we want slave 2 to come up next and clone data from slave 1.

**Now with deployments, you can't guarantee that order**. With deployments, all pods part of the deployment come up at the same time.
So the first step, can't be implemented with a deployment. Another requirement is in steps(2 and 5) which is while cloning data from master to slave 1 or
slave one to slave 2, or while enabling continuous replication on the slaves(3, 5, 6) and step 7, we must be able to differentiate the master and the slave
pods from each other. So we want to designate one of these pods as master and others as slave. So we need the master to have a constant hostname or address,
one that does not change. We cannot rely on the IP address as we know by now that IP addresses of pods are dynamically assigned and they may change
when the pod gets re-created. So we definitely need a static hostname.

As we've seen while working with deployments, the pods come up with random names, so that won't help us here. Even if we decide to designate one of
these pods as the master and use it's name to configure the slaves, if that pod(master) crashes and the deployment creates a new pod in it's place,
it's gonna come up with a totally new pod name and now the slaves are pointing to an address that does not exist and because of all of these,
the remaining steps can't be executed and that's where stateful sets come in.

Stateful sets are similar to deployment sets as in they create pods based on a template. They can scale up and down. They can perform rolling updates and
rollbacks. But there are some differences. **With stateful sets, pods are created in a sequential order.** After the first pod is deployed,
it must be in a running and ready state before the next pod is deployed.

So that helps us ensure that the master is deployed first and then slave 1 and then slave 2. That helps us with steps 1 and 4.

Stateful sets assign a unique original index to each pod, a number starting from zero for the first pod and increments by one.
Each pod gets a unique name derived from this index combined with the stateful set name. For example the first pod gets mysql-0,
second gets mysql-1 and ... . So no more random names. You can rely on these names going forward.

You can be sure that the first pod in any stateful set, is always going to be the name of the stateful set followed by `-0`. So you can always
use that as the master in any setup. The pod mysql-2 knows that it has to perform an initial clone of data from the pod mysql-1.
Say if you were to scake up the deployment(stateful set) by deploying another pod, for instance mysql-3, then it would know that it can perform
a clone from mysql-2.

To enable continuous replication, you can now point the slaves to the master at mysql-0. Even if the master fails and the pod for master is re-created,
it would still come up with the same name.

Stateful sets maintain a sticky identity for each of their pods and these help with the remaining steps.

The master is now always the master and available at the address mysql-0.

This is why we need stateful sets.

The orange ones are pods in stateful set.

## 125 - Stateful Sets Introduction
You might not always need a stateful set. It depends on the kind of the app you're trying to deploy. If the instances need to come up in a
particular order, if the instances need a stable name and ... .

You can create a stateful set just like a deployment(with the pod definition template inside it), but with the right `Kind`. A stateful set also
requires a `serviceName`. You must specify the name of a headless service. We will discuss why you need this in the upcoming lecture.

When you create a stateful set, **it creates pods one after the other**, that's **ordered graceful deployment**. Now each pod gets a stable,
unique DNS record on the network that any other app can use to access a pod.

When you scale a stateful set, it scales in an ordered graceful fashion where each pod comes up, becomes ready and only then the next one comes up.
This helps when you want to for example mysql DBs, as each new instance can clone from the previous instance. It works in the **reverse order** when you
scale it down. The last instance is removed first followed by the second last one. The same is true on termination. **When you delete a stateful set,
the pods are deleted in the reverse order.**

That's the default behavior of a stateful set. But you can override that behavior to cause stateful set to not follow an ordered launch. But still have
the other benefits of stateful set such as a stable and unique network id. For that, you could set `podManagementPolicy` field to `Parallel`, to instruct
stateful set to not follow an ordered approach. Instead, deploy all pods in parallel. The default value of this field is OrderedReady.

![](../img/125-1.png)
![](../img/125-2.png)

## 126 - Headless Services
We said when you create a stateful set, it deploys one pod at a time. Gets an ordinal index and then each pod has a stable, unique name which is
mysql-0, mysql-1 and ... . So we can point the slaves to reach the master at mysql-0.

But one thing is missing here. From what we know about services and DNS in k8s, the way you point one app within the cluster to another app,
is through a service. For example to make the DB server accessible to web server, we create a service for the DB. The service acts as a load balancer
for the pods. The traffic comes into the service is balanced across all the pods in the deployment. The service has a cluster IP(an IP in the cluster)
which is the green number in the img and a DNS name(green text) associated with it which usually goes like `<service name>.default.svc.cluster.local` .
So now any other pod within the cluster can use this DNS name to reach the DB.

Now remember we said that since this is a master-slave topology, the reads could be served by master or slaves. But the writes must only
be processed by the master. So it is ok for the web server to **read** from the mysql service we created, but you can't **write** to that service, as it's
going to load balance the writes to all pods under the service and that's not what we want for our mysql cluster(we don't want to write to slave DBs).

So you wanna point the web server to the master DB server **only**. In other words, we wanna reach one specific pod which is exposed by a service. How?

If you know the IP of the master pod, you can configure that in the web server. But as we know, IP addresses are dynamic and can change if the pod is
re-created. So we can't use that. We know that each pod can be reached through it's DNS address, but the pod's DNS address is created from it's IP.
For example if the IP is: 10.40.2.8 , the DNS would be: 10-40-2-8.default.pod.cluster.local . So we can't use that either.

What we need, is a service that does not load balance reqs, but gives us a DNS entry to reach each pod and that's what a headless service is.
![](../img/126-1.png)

**A headless service does not have an IP of it's own(clusterIP has an ip of it's own). It does not perform any load balancing. All it does,
is creating DNS entries for each pod using pod name and a subdomain.**

So when you create a headless service with the name mysql-h, each pod gets a DNS record created in the
form: `<pod name>.<headless service name>.namespace.<cluster domain>` for example the first pod would get: mysql-0.mysql-h.default.svc.cluster.local .

The web app can now point to the DNS entry for the master mysql server at mysql-0.mysql-h.default.svc.cluster.local and that should always work.
That DNS entry will always point to the master pod in the mysql deployment.
![](../img/126-2.png)

For a headless service, we need to set `ClusterIP: None` .

When the headless service is created, the DNS entries are created for pods, only if the two conditions are met while creating the pod.
Under the spec section of a pod definition file, you have two optional fields: `hostname` and `subdomain`. You must specify the subdomain to
a value the same as the name of the headless service. When you do that, it creates a DNS record for the name of the service to point to the pod.
However, it still doesn't create A record for individual pods. For that, you must the hostname option in the pod definition file. Only then it creates
a DNS record with the pod name as well.

So these are two properties that are required on the pod for a headless service to create a DNS record for the pod.
![](../img/126-3.png)

This was for an the actual pod-definition file. Let's see how it works in the deployment object.

When you deploy the pod as part of a deployment, by default if there is no hostname or subdomain specified, the deployment does not add a hostname
or subdomain to the pod. So the headless service does not create A records for the pods. If we specified the hostname or subdomain in the pod `template`
section like we did for the pod in the previous img, then it assigns the same hostname and subdomain for all the pods of that deployment.
Because a deployment simply duplicates all the properties for the same pod and that results in creating DNS A records for all pods, but with the **same** name.
For example: `mysql-pod.mysql-h.default.svc.cluster.local` . But all the pods now have the same host name, so this doesn't help us meet our
requirement of addressing the pods separately with separate DNS records. And that is yet another way, how a stateful set differs from a deployment.

When creating a stateful set, you do not need to specify a subdomain or hostname. The stateful set automatically assigns the right hostname for
each pod, based on the pod name and it automatically assigns the right subdomain based on the headless service name.

Q: But how does a stateful set know which headless service we are using? There could be many headless services created for different apps. How would it know
which one is the headless service we created for this app?

A: When creating a stateful set, you must specify the `serviceName` explicitly. That's how it knows what subdomain to assign to the pod. The stateful set
takes the names that we specified and adds them as subdomain property when the pod is created.

All pods now get a separate DNS record created.
![](../img/126-4.png)

## 127 - Storage in StatefulSets
What we already know about storage in k8s?

With PVs we create volume objects in k8s which are then claimed by PVCs and finally used in pod definition files.

That's a single PV mapped to a single PVC to a single pod definition file.
![](../img/127-1.png)

Now with dynamic provisioning we then said: With storage class definition we take out the manual creation of PVs and use storage provisioners to automatically
provision volume on cloud providers. Now the PV is created automatically, we still create a PVC manually and associate that to a pod.
![](../img/127-2.png)

Now this works fine for a pod with the volume.

Q: How does that change with deployments or stateful sets?

A: With stateful sets when you specify the same PVC under the pod-definition(`template`), all pods created by that stateful set, try to use the same volume.
Now this is possible if it is what is desired, as in, you really want multiple pods are multiple instances of your app to share and access the same
storage. That's how you would configure it. And that also depends on the kind of volume created and the provision are used. Note that not all
storage types support that operation which is read or write from multiple instances at the same time.

Q: Now what if you want separate volumes for each pod? As in the mysql replication use case. 

A: The pods don't want to share data. Instead each pod needs it's own local storage. Each instance has it's own DB and the replication of
data between the DBs is done at the DB level. So then each pod needs a PVC for itself. A PVC is bound to a PV, so each PVC needs a PV and OFC
these PVs can be created from a single or different storage classes. So how do you automatically create a PVC for each pod in a stateful set?
![](../img/127-3.png)

You can achieve that using a volume claim template.

A volume claim template is really a PVC but templetized. It just means instead of creating a PVC manually and then specifying it in the stateful set
definition file, you move the entire PVC definition into a section named volumeClaimTemplates under stateful set `spec`.

So we have now a stateful set with volume claim templates and a storage class definition with the right provisioner for gce.
When the stateful set is created, first it creates the first pod and during the creation of the pod, a PVC is created. The PVC is associated to a 
storage class, so the storage class provisions a volume on GCP and then creates a PV and associates the PV with the volume and then binds the PVC to the PV.

Then the second pod is created, the second pod creates a PVC. Then the storage class provisons a new volume, associates that to a PV and binds the PV to the
PVC and so on for the third pod.

Q: What if one of these pods fail and is re-created or rescheduled onto a node?

A: Stateful sets do not automatically delete the PVC or the associated volume to the pod. Instead, it ensures that the pod is re-attached to the same PVC
that it was attached to before. Thus, **stateful sets ensure stable storage for pods**.
![](../img/127-4.png)

Note: Pods are the orange circles.