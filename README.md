# Kubernetes

Dictionary :)
 * https://www.digitalocean.com/community/tutorials/an-introduction-to-kubernetes
 * http://kubernetes.io/docs/getting-started-guides/kubeadm/
 * https://github.com/kubernetes/kubernetes/issues/34101
 * https://blog.openshift.com/building-kubernetes-bringing-google-scale-container-orchestration-to-the-enterprise/
 * https://research.google.com/pubs/pub43438.html


Up cluster:
```
$ vagrant up
```

The emergence of the type of error log:
```
==> master: mesg:
==> master: ttyname failed
==> master: :
==> master: Inappropriate ioctl for device
==> master: OK
```
This is normal, as it should be.


## Initialization of the cluster

1) Basic initialization
```
$ vagrant ssh master
$ sudo su -
# kubeadm init --api-advertise-addresses=192.168.50.2
```

Initialization - long procedure, shakes a bunch of images docker
but in the end it gives the token and the ip address to connect to
slave node:

An example of a start command:
```
root@master:~# kubeadm init --api-advertise-addresses=192.168.50.2
<master/tokens> generated token: "60998a.9d910d1359285a3f"
<master/pki> created keys and certificates in "/etc/kubernetes/pki"
<util/kubeconfig> created "/etc/kubernetes/admin.conf"
<util/kubeconfig> created "/etc/kubernetes/kubelet.conf"
<master/apiclient> created API client configuration
<master/apiclient> created API client, waiting for the control plane to become ready
<master/apiclient> all control plane components are healthy after 42.879415 seconds
<master/apiclient> waiting for at least one node to register and become ready
<master/apiclient> first node is ready after 6.007718 seconds
<master/discovery> created essential addon: kube-discovery, waiting for it to become ready
<master/discovery> kube-discovery is ready after 27.506506 seconds
<master/addons> created essential addon: kube-proxy
<master/addons> created essential addon: kube-dns

Kubernetes master initialised successfully!

You can now join any number of machines by running the following on each node:

kubeadm join --token 60998a.9d910d1359285a3f 192.168.50.2
```

2) Vagrant tune - run `vagrant-kubernetes-tune.sh`
```
# cd /vagrant
# ./vagrant-kubernetes-tune.sh
```

If something went wrong, to start over, you must
execute on each node:
```
$ cd /vagrant
$ sudo ./vagrant-kubernetes-clean.sh
```


## Connect and slave1 slave2
The new terminal (for slave1 and slave2 machines)
Command instead `kubeadm join --token 1e55bc.eb74aea68d5225d1 192.168.50.2`
substitute the value obtained in step `kubeadm init`:
```
$ vagrant ssh slaveN
$ sudo su -
# kubeadm join --token ed9add.0e789e91c18e731c 192.168.50.2
...
```

An example of a start command:
```
root@slave1:~# kubeadm join --token 60998a.9d910d1359285a3f 192.168.50.2
<util/tokens> validating provided token
<node/discovery> created cluster info discovery client, requesting info from "http://192.168.50.2:9898/cluster-info/v1/?token-id=60998a"
<node/discovery> cluster info object received, verifying signature using given token
<node/discovery> cluster info signature and contents are valid, will use API endpoints [https://192.168.50.2:443]
<node/csr> created API client to obtain unique certificate for this node, generating keys and certificate signing request
<node/csr> received signed certificate from the API server, generating kubelet configuration
<util/kubeconfig> created "/etc/kubernetes/kubelet.conf"

Node join complete:
* Certificate signing request sent to master and response
  received.
* Kubelet informed of new secure connection details.

Run 'kubectl get nodes' on the master to see this machine join.
```

## End of cluster initialization

Make sure that the slave connected to the master
```
# kubectl get nodes
```

An example of a start command:
```
root@master:~# kubectl get nodes
NAME      STATUS    AGE
master    Ready     2m
slave1    Ready     56s
```

Connect Weave Net (network stack for containers):
```
# kubectl apply -f https://git.io/weave-kube
```

Example command call:
```
root@master:~# kubectl apply -f https://git.io/weave-kube
daemonset "weave-net" created
```

Download and run the images takes some time, so
you need to wait a few minutes.


Make sure that the cluster is functioning:
```
# kubectl get pods --all-namespaces
```

```
root@master:~# kubectl get pods --all-namespaces
NAMESPACE     NAME                             READY     STATUS    RESTARTS   AGE
kube-system   etcd-master                      1/1       Running   0          7m
kube-system   kube-apiserver-master            1/1       Running   1          4m
kube-system   kube-controller-manager-master   1/1       Running   0          7m
kube-system   kube-discovery-982812725-ukbaj   1/1       Running   0          7m
kube-system   kube-dns-2247936740-dehju        2/3       Running   0          7m
kube-system   kube-proxy-amd64-blwb7           1/1       Running   0          4m
kube-system   kube-proxy-amd64-n8hk9           1/1       Running   0          1m
kube-system   kube-proxy-amd64-yv2q7           1/1       Running   0          1m
kube-system   kube-scheduler-master            1/1       Running   0          7m
kube-system   weave-net-3h3r0                  2/2       Running   0          48s
kube-system   weave-net-3r7lq                  2/2       Running   0          48s
kube-system   weave-net-kgbb0                  2/2       Running   0          48s
```

## The administrative interface

```
$ vagrant ssh master
$ sudo su -
# kubectl create -f https://rawgit.com/kubernetes/dashboard/master/src/deploy/kubernetes-dashboard.yaml
```

Access
```
$ vagrant ssh master
$ sudo su -
# kubectl proxy --accept-hosts='.*' --address='192.168.50.2' --port=9090
```

And go to the browser http://192.168.50.2:9090/ui/

## Docker Registry

To test the application of the `sample / hello-world`, you must set up a local
Docker Registry. Application Description in `sample / hello-world / README.md`

Set Registry:
```
$ vagrant ssh master
$ cd /vagrant
$ sudo ./vagrant-setup-registry.sh
```

## Logs

### For a custom namespace

```
$ kubectl get pods
NAME                                     READY     STATUS    RESTARTS   AGE
hello-world-deployment-863226332-9si7n   1/1       Running   0          11h
hello-world-deployment-863226332-pgfbf   1/1       Running   1          11h
```

```
$ kubectl logs hello-world-deployment-863226332-9si7n
....
....
```

### For the system namespace,

It should convey the key `-n kube-system`. One pod may contain
running more than one container, it is necessary in such cases to transfer
Key `-c <container_name>`.

Example:
```
$ kubectl get pods -n kube-system
NAME                                    READY     STATUS    RESTARTS   AGE
etcd-master                             1/1       Running   0          17h
heapster-2193675300-ow8nf               1/1       Running   1          10h
kube-apiserver-master                   1/1       Running   1          17h
kube-controller-manager-master          1/1       Running   0          17h
kube-discovery-982812725-60eoz          1/1       Running   0          17h
kube-dns-2247936740-s3hrm               3/3       Running   0          17h
kube-proxy-amd64-jjat4                  1/1       Running   0          17h
kube-proxy-amd64-jur2c                  1/1       Running   0          17h
kube-proxy-amd64-oda80                  1/1       Running   1          17h
kube-scheduler-master                   1/1       Running   0          17h
kubernetes-dashboard-1655269645-338mr   1/1       Running   1          17h
monitoring-grafana-927606581-yg6of      1/1       Running   1          10h
monitoring-influxdb-3276295126-kl8h2    1/1       Running   1          10h
weave-net-9nyoz                         2/2       Running   0          17h
weave-net-adpds                         2/2       Running   0          17h
weave-net-q76fe                         2/2       Running   3          17h

$ kubectl logs kube-dns-2247936740-s3hrm -n kube-system
Error from server: a container name must be specified for pod kube-dns-2247936740-s3hrm, choose one of: [kube-dns dnsmasq healthz]

$ kubectl logs kube-dns-2247936740-s3hrm -n kube-system -c kube-dns
I1021 15:16:49.059599       1 server.go:94] Using https://100.64.0.1:443 for kubernetes master, kubernetes API: <nil>
....
....
```
