---
title: Nodes Grouping Management
authors:
    
approvers:
  
creation-date: 2021-11-03
last-updated: 2022-01-18
status: implementable
---
# Nodes Grouping Management

## Motivation
In edge computing scenarios, nodes are geographically distributed. The same application may be deployed on nodes in different regions.

Taking Deployment as an example, the traditional practice is to set the same label for edge nodes in the same region, and then create Deployment per region. Different deployments select nodes in different regions through NodeSelector.

![image](https://camo.githubusercontent.com/f128002038cd1f1cae5e742c3cb1fa558610e9499527066f83aaffbe2b64bfbd/68747470733a2f2f692e626d702e6f76682f696d67732f323032312f30392f306234666234393264663930623335612e706e67)

However, with the increasing geographical distribution, operation and maintenance become more and more complex. 

### Goals
* Pods can be deployed to multiple groups of nodes with a single Deployment.
* The number of pod replicas required for each node group is specified in the relative policy.
* Support pod rescheduling when the relative policy has been changed.

### Non-goals
* Copy everything from Karmada

## Proposal
### Use Cases
* Define a NodeGroup CR to indicate which nodes belong to the NodeGroup.
* Define a Policy CR that specifies which NodeGroup the pods should be deployed in and how many pods need to be deployed in this NodeGroup.

## Design Details
### Architecture Diagram
![image](https://i.bmp.ovh/imgs/2022/01/f8b0da832d9feb6e.png)
The implementation consists of two new components: KeScheduler and GroupManagementControllerManager. The introduction of both two component is as follows.   

### KeScheduler
The [scheduler-plugin](https://github.com/kubernetes-sigs/scheduler-plugins) is a feature supported by kubernetes to extend the capability of kube-scheduler. The scheduler-plugin takes the responsibilty of filtering out the irrelevant nodes and scoring each condidate node according to the relative policy, so that pods can finally be scheduled to the appropriate NodeGroup.  
#### Filter
The filter plugin will filter out nodes the pod should not be placed on according to the policy. The nodes that will be filtered out in scheduler-plugin are as follows:  
1. Node that is not in any NodeGroup which the pod should be scheduled to
2. Node that is in the NodeGroup which has already had enough replica number of the pod

#### Score
The score plugins will give a score for each candidate node that has passed the filter, Kube-Scheduler will combine all score results and finally picks one has the highest score. The score policy is mainly based on the difference between current pod replica number and descired pod replica number in the NodeGroup. In other words, a node will get a higher score if it's in a NodeGroup with greater value of `DesiredPodReplicaNumber - CurrentPodReplicaNumber`.  
> Note:  
> The score policy of scheduler-plugin is at the NodeGroup level, which means all nodes in the same NodeGroup will get the same score from schduler-plugin. The prioritization among these nodes depends on other score plugins in Kube-Scheduler, such as `NodeAffinityPriority`.


### Controller
NodeGroupController is responsible for reconciling the current NodeGroup Objects that collect the nodes belong to the NodeGroup according to labels. 

PolicyController is responsible for reconciling the current replica number of distributed pods in each NodeGroup with the desried distribution status which is defined in the policy. It will evict pods in NodeGroups where they should not run and also evict pods in NodeGroups when they are redundant(which means the current replica number in the NodeGroup exceeds what the policy desires).
#### Reconcile
NodeGroupController is continuously watching the Add/Update/Delete event of NodeGroup and Node resource. When some events occur, NodeGroupController will update the status of the NodeGroup.

PolicyController is continuously watching the Add/Update/Delete event of Policy and NodeGroup resource. When some events occur, policy controller will find the associated NodeGroups and policies, then delete pods that are not needed or redundant.  

### New NodeGroup API
NodeGroup represents a group of nodes that has the same labels.
```go
// NodeGroup is the Schema for the nodegroups API
type NodeGroup struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	// Spec represents the specification of the desired behavior of member nodegroup.
	// +required
	Spec NodeGroupSpec `json:"spec"`

	// Status represents the status of member nodegroup.
	// +optional
	Status NodeGroupStatus `json:"status,omitempty"`
}

// NodeGroupSpec defines the desired state of NodeGroup
type NodeGroupSpec struct {
	// Nodes contains names of all the nodes in the nodegroup.
	// +optional
	Nodes []string `json:"nodes,omitempty"`

	// MatchLabels match the nodes that have the labels
	// +optional
	MatchLabels map[string]string `json:"matchLabels,omitempty"`
}

// NodeGroupStatus defines the observed state of NodeGroup
type NodeGroupStatus struct {
	// ContainedNodes represents names of all nodes the nodegroup contains.
	// +optional
	ContainedNodes []string `json:"containedNodes,omitempty"`
}
```

### New PropagationPolicy API

PropagationPolicy specifies how to propagate pods to nodegroups.

```go
// PropagationPolicy represents the policy that propagates a group of resources to one or more nodegroups.
type PropagationPolicy struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	// Spec represents the desired behavior of PropagationPolicy.
	// +required
	Spec PropagationSpec `json:"spec"`
}

// PropagationPolicySpec represents the desired behavior of PropagationPolicy.
type PropagationPolicySpec struct {
	// ResourceSelectors used to select resources.
	// +required
	ResourceSelectors []ResourceSelector `json:"resourceSelectors"`

	// Placement represents the rule for select nodegroups to propagate resources.
	// +optional
	Placement NodeGroupPreferences `json:"placement,omitempty"`
}

// ResourceSelector the resources will be selected.
type ResourceSelector struct {
	// APIVersion represents the API version of the target resources.
	// +required
	APIVersion string `json:"apiVersion"`

	// Kind represents the Kind of the target resources.
	// +required
	Kind string `json:"kind"`

	// Namespace of the target resource.
	// Default is empty, which means inherit from the parent object scope.
	// +optional
	Namespace string `json:"namespace,omitempty"`

	// Name of the target resource.
	// Default is empty, which means selecting all resources.
	// +optional
	Name string `json:"name,omitempty"`

	// A label query over a set of resources.
	// If name is not empty, labelSelector will be ignored.
	// +optional
	LabelSelector *metav1.LabelSelector `json:"labelSelector,omitempty"`
}

// NodeGroupPreferences describes weight for each nodegroups or for each group of nodegroup.
type NodeGroupPreferences struct {
	// StaticWeightList defines the static nodegroup weight.
	// +required
	StaticWeightList []StaticNodeGroupWeight `json:"staticWeightList"`
}

// StaticNodeGroupWeight defines the static NodeGroup weight.
type StaticNodeGroupWeight struct {
	// NodeGroupNames specifies nodegroups with names.
	// +required
	NodeGroupNames []string `json:"nodeGroupNames"`

	// Weight expressing the preference to the nodegroup(s) specified by 'TargetNodeGroup'.
	// +kubebuilder:validation:Minimum=1
	// +required
	Weight int64 `json:"weight"`
}
```
### Example

#### Enable KeScheduler

```yaml
apiVersion: kubescheduler.config.k8s.io/v1beta2
kind: KubeSchedulerConfiguration
leaderElection:
  leaderElect: false
clientConnection:
  kubeconfig: /home/ys3/.kube/config
profiles:
- schedulerName: default-scheduler
  plugins:
    preFilter:
      enabled:
      - name: NodeGroupScheduling
    filter:
      enabled:
      - name: NodeGroupScheduling
    score:
      enabled:
      - name: NodeGroupScheduling
  pluginConfig:
  - name: NodeGroupScheduling
    args:
      kubeConfigPath: /home/ys3/.kube/config
```

####  Deploy Application with policy
The deployment template
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 5
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
```

The PropagationPolicy for nginx deployment
```yaml
apiVersion: policy.kubeedge.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: nginx-propagation
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: nginx
  placement:
    staticWeightList:
      - nodeGroupNames:
        - beijing
        weight: 2
      - nodeGroupNames:
        - hangzhou
        weight: 3
```
#### Deployment result



### Test Plan

- Unit Test covering:
  - NodeGroup selects the corresponding nodes according to the labels
  - The pod can be scheduled correctly by KeScheduler

- E2E Test covering:
  - Deploy KeScheduler and GroupManagerControllerManager in Kubeedge cluster
  - Deploy pods to different NodeGroups in Kubeedge cluster