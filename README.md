# Kubernetes debugging with a tool container

## Introduction

Sometimes, there are issues on your cluster and pods are not communicating or you want to
debug something in the cluster related to networking, DNS, or anything else.  Sometimes
you want to test whether network connectivity issues are isolated to your pods or the
node, or the cluster for that matter.

The containers you may want to debug (i.e., the ones debugging your application) may not
build suitable to use for debugging.  For example, they may be just too stripped down to
the base essentials for the applications or they just don't contain any of the debugging
tools you might need.

## Method

The method here is to develop a container image equipped with all of the tools you may want
or need to do the debugging.  This could include utilities like ping, nslookup, curl, etc.
or any other "home grown" tools necessary for debugging your custom application.
Once you have a container image, you can upload it to an easily accessible place like
your local docker registry or hub.docker.com (assuming its contents are ok being in the
public domain).

Then you will want to create a pod using that container.  Further, you may want tthat pod
to be scheduled on the presumably problematic Kubernetes node.  You want to do this quickly
so that you can spend the bulk of your time investigating your actual problem.

In the examples below, I assume that kubectl is already setup and pointing to
the relevant Kubernetes cluster.  The node names are of the form "testnode-x" where x
is 1 to 6 for a 6 node Kubernetes cluster.  I also assume that there already exists a
container with all of your tools installed on hub.docker.com/mytool

## Debugging

The first thing you want to do is label your nodes with a special label to help you use
node selectors so that your tool contain is scheduled onto the specific node you want to
look at.


```
# Label your nodes so you can pin your debug pod to one
#
for i in {1..6}; do
  kubectl label nodes testnode-$i dennis=$i
done
```

Now create a yaml file to create your pod; here's mine:

```
$ cat dp-client.yaml
apiVersion: v1
kind: Pod
metadata:
  name: dp-client
spec:
  nodeSelector:
    dennis: "4"
  hostname: dp-client
  containers:
  - image: myuser/mytool
    imagePullPolicy: Always
    name: dp-client
    command: [ "sleep", "999999" ]
```

See that the container is build and then just sleeps to prevent it self from exiting.
Also note that we use the nodeSelector to pin us to testnode-4 (where I'm assuming there's
a problem).

Now, create the container and confirm that pod is on testnode-4; using the "-o wide" option
will show the Kubernetes node where the pod was scheduled.

```
$ kubectl apply -f dp-client.yaml
$ kubectl get pod -o wide
```

Now that the pod has been created, you can "get onto" the pod and start to try things out.

```
$ kubectl exec -ti dp-client -- sh
curl -s http://some-host/health
ping some-host
nslookup some-host
```

After you are done, remove the pod to leave no traces behind.
Also remove the labels.


```
$ kubectl delete pod dp-client
$ kubectl delete pod dp-server
$ kubectl delete service dp-srv1
for i in {1..6}; do
  kubectl label node testnode-1 dennis-
done
```

## Other ways to create a pod

Create quick deployment on the fly with busybox.

```
$ kubectl run dennis-shell -i --image busybox
```

Deployment with busybox plus curl.

```
$ kubectl run dennis1-shell -i —image yauritux/busybox-curl
```

# Other debugging techniques

Sometimes you want to test tcp connectivity using a webserver.  You can easily do this using netcat while
inside the tool container (or any container that has netcat builtin).

Here, we build a webserver listening on port 8888.

```
/home # cat index.html
<html>
  <head>
    <title>Test Connectivity</title>
  </head>
  <body>
    <p>You can see me</p>
  </body>
</html>
/home # nc -l -p 8888 < index.html
```

From another container, we curl to that container's webserver.  You will need to use ``ifconfig`` or the
output of ``kubectl get pod -o wide`` to get the IP address of the container running the webserver.

``` 
/home # curl http://<IPOfFirstContainer>:8888
```

## Using a Kubernetes Service so you can use DNS

It's nice to be able to start a container that can be a webserver and then in another container, we
can run curl but using a name (vs. having to lookup the IP address as done above).  To do this, use
a Kubernetes Service.

```
$ cat dp-server.yaml
apiVersion: v1
kind: Service
metadata:
  name: dp-srv1  <-- this name will get a DNS entry
spec:
  ports:
  - name: webserver1
    port: 8888
    protocol: TCP
    targetPort: 8888
  selector:
    app: dp-server  <-- ensure this matches below

---
apiVersion: v1
kind: Pod
metadata:
  name: dp-server
  labels:
    app: dp-server   <-- enure this matches above
spec:
  nodeSelector:
    dennis: "4"
  hostname: dp-server
  containers:
  - image: busybox
    imagePullPolicy: Always
    name: dp-server
    command: [ "sleep", "999999" ]

$ kubectl apply -f dp-server.yaml
```

From another container, you can do this below.  Note the use of dp-srv1 DNS name vs. having to determine
the IP address as above.  This is convenient because you can use dp-srv1 no matter what IP address is
given to the dp-server container.

```
/home # curl http://dp-srv1:8888
```

## Testing using iperf

Sometimes it's nice to launch a pod on two different k8s nodes and run iperf between them for the
purpose of testing things like network plugins (e.g. Calico).  I realize you can run ping as a network
connectivity test but ping packets have a big spacing between them and if there was a quick network
outage, you would not notice.

If you run an iperf between two pods that reside on two different k8s nodes (using low bandwidth),
connectivity should be ok and iperf should not see any packet loss -- you can see this on the iperf
output.

The calico test is this:

* run iperf between two pods on two different k8s nodes
* upgrade calico (you will see the calico pods restart in a rolling fashion)
* iperf should show no packet loss

Repeat that test using different combinations of k8s node pairs.  The more iperfs you have running,
the better -- i.e. ideally, run iperf full mesh between all k8s node pairs and upgrade calico.
If things work out, you will see no packet loss.  If this is true, you know it's safe to upgrade
calico.

NOTE: when you upgrade calico, there is a time where the calico daemonset must download the new version.
This takes some time and contributes to the amount of time the upgrade takes.  When you run the test
repeatedly, this download time goes away because the image version may have already been downloaded.
To keep the tests consistent, before running any test, goto each k8s host and do `sudo docker rmi ...`
using all versions of calico except for the current one being used.  In fact docker won't let you
delete the one being used.

Sample iperf commands to run on a shell in the pod via `kubectl exec -ti (podName) -- sh`:

Run this on the iperf client pod:

```
# Use iperf server, UDP traffic, report interval of 1 second.
# In this example the pod IP=10.239.73.91
iperf -sui 1
```

Run this on the iperf server pod:

```
# Use iperf client, 5 parallel streams of data, UDP traffic, report interval of 1 second,
# iterate for 1000 times.
#
iperf -P 5 -udi1 -c 10.239.73.91 -t 1000
```

## Conclusion

Using a tool container equipped with all the tools needed to do debugging can be very useful when you
want to troubleshoot your application.  Having the commands and syntax handy will help you spend less
time getting the tool container running and more time troubleshooting your problem.

Another alternative is to build the right debugging tools into your application container.  This would
be even more convenient because you tool container will already be there.  The downside is that there
will be extra software there which may not be appropriate.

