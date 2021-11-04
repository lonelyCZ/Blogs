---
title: Nodes Grouping Management
authors:
    
approvers:
  
creation-date: 2021-11-03
last-updated: 2021-11-03
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
In edge computing scenarios, compute nodes are geographically distributed. The same application may be deployed on compute nodes in different regions.

Taking Deployment as an example, the traditional practice is to first set the same label for edge nodes in the same region, and then create multiple Deployments. Different deployments select different labels through NodeSelector, so as to meet the requirements of deploying the same application to different regions.

![image](https://camo.githubusercontent.com/f128002038cd1f1cae5e742c3cb1fa558610e9499527066f83aaffbe2b64bfbd/68747470733a2f2f692e626d702e6f76682f696d67732f323032312f30392f306234666234393264663930623335612e706e67)

However, with the increasing geographical distribution, operation and maintenance becomes more and more complex. 

### Goals

* Pods can be deployed to multiple groupings with a single Deployment
* The number of copies required for each grouping is realized by writing policies(According to the weight or others)
* Pods rescheduling is supported when policies are changed
* Extending kube-scheduler by scheduler-extenders

### Non-goals

* Copy everything from karmada

## Proposal


### Use Cases

* Defining a Cluster to indicate which nodes belong to the group
* Defining policies that indicate which clusters the Pods need to be deployed in, and assign copies according to their weight

## Design Details

### Architecture Diagram

![image](https://i.bmp.ovh/imgs/2021/11/b039a9ebbad1aa7f.png)

We can extend kube-scheduler's functionality with scheduler-extender to re-score each node according to our grouping policy, so that pods can be scheduled to the appropriate grouping.

The PropagationPolicy contains the nodes to which the Pods in Deployment are scheduled based on their weight. The Server made up of scheduler-extender and PolicyController  is used to score the nodes for the pods with annotation `groupingpolicy.kubeedge.io`.



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
	// Region represents the region of the member cluster locate in.
	// +optional
	Region string `json:"region,omitempty"`

	// Zone represents the zone of the member cluster locate in.
	// +optional
	Zone string `json:"zone,omitempty"`

	// Taints attached to the member cluster.
	// Taints on the cluster have the "effect" on
	// any resource that does not tolerate the Taint.
	// +optional
	Taints []corev1.Taint `json:"taints,omitempty"`

	// Nodes contains names of all the nodes in the cluster.
	Nodes []string `json:"nodes"`

	// MatchLabels match the nodes that have the labels
	MatchLabels map[string]string `json:"matchLables,omitempty"`
}

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

// PropagationSpec represents the desired behavior of PropagationPolicy.
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
	// ClusterAffinity represents scheduling restrictions to a certain set of clusters.
	// If not set, any cluster can be scheduling candidate.
	// +optional
	ClusterAffinity *ClusterAffinity `json:"clusterAffinity,omitempty"`

	// ClusterTolerations represents the tolerations.
	// +optional
	ClusterTolerations []corev1.Toleration `json:"clusterTolerations,omitempty"`

	// SpreadConstraints represents a list of the scheduling constraints.
	// +optional
	SpreadConstraints []SpreadConstraint `json:"spreadConstraints,omitempty"`

	// ReplicaScheduling represents the scheduling policy on dealing with the number of replicas
	// when propagating resources that have replicas in spec (e.g. deployments, statefulsets) to member clusters.
	// +optional
	ReplicaScheduling *ReplicaSchedulingStrategy `json:"replicaScheduling,omitempty"`
}
```



### New ReplicaSchedulingPolicy API

ReplicaSchedulingPolicy represents the policy that propagates total number of replicas for deployment.

```go
type ReplicaSchedulingPolicy struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	// Spec represents the desired behavior of ReplicaSchedulingPolicy.
	Spec ReplicaSchedulingSpec `json:"spec"`
}

// ReplicaSchedulingSpec represents the desired behavior of ReplicaSchedulingPolicy.
type ReplicaSchedulingSpec struct {
	// ResourceSelectors used to select resources.
	// +required
	ResourceSelectors []ResourceSelector `json:"resourceSelectors"`

	// TotalReplicas represents the total number of replicas across member clusters.
	// The replicas(spec.replicas) specified for deployment template will be discarded.
	// +required
	TotalReplicas int32 `json:"totalReplicas"`

	// Preferences describes weight for each cluster or for each group of cluster.
	// +required
	Preferences ClusterPreferences `json:"preferences"`
}

// ClusterPreferences describes weight for each cluster or for each group of cluster.
type ClusterPreferences struct {
	// StaticWeightList defines the static cluster weight.
	// +required
	StaticWeightList []StaticClusterWeight `json:"staticWeightList"`
}

// StaticClusterWeight defines the static cluster weight.
type StaticClusterWeight struct {
	// TargetCluster describes the filter to select clusters.
	// +required
	TargetCluster ClusterAffinity `json:"targetCluster"`

	// Weight expressing the preference to the cluster(s) specified by 'TargetCluster'.
	// +kubebuilder:validation:Minimum=1
	// +required
	Weight int64 `json:"weight"`
}

// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// ReplicaSchedulingPolicyList contains a list of ReplicaSchedulingPolicy.
type ReplicaSchedulingPolicyList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`
	Items           []ReplicaSchedulingPolicy `json:"items"`
}
```



### Example

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



### Test Plan

- Unit Test covering:
  - Cluster selects the corresponding nodes according to the label
  - The nodes can be scored correctly by scheduler-extender

- E2E Test covering:
  - Deploy scheduler-extenders in kubeedge cluster
  - Deploy pods to different clusters in kubeedge cluster