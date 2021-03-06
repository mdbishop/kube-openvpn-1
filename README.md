kubernetes-openvpn
==================

[![Docker Repository on Quay](https://quay.io/repository/plange/openvpn/status "Docker Repository on Quay")](https://quay.io/repository/plange/openvpn)
[![Docker Repository on Docker Hub](https://img.shields.io/docker/automated/ptlange/openvpn.svg "Docker Repository on Docker Hub")](https://hub.docker.com/r/ptlange/openvpn/)

## Synopsis
Simple OpenVPN deployment using native kubernetes semantics. There is no persistent storage, CA management (key storage, cert signing) needs to be done outside of the cluster for now. I think this is better - unless you leave your keys on your dev laptop.

## Motivation
The main motivator for this project was having the ability to route service requests back to local apps (running on the VPN client), making life much easier for development environments where developers cannot run the entire app stack locally but need to iterate on 1 app quickly.

## Usage
First, you need to initialize your PKI infrastructure. Easyrsa is bundled in this container, so this is fairly easy. Replace `OVPN_SERVER_URL` with your endpoint to-be.
```
docker run -e OVPN_SERVER_URL=tcp://vpn.my.fqdn:1194 -v $PWD:/etc/openvpn -ti ptlange/openvpn ovpn_initpki
```
Follow the instructions on screen. Remember (or better: securely store) your secure password for the CA. You are now left with a `pki` folder in your current working directory.

Deploy the VPN server (namespace needs to exist already):
```
$ ./kube/deploy.sh
Usage: ./kube/deploy.sh <namespace> <OpenVPN URL> <service cidr> <pod cidr>

$ ./kube/deploy.sh default tcp://vpn.my.fqdn:1194 10.3.0.0/24 10.2.0.0/16
secret "openvpn-pki" created
configmap "openvpn-settings" created
configmap "openvpn-ccd" created
deployment "openvpn" created
You have exposed your service on an external port on all nodes in your
cluster.  If you want to expose this service to the external internet, you may
need to set up firewall rules for the service port(s) (tcp:30xxx) to serve traffic.

See http://releases.k8s.io/release-1.3/docs/user-guide/services-firewalls.md for more details.
service "openvpn-ingress" created
```

Your VPN endpoint is now reachable on every node in the cluster on port 30XXX. A cloud loadbalancer should be manually configured to route `vpn.my.fqdn:1194` to all nodes on port 30xxx.

## Accessing the cluster
With the pki still in `$PWD/pki` we can create a new VPN user and grab the `.ovpn` configuration file:

```
# Generate VPN client credentials for CLIENTNAME without password protection; leave 'nopass' out to enter password
docker run -v $PWD:/etc/openvpn -ti ptlange/openvpn easyrsa build-client-full CLIENTNAME nopass
docker run -v $PWD:/etc/openvpn --rm ptlange/openvpn ovpn_getclient CLIENTNAME > CLIENTNAME.ovpn
```

`CLIENTNAME.ovpn` can now be used to connect to the cluster and interact with k8s services and pods directly. Whoohoo!

![One-way traffic](kube/routing1.png "Direct access to kubernetes services")


## Routing back to the client

In order to route cluster traffic back to a service running on the client, we need to assign `CLIENTNAME` a static IP. If you haven't configured an `$OVPN_NETWORK` you need to pick something in the `10.140.0.0/24` range. Keep between `10.140.0.2` and `10.140.0.65` to prevent numbering conflicts. This enables us to route up to 1000 services per client.

Edit the CCD (client configuration directory) configmap:
```
$ kubectl edit configmap openvpn-ccd
```
Look at the example and add another for the `CLIENTNAME` you added before. You do not have to restart openvpn but if you're already connected you will need to reconnect to get the static IP.

![Two-way traffic](kube/routing2.png "Direct access to the client from other kubernetes services!")

Note:
  * The port mapping on the VPN server pod is the same on the VPN client.
  * This means every VPN client needs to be able to configure the port their app runs on (`$ip:$mapped_port`)

## Tests
NONE. See next section.

## Contributing
Please file issues on github. PR's are welcomed.

## Thanks
I used kylemanna/openvpn extensively for a long time and lend quite a few of his scripts for easing the PKI handling. offlinehacker/openvpn-k8s provided some inspiration as well and showed i can run openvpn without any persistent storage, prompting me to write this collection of scripts.
