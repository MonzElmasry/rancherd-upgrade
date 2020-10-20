# Rancherd-upgrade

rancherd-upgrade is an image that is responsible of upgrading rancherd version via the [System Upgrade Controller](https://github.com/rancher/system-upgrade-controller), it does that by doing the following:

- Replace the rancherd binary with the new version
- Kill the old rancherd process allowing the supervisor to restart rancherd with the new version

## Build

To build the rancherd-upgrade image locally, you can run the following:

```
export ARCH=amd64 TAG=v2.5.1
docker build --build-arg ARCH --build-arg TAG --tag ${REPO:=rancher}/rancherd-upgrade:${TAG/+/-} .
```

## Usage

### Prerequisites

- rancherd has to be installed using the install script using the curl command:
```
curl -sfL https://get.rancher.io | sh -
```

## Example

1- To use the image with the system-upgrade-controller, you have first to run the controller either directly or deploy it on the rancherd cluster:

```
kubectl apply -f https://raw.githubusercontent.com/rancher/system-upgrade-controller/master/manifests/system-upgrade-controller.yaml
```

You should see the upgrade controller starting in `system-upgrade` namespace.

2- Label the nodes you want to upgrade with the right label:
```
kubectl label node -l node-role.kubernetes.io/master==true rancherd-upgrade=server
kubectl label node -l node-role.kubernetes.io/master!=true rancherd-upgrade=agent
```

3- Run the upgrade plan in the rancherd cluster

```
---
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: rancherd-server
  namespace: system-upgrade
  labels:
    rancherd-upgrade: server
spec:
  concurrency: 1
  version: v2.5.2-rc1
  nodeSelector:
    matchExpressions:
      - {key: rancherd-upgrade, operator: Exists}
      - {key: rancherd-upgrade, operator: NotIn, values: ["disabled", "false"]}
      - {key: node-role.kubernetes.io/master, operator: In, values: ["true"]}
  serviceAccountName: system-upgrade
  cordon: true
#  drain:
#    force: true
  upgrade:
    image: rancher/rancherd-upgrade
---
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: rancherd-agent
  namespace: system-upgrade
  labels:
    rancherd-upgrade: agent
spec:
  concurrency: 2
  version: v2.5.2-rc1
  nodeSelector:
    matchExpressions:
      - {key: rancherd-upgrade, operator: Exists}
      - {key: rancherd-upgrade, operator: NotIn, values: ["disabled", "false"]}
      - {key: node-role.kubernetes.io/master, operator: NotIn, values: ["true"]}
  serviceAccountName: system-upgrade
  prepare:
    # Since v0.5.0-m1 SUC will use the resolved version of the plan for the tag on the prepare container.
    image: rancher/rancherd-upgrade
    args: ["prepare", "rancherd-server"]
  drain:
    force: true
  upgrade:
    image: rancher/rancherd-upgrade
``` 

The upgrade controller should watch for this plan and execute the upgrade on the labeled nodes. For more information about system-upgrade-controller and plan options please visit [system-upgrade-controller](https://github.com/rancher/system-upgrade-controller) official repo.
