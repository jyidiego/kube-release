# BOSH release for Kubernetes

This is a project to better understand how to develop BOSH releases and to get to know Kubernetes better.
This release was based largely off Kelsey Hightower's blog
[Kubernetes the Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)

## Disclaimer

This provides what I'd call a POC quality deployment. This is a work in progress and is primarily to
explore BOSH and Kubernetes. It's also to provide a quick deployment on non-GCE infrastructure.
(i.e. This should work on vSphere, Azure, and whatever else BOSH works on.)

### Upload the BOSH releases

To use this BOSH release first upload it to your BOSH:

```
bosh target BOSH_HOST
git clone https://github.com/jyidiego/kube-release.git
cd kube-release
bosh upload release releases/kube-release/kube-release-1.0-alpha.0.yml
```

You'll also need a modified version of a etcd release:

```
git clone https://github.com/jyidiego/kube-etcd.git
cd kube-etcd
bosh upload release releases/kube-etcd/kube-etcd-1.0-alpha.0.yml
```

Finally you'll need a consul and docker bosh release:
```
bosh upload release https://bosh.io/d/github.com/cf-platform-eng/docker-boshrelease?v=28.0.2
bosh upload release https://bosh.io/d/github.com/cloudfoundry-incubator/consul-release?v=139
```

### Example BOSH deployment manifest

There are two example deployment manifests one for bosh-lite and the other for aws. They
can be found in the manifests directory. Each of the deployments will require a "seed" set
of IP addresses. Generally you would assign the first three available ip addresses in a given
network for the kubernetes master (which will also be running etcd and consul.) Consul is
being used for DNS. As an example on AWS, if you have a subnet 10.0.1.0/24 to deploy to than you would
do the following:

* Update the both the deployment manifests and cloud-config with your director UUID and AWS subnet id
  security group id
* Update BOSH cloud-config and run the deployment as shown below

```
cd kube-release/manifests/aws
bosh update cloud-config cloud-config.yml
bosh deployment 3-node-with-ssl-consul.yml
bosh deploy
```

### Kubernetes POD Networking

This deployment does not setup networking for the PODs. It uses the kubnet network plugin.

If you deploy to AWS you'll need to follow the instructions given by
Kelsey [here](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/07-network.md)

On bosh-lite you need to setup the routes for each of the kubernetes worker. As an example:
* POD CIDR network: 10.200.0.0/16
* Number of Workers: 3
* Default bosh-lite network: 10.244.2.0/24

If worker1 has the cbr0 10.200.0.1 and eth0 10.244.2.8 than setup the routes
```
route add -net 10.200.1.0/24 gw 10.244.2.9
route add -net 10.200.2.0/24 gw 10.244.2.10
```

If worker2 has the cbr0 10.200.1.1 and eth0 10.244.2.9 than setup the routes
```
route add -net 10.200.0.0/24 gw 10.244.2.8
route add -net 10.200.2.0/24 gw 10.244.2.10
```

If worker3 has the cbr0 10.200.2.1 and eth0 10.244.2.10 than setup the routes
```
route add -net 10.200.0.0/24 gw 10.244.2.8
route add -net 10.200.1.0/24 gw 10.244.2.9
```

It's useful to get an interactive shell into a container. You can do this by:
```
kubectl run -i --tty busybox --image=busybox --restart=Never -- sh
```
Once in the container you can test your networking by pinging
the respective gateways. (i.e. 10.200.0.1, 10.200.1.1, 10.200.2.1)

### Known Issues

* Strangely enough the cbr0 bridge interface doesn't get created until you deploy
  pods. This can cause issues with POD network testing as pinging the gateways above
  won't work till you deploy pods. Use Kelsey's nginx smoke test to bring the cbr0
  interface online.
```
kubectl run nginx --image=nginx --port=80 --replicas=5
```
* BOSH will faithfully self-heal VMs that are not responsive. However on AWS since the POD networking
  uses route tables that are tied to a specific elastic network interface (eni);
  BOSH can in-advertently cause issues when it re-provisions a VM as the routing table will not reflect
  the new routes. This shouldn't be a problem if routes are based on ip networks and addresses.
