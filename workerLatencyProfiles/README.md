### Worker Latency Profiles

- Openshift 4.11 comes up with a feature to ensure the cluster runs optimally inspite of network latencies between the control plane and the worker nodes.
- The OCP users can leverage this feature to tweak the frequency of the status updates between the control plane and the worker nodes.
- Generally, `Kubelet` updates its node status to the `Kube API Server` every `--node-status-update-frequency` interval which defaults to `10s`
- `Kube Controller Manager` checks the statuses of the kubelet every `--node-monitor-period`(defaults to `5s`) and considers the node healthy if the status is updated by the kubelet within `--node-monitor-grace-period` time which defaults to `40s`
-  Once the node is considered unhealthy it is given `node.kubernetes.io/not-ready` or `node.kubernetes.io/unreachable` taints.
- Pods on such a node gets evicted in `--default-not-ready-toleration-seconds` or `--default-unreachable-toleration-seconds` defaulting to `300s`

### Usage
- An OCP user can update the `workerLatencyProfile` field of the [`nodes.config.openshift.io`](https://github.com/openshift/api/blob/master/config/v1/types_node.go#L19) custom resource `cluster` with the possible values as follows:
  - _Default_ (default value)
    - `--node-status-update-frequency` is set to `10s`
	- `--node-monitor-grace-period` is set to `40s`
	- `--default-not-ready-toleration-seconds`, `--default-unreachable-toleration-seconds` are set to `300`
  - _MediumUpdateAverageReaction_
    - `--node-status-update-frequency` is set to `20s`
	- `--node-monitor-grace-period` is set to `2m`
	- `--default-not-ready-toleration-seconds`, `--default-unreachable-toleration-seconds` are set to `60`
  - _LowUpdateSlowReaction_
    - `--node-status-update-frequency` is set to `1m`
	- `--node-monitor-grace-period` is set to `5m`
	- `--default-not-ready-toleration-seconds`, `--default-unreachable-toleration-seconds` are set to `60`

### Call flow
- The following diagram captures the call flow that happens internally when there is a change detected to the above workerLatencyProfile field.

![OCP-WLP Call flow](https://github.com/sairameshv/docs/blob/main/workerLatencyProfiles/WorkerLatencyProfiles.png "OCP-WLP Call flow")

### Hands-On
- Deploy an OCP 4.11 cluster that has the worker latency profiles feature.
- By default, the cluster comes up with the default workerLatencyProfile even though it is not explicitly displayed in the nodes.config custom resource spec.
- Update the `workerLatencyProfile` to `MediumUpdateAverageReaction`
`$ oc edit nodes.config cluster`
```yaml
    # Please edit the object below. Lines beginning with a '#' will be ignored,
    # and an empty file will abort the edit. If an error occurs while saving this file will be
    # reopened with the relevant failures.
    #
    apiVersion: config.openshift.io/v1
    kind: Node
    metadata:
      annotations:
        include.release.openshift.io/ibm-cloud-managed: "true"
        include.release.openshift.io/self-managed-high-availability: "true"
        include.release.openshift.io/single-node-developer: "true"
        release.openshift.io/create-only: "true"
      creationTimestamp: "2022-09-30T09:28:05Z"
      generation: 1
      name: cluster
      ownerReferences:
      - apiVersion: config.openshift.io/v1
        kind: ClusterVersion
        name: version
        uid: 693f796c-1a32-47f5-b71b-9df540984968
      resourceVersion: "1888"
      uid: 6e759e18-b7ff-40cb-8f4f-69f24b7cd4b1
    spec: 
      workerLatencyProfile: "MediumUpdateAverageReaction"   #Update here
```
- Wait for some time until all the nodes come to `Ready` state, debug into the node, observe the above mentioned parameters are updated as expected.
  - On worker nodes, `cat /etc/kubernetes/kubelet.conf | grep "nodeStatusUpdateFrequency"` to be `20s`
  - On master nodes, `cat /etc/kubernetes/static-pod-resources/kube-controller-manager-pod-<revision-number>/configmaps/config/config.yaml | grep "node-monitor-grace-period"` to be `2m`
  - On master nodes, `cat /etc/kubernetes/static-pod-resources/kube-apiserver-pod-<revision-number>/configmaps/config/config.yaml | grep "default-not-ready-toleration-seconds"` to be `60`
- Create a simple `nginx` deployment with the following configuration.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```
- Stop the `kubelet` service on the worker node where the `nginx` pod is deployed.
`systemctl stop kubelet`
- Collect the `events` related to the `nginx` and observe a `NodeNotReady` event which is created by the `kube-controller-manager` after around `2m`(`node-monitor-grace-period`) of stopping the `kubelet` service.
- Then observe that the `nginx` pod enters the `Terminating` state after `60s` which is the `default-not-ready-toleration-seconds` and a new pod is created on another worker node.

### Summary
- The pod termination in the case of “MediumUpdateAverage” reaction worker latency profile takes around “3m” time interval (node-monitor-grace-period + default-not-ready-toleration-seconds) where as it takes around “5m40s”, “5m60s” in the case of “Default” and “LowUpdateSlowReaction” worker latency profiles respectively.
- We can observe, the Default profile’s reaction time is in between the Medium and the Low profile reaction times.
- The “Default” profile works for most of the cases and depending on the network latencies, high pod densities, high disk i/o, etc. the desired profile can be set.


### References
- [Openshift enhancement proposal](https://github.com/openshift/enhancements/blob/master/enhancements/worker-latency-profile/worker-latency-profile.md)
- https://github.com/kubernetes-sigs/kubespray/blob/master/docs/kubernetes-reliability.md
- https://kubernetes.io/docs/concepts/architecture/nodes/
