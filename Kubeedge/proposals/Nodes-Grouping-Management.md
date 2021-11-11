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
		- [New Cluster API](#new-cluster-api)
		- [New PropagationPolicy API](#new-propagationpolicy-api)
		- [New ReplicaSchedulingPolicy API](#new-replicaschedulingpolicy-api)
		- [Example](#example)
		- [Test Plan](#test-plan)

## Motivation
In edge computing scenarios, nodes are geographically distributed. The same application may be deployed on nodes in different regions.

Taking Deployment as an example, the traditional practice is to set the same label for edge nodes in the same region, and then create Deployment per region. Different deployments select nodes in different regions through NodeSelector.

![image](https://camo.githubusercontent.com/f128002038cd1f1cae5e742c3cb1fa558610e9499527066f83aaffbe2b64bfbd/68747470733a2f2f692e626d702e6f76682f696d67732f323032312f30392f306234666234393264663930623335612e706e67)

However, with the increasing geographical distribution, operation and maintenance become more and more complex. 

### Goals

* Pods can be deployed to multiple groupings with a single Deployment
* The number of replicas required for each grouping is specified in the relative policy(According to the weight or others)
* Support pods rescheduling when the relative policy has been changed
* Extend kube-scheduler with scheduler-extender to avoid recompilation

### Non-goals

* Copy everything from karmada

## Proposal


### Use Cases

* Define a Cluster CRD to indicate which nodes belong to the cluster
* Define a policy CRD that specifies where and how many the Pods need to be deployed 

## Design Details

### Architecture Diagram

![image](https://i.bmp.ovh/imgs/2021/11/6785d566e09a5746.png)

We can extend kube-scheduler with scheduler-extender to score each node according to the policy, so that pods can be scheduled to the appropriate cluster.

Policy specifies the weights of pod replica number for different clusters. The Server made up of scheduler-extender and PolicyController  is used to score the nodes for the pods with annotation `groupingpolicy.kubeedge.io`.

### Scheduler

#### Filter



#### Score



### Controller



#### Reconcile



### New Cluster API

Cluster, which groups nodes mainly through node labels, also collects resource situations for all the nodes in the cluster for scheduling analysis.

```go
type Cluster struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	// Spec represents the specification of the desired behavior of member cluster.
	Spec ClusterSpec `json:"spec"`

	// Status represents the status of member cluster.
	// +optional
	Status ClusterStatus `json:"status,omitempty"`
}

type ClusterSpec struct {
	// Nodes contains names of all the nodes in the cluster.
	Nodes []string `json:"nodes"`

	// MatchLabels match the nodes that have the labels
	MatchLabels map[string]string `json:"matchLables,omitempty"`
}

type ClusterStatus struct {
	// ContainedNodes represents names of all nodes the cluster contains.
	// +optional
	ContainedNodes []string `json:"containedNodes,omitempty"`
}

```

### New PropagationPolicy API

PropagationPolicy represents the policy that propagates a group of resources to one or more clusters.

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
// Placement represents the rule for select clusters.
type Placement struct {
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
      - targetCluster:
          clusterAffinity:
            clusterNames:
              - beijing
        weight: 2
      - targetCluster:
          clusterAffinity:
            clusterNames:
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