---
title: Node Group Management
authors: 
  - "@Congrool"
  - "@lonelyCZ"
approvers:
  
creation-date: 2021-11-03
last-updated: 2022-01-19
status: implementable
---
# Node Group Management

- [Node Group Management](#node-group-management)
	- [Summary](#summary)
	- [Motivation](#motivation)
		- [Goals](#goals)
		- [Non-goals](#non-goals)
	- [Proposal](#proposal)
		- [Use Cases](#use-cases)
	- [Design Details](#design-details)
		- [Architecture Diagram](#architecture-diagram)
		- [GroupSchedulingExtender](#groupschedulingextender)
			- [Filter](#filter)
			- [Score](#score)
		- [GroupManagementControllerManager](#groupmanagementcontrollermanager)
		- [NodeGroup API](#nodegroup-api)
		- [PropagationPolicy API](#propagationpolicy-api)
		- [Example](#example)
	- [Plan](#plan)
		- [Develop Plan](#develop-plan)
		- [Test Plan](#test-plan)

## Summary
In some scenarios, we may want to deploy an application among serval locations. In this case, the typical pratice is to write a deployment for each location, which means we have to manage serval deployments for one application. As the number of applications and their required locations continously increasing, it will be more and more complicated to manage.  
The nodes grouping management feature will help users to group nodes, and also provide a way to control how to spread pods among node groups.

## Motivation
In edge computing scenarios, nodes are geographically distributed. The same application may be deployed on nodes at different locations.

Taking Deployment as an example, the traditional practice is to set the same label for edge nodes in the same location, and then create Deployment per location. Different deployments will deploy application in the location specified by its NodeSelector.

![image](https://camo.githubusercontent.com/f128002038cd1f1cae5e742c3cb1fa558610e9499527066f83aaffbe2b64bfbd/68747470733a2f2f692e626d702e6f76682f696d67732f323032312f30392f306234666234393264663930623335612e706e67)

However, with the number of locations increasing, operation and maintenance of applications become more and more complex. 

### Goals
* Pods can be deployed to nodes at multiple locations with a single Deployment.
* The number of pod replicas required for each node group is specified in the relative policy.
* Support pod rescheduling when the relative policy has been changed.
* Extend kube-scheduler with scheduler-extender to avoid recompilation.

### Non-goals
* Split one deployment into serval deployments for all locations.
* Replace the native kube-scheduler.
* Impose influence on all of the running pod instances.

## Proposal
### Use Cases
* Define a NodeGroup CR to indicate which nodes belong to the nodegroup.
* Define a PropagationPolicy CR that specifies which nodegroup pods should be scheduled to and how many pods should run in this nodegroup.

## Design Details
### Architecture Diagram
![image](https://i.bmp.ovh/imgs/2021/11/6785d566e09a5746.png)  

The implementation consists of two new components: `GroupSchedulingExtender` and `GroupManagementControllerManager`. When users apply a deployment, kubernetes will automatically create pods for it. The `kube-scheduler` takes the responsibility of scheduling pods to nodes. Users can specify how do these pods spread among different locations with `PropagationPolicy`. In this example, the policy defines that pods should spread among Beijing and Hangzhou with rate 2:3. The `kube-scheduler` will ask `GroupSchedulingExtender` for how to schedule pods, and the `GroupSchedulingExtender` will make the decision according to the policy. During runtime, `GroupManagementControllerManager` will continuously watch policies and nodegroups, and do the neccessary work to reconcile the current condition with the condition what defined in the policy. In this example, the `GroupManagementControllerManager` deletes one pod running in the Beijing NodeGroup to make this pod rescheduled. Then a new pod will be created and be scheduled to the Hangzhou NodeGroup by the `kube-scheduler` and the `GroupSchedulingExtender` to make pod number rate of Beijing:Hangzhou as 2:3. 

The details of both two component are as follows.   

### GroupSchedulingExtender
`GroupSchedulingExtender` is implemented based on [scheduler-extender](https://github.com/kubernetes/enhancements/tree/master/keps/sig-scheduling/1819-scheduler-extender) which is a feature supported by kubernetes to extend the capability of `kube-scheduler`. `GroupSchedulingExtender` takes the responsibilty of filtering out the irrelevant nodes and scoring each condidate node according to the relative policy, so that pods can finally be scheduled to the appropriate nodegroup.  
#### Filter
After `kube-scheduler` running all built-in filter plugins, it will send the filtered nodes to `GroupSchedulingExtender` for further filtering. `GroupSchedulingExtender` will filtered out nodes which the pod should not be placed on according to the policy, then send all condidiate nodes back to `kube-scheduler` and continue the scheduling progress. Nodes that will be filtered out in `GroupSchedulingExtender` are as follows:  
1. Node that is not in any nodegroup which the pod should be scheduled to
2. Node that is in the nodegroup which has already had enough replica number of the pod

#### Score
`GroupSchedulingExtender` will give a score for each candidate node that has passed the filter, and then send the scores back to the `kube-scheduler`. `kube-scheduler` will combine all scores(from built-in score plugins and the `GroupSchedulingExtender`), and finally pick one which has the highest score. The score method of `GroupSchedulingExtender` is mainly based on the difference between current pod replica number and desired pod replica number. In other words, a node will get a higher score if it's in a nodegroup with greater value of `DesiredPodReplicaNumber - CurrentPodReplicaNumber`.  
> Note:  
> The score method of `GroupSchedulingExtender` is at the nodegroup level, which means all nodes in the same nodegroup will get the same score from `GroupSchedulingExtender`. The prioritization among these nodes depends on other score plugins in `kube-scheduler`, such as `NodeAffinityPriority`.


### GroupManagementControllerManager 
`GroupManagementControllerManager` contains two controllers, called `NodeGroupController` and `PropagationPolicyController`.

`NodeGroupController` is responsible for reconciling the NodeGroup Object which will collect nodes belonging to the NodeGroup according to node names and labels and fill the status field.

`PropagationPolicyController` is responsible for reconciling the current replica number of pods in each nodegroup with the desired distribution status which is defined in the PropagationPolicy. It will evict pods in nodegroups where they should not run and also evict pods in nodegroups when they are redundant(which means the current replica number of pods in the nodegroup exceeds what the policy desires).   

### NodeGroup API
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

### PropagationPolicy API

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
We give an example of how to use NodeGroup and PropagationPolicy APIs. This example is the same as what the [Architecture Diagram](#architecture-diagram) shows above.  

Assuming that we have 5 nodes at edge, 2 in Hangzhou and 3 in Beijing, called NodeA, NodeB, NodeC, NodeD and NodeE respectively. NodeA and NodeB, which are located in Hangzhou, have the label `location: hangzhou`. NodeC, NodeD and NodeE, which are located in Beijing, have the label `location: beijing`.  Also we have already had a deployment called nginx running in the cluster whose replicas is 5, and currently 2 pods running in Hangzhou and 3 pods running in Beijing.

First, we create nodegroups called beijing and hangzhou. The yaml is as follows:

```yaml
apiVersion: groupmanagement.kubeedge.io/v1alpha1
kind: NodeGroup
metadata:
  name: hangzhou
spec:
  matchLabels:
    location: hangzhou
---
apiVersion: groupmanagement.kubeedge.io/v1alpha1
kind: NodeGroup
metadata:
  name: beijing 
spec:
  matchLabels:
    location: beijing
```

Second, create the propagation policy. In this case, we want 3 pods to run in Hangzhou and 2 pods to run in Beijing. Thus, we can apply the PropagationPolicy like this:

```yaml
apiVersion: groupmanagement.kubeedge.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: nginx-policy 
spec:
  resourceSelectors:
  - apiVersion: apps/v1
    kind: Deployment
    name: nginx
    namespace: default
  placement:
    staticWeightList:
    - nodeGroupNames:
      - Beijing 
      weight: 2
    - nodeGroupNames:
      - Hangzhou
      weight: 3
```
We specify the deployment to control with GVK, namespace and name in the `resourceSelector` of PropagationPolicy. And fill the `staticWeightList` with the desired rate, setting Beijing with weight 2 and Hangzhou with weight 3. After applying this propagation policy, we will get what we want.


## Plan
### Develop Plan
- [ ] Support the overwrite feature, which will enable users to run different pod instances in different locations.
- [ ] Make the extender share the same network namespace with kube-scheduler, in order to solve the network traffic problem.  
### Test Plan

- Unit Test covering:
  - [ ] `GroupSchedulingExtender` can filter out nodes which the pod should not be scheduled to.
  - [ ] `GroupSchedulingExtender` can give a suitable score for each candicate node.

- E2E Test covering:
  - [ ] Deploy scheduler-extenders in kubeedge cluster.
  - [ ] Apply NodeGroup and check status.
  - [ ] Apply PropagationPolicy and check the distribution status of pods.