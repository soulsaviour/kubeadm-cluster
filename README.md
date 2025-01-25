# kubeadm-cluster

Ansible playbooks that build a Kubernetes cluster with kubeadm on Ubuntu or
Debian machines: one control-plane node and any number of workers, or a
highly available control plane behind a load balancer. Calico provides pod
networking.

## What the playbooks do

| Step | Role | Hosts |
|------|------|-------|
| Swap off, `overlay` and `br_netfilter` modules, bridge and forwarding sysctls | `common` | all |
| containerd 2.x from Docker's repository, with the systemd cgroup driver | `containerd` | all |
| kubeadm, kubelet and kubectl from pkgs.k8s.io, pinned and held | `kubernetes` | all |
| `kubeadm init` from a config file, with control-plane certs uploaded | `control_plane` | first control-plane node |
| Calico via the Tigera operator, using the cluster's pod CIDR | `cni` | first control-plane node |
| Join further control-plane nodes, one at a time | `join` | other control-plane nodes |
| Join workers with short-lived tokens | `join` | workers |

Every step is safe to re-run: nodes that are already initialised or joined
are left alone.

## Usage

```bash
ansible-galaxy collection install -r requirements.yml
cp inventory/hosts.example.yml inventory/hosts.yml   # your machines
ansible-playbook site.yml
ansible-playbook verify.yml
```

`verify.yml` waits for every node to be Ready, Calico to report Available and
kube-system pods to be running, then resolves
`kubernetes.default.svc.cluster.local` from a throwaway pod, which exercises
CoreDNS and the pod network together.

The admin kubeconfig is copied to `~/.kube/config` for the SSH user on each
control-plane node.

### Settings (`group_vars/all.yml`)

| Variable | Default | |
|----------|---------|-|
| `kubernetes_minor` | `1.37` | Selects the pkgs.k8s.io repository |
| `kubernetes_package_version` | `1.37.2-1.1` | Exact package version installed |
| `pod_network_cidr` | `10.244.0.0/16` | Pod network, passed to kubeadm and Calico |
| `service_cidr` | `10.96.0.0/12` | Service network |
| `control_plane_endpoint` | empty | Load balancer address, required for HA |
| `node_ip` | `ansible_host` | Address kubelet advertises (matters on multi-NIC hosts) |

## Highly available control plane

Put a TCP load balancer (HAProxy, an NLB, kube-vip) in front of port 6443 on
every control-plane node, list three control-plane nodes in the inventory and
set:

```yaml
control_plane_endpoint: "k8s-api.example.com:6443"
```

The first node runs `kubeadm init --upload-certs`. Each additional one fetches
a fresh certificate key and joins with `--control-plane`, serially, because
etcd adds members one at a time. The playbook refuses to run with more than
one control-plane node and no shared endpoint.

## Local lab

```bash
vagrant up
ansible-playbook -i inventory/vagrant.yml site.yml verify.yml
```

Three VirtualBox VMs on `192.168.56.0/24`: `cp1`, `worker1` and `worker2`.
Because Vagrant gives each VM a NAT interface first, `node_ip` makes kubelet
and the API server use the host-only address instead.

## Upgrades

Packages are held, so `apt upgrade` never moves the cluster. To upgrade,
bump `kubernetes_minor` and `kubernetes_package_version`, run the
`kubernetes` role, then follow the kubeadm upgrade procedure (control plane
first, one node at a time, draining workers as you go).

## Tearing down

```bash
ansible-playbook reset.yml            # asks for confirmation
ansible-playbook reset.yml -e force=true
```
