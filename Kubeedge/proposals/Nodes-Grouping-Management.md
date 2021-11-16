---
title: Nodes Grouping Management
authors:
    
approvers:
  
creation-date: 2021-11-03
last-updated: 2021-11-11
status: implementable
---
# Nodes Grouping Management

- [Nodes Grouping Management](#nodes-grouping-management)
	- [Motivation](#motivation)
		- [Goals](#goals)
		- [Non-goals](#non-goals)
	- [Proposal](#proposal)
		- [Use Cases](#use-cases)
	- [Design Details](#design-details)
		- [Architecture Diagram](#architecture-diagram)
		- [Scheduler](#scheduler)
			- [Filter](#filter)
			- [Score](#score)
		- [Controller](#controller)
			- [Reconcile](#reconcile)
		- [New Cluster API](#new-cluster-api)
		- [New PropagationPolicy API](#new-propagationpolicy-api)
		- [Example](#example)
			- [Enable scheduler-extender](#enable-scheduler-extender)
			- [Deploy Application with policy](#deploy-application-with-policy)
			- [Deployment result](#deployment-result)
		- [Test Plan](#test-plan)

## Motivation
In edge computing scenarios, nodes are geographically distributed. The same application may be deployed on nodes in different regions.

Taking Deployment as an example, the traditional practice is to set the same label for edge nodes in the same region, and then create Deployment per region. Different deployments select nodes in different regions through NodeSelector.

![image](https://camo.githubusercontent.com/f128002038cd1f1cae5e742c3cb1fa558610e9499527066f83aaffbe2b64bfbd/68747470733a2f2f692e626d702e6f76682f696d67732f323032312f30392f306234666234393264663930623335612e706e67)

However, with the increasing geographical distribution, operation and maintenance become more and more complex. 

### Goals
* Pods can be deployed to multiple groups of nodes with a single Deployment.
* The number of pod replicas required for each node group is specified in the relative policy.
* Support pod rescheduling when the relative policy has been changed.
* Extend kube-scheduler with scheduler-extender to avoid recompilation.

### Non-goals
* Copy everything from karmada

## Proposal
### Use Cases
* Define a Cluster CR to indicate which nodes belong to the cluster.
* Define a Policy CR that specifies which cluster the pods should be deployed in and how many pods need to be deployed in this cluster.

## Design Details
### Architecture Diagram
![image](https://i.bmp.ovh/imgs/2021/11/6785d566e09a5746.png)
The implementation consists of two new components: Scheduler-Extender and PolicyController. The introduction of both two component is as follows.   
Note: All of the machanisms only work on pods with the annotation key of `groupingpolicy.kubeedge.io`.


### Scheduler
The [scheduler-extender](https://github.com/kubernetes/enhancements/tree/master/keps/sig-scheduling/1819-scheduler-extender) is a feature supported by kubernetes to extend the capability of kube-scheduler. The scheduler-extender takes the responsibilty of filtering out the irrelevant nodes and scoring each condidate node according to the relative policy, so that pods can finally be scheduled to the appropriate cluster.  
#### Filter
After kube-scheduler running all built-in filter plugins, it will send the fitered nodes to scheduler-extender for futher filtering. The scheduler-extender will filter out nodes the pod should not be placed on according to the policy, then send all condidiate nodes back to kube-scheduler to continue the scheduling progress. The nodes that will be filtered out in scheduler-extender are as follows:  
1. Node that is not in any cluster which the pod should be scheduled to
2. Node that is in the cluster which has already had enough replica number of the pod

#### Score
Scheduler-extender will give a score for each candidate node that has passed the filter, and then send its scores back to the kube-scheduler. Kube-scheduler will combine all score results(from score plugins and the scheduler-extender), and finally picks one has the highest score. The score policy of scheduler-extender is mainly based on the difference between current pod replica number and descired pod replica number in the cluster. In other words, a node will get a higher score if it's in a cluster with greater value of `DesiredPodReplicaNumber - CurrentPodReplicaNumber`.  
> Note:  
> The score policy of scheduler-extender is at the cluster level, which means all nodes in the same cluster will get the same score from schduler-extender. The prioritization among these nodes depends on other score plugins in kube-scheduler, such as `NodeAffinityPriority`.


### Controller
PolicyController is responsible for reconciling the current replica number of distributed pods in each cluster with the desried distribution status which is defined in the policy. It will evict pods in clusters where they should not run and also evict pods in clusters when they are redundant(which means the current replica number in the cluster exceeds what the policy desires).   
#### Reconcile
PolicyController is continuously watching the Add/Update/Delete event of Policy and Cluster resource. When some events occur, policy controller will find the associated clusters and policies, then delete pods that are not needed or redundant.  

### New Cluster API
Cluster represents a group of nodes that has the same labels.
```go
// Cluster is the Schema for the clusters API
type Cluster struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	// Spec represents the specification of the desired behavior of member cluster.
	Spec ClusterSpec `json:"spec"`

	// Status represents the status of member cluster.
	// +optional
	Status ClusterStatus `json:"status,omitempty"`
}

// ClusterSpec defines the desired state of Cluster
type ClusterSpec struct {
	// Nodes contains names of all the nodes in the cluster.
	Nodes []string `json:"nodes"`

	// MatchLabels matches the nodes that have the labels
	MatchLabels map[string]string `json:"matchLables,omitempty"`
}

// ClusterStatus defines the observed state of Cluster
type ClusterStatus struct {
	// ContainedNodes represents names of all nodes the cluster contains.
	// +optional
	ContainedNodes []string `json:"containedNodes,omitempty"`
}
```

### New PropagationPolicy API

PropagationPolicy specifies how to propagate pods to clusters.

```go
type PropagationPolicy struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	// Spec represents the desired behavior of PropagationPolicy.
	// +required
	Spec PropagationSpec `json:"spec"`
}

type PropagationSpec struct {
	// ResourceSelectors used to select resources.
	// +required
	ResourceSelectors []ResourceSelector `json:"resourceSelectors"`

	// Placement represents the rule for select clusters to propagate resources.
	// +optional
	Placement ClusterPreferences `json:"placement,omitempty"`
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

// ClusterPreferences describes weight for each cluster or for each group of cluster.
type ClusterPreferences struct {
	// StaticWeightList defines the static cluster weight.
	// +required
	StaticWeightList []StaticClusterWeight `json:"staticWeightList"`
}

// StaticClusterWeight defines the static cluster weight.
type StaticClusterWeight struct {
	// ClusterNames specifies clusters with names.
	// +required
	ClusterNames []string `json:"clusterNames"`

	// Weight expressing the preference to the cluster(s) specified by 'TargetCluster'.
	// +kubebuilder:validation:Minimum=1
	// +required
	Weight int64 `json:"weight"`
}
```
### Example



#### Enable scheduler-extender



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
      - clusterNames:
		- beijing
        weight: 2
      - clusterNames:
        - hangzhou
        weight: 3
```
#### Deployment result



### Test Plan

- Unit Test covering:
  - Cluster selects the corresponding nodes according to the label
  - The nodes can be scored correctly by scheduler-extender

- E2E Test covering:
  - Deploy scheduler-extenders in kubeedge cluster
  - Deploy pods to different clusters in kubeedge cluster