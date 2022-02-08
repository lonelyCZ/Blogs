---
title: Node Group Management
authors: 
  - "@Congrool"
  - "@lonelyCZ"
approvers:
  
creation-date: 2021-11-03
last-updated: 2022-02-08
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
		- [Architecture](#architecture)
		- [Introduced Labels](#introduced-labels)
		- [Introduced ConfigMap](#introduced-configmap)
		- [GroupSchedulingExtender](#groupschedulingextender)
			- [Filter](#filter)
			- [Score](#score)
		- [GroupManagementControllerManager](#groupmanagementcontrollermanager)
			- [NodeGroupController](#nodegroupcontroller)
			- [PropagationPolicyController](#propagationpolicycontroller)
			- [OverridePolicyController](#overridepolicycontroller)
		- [Filter Function of CloudCore](#filter-function-of-cloudcore)
		- [NodeGroup API](#nodegroup-api)
		- [PropagationPolicy API](#propagationpolicy-api)
		- [OverridePolicy API](#overridepolicy-api)
		- [Example](#example)
	- [Plan](#plan)
		- [Develop Plan](#develop-plan)
		- [Test Plan](#test-plan)

## Summary
In some scenarios, we may want to deploy an application among serval locations. In this case, the typical pratice is to write a deployment for each location, which means we have to manage serval deployments for one application. As the number of applications and their required locations continously increasing, it will be more and more complicated to manage.  
The node group management feature will help users to manage nodes in groups, and also provide a way to control how to spread pods among node groups and how to run different editions of pod instances in different node groups.

## Motivation
In edge computing scenarios, nodes are geographically distributed. The same application may be deployed on nodes at different locations.

Taking Deployment as an example, the traditional practice is to set the same label for edge nodes in the same location, and then create Deployment per location. Different deployments will deploy application in the location specified by its NodeSelector.

![image](https://camo.githubusercontent.com/f128002038cd1f1cae5e742c3cb1fa558610e9499527066f83aaffbe2b64bfbd/68747470733a2f2f692e626d702e6f76682f696d67732f323032312f30392f306234666234393264663930623335612e706e67)

However, with the number of locations increasing, operation and maintenance of applications become more and more complex. 

### Goals
* Support all application resources based on pod API automatically.
* Pods can be deployed to nodes at multiple locations with a single Deployment.
* The number of pod and the difference of pod instance required for each node group is specified in the relative policy.
* Support pod rescheduling/restarting when the relative policy has been changed.
* Extend kube-scheduler with scheduler-extender to avoid recompilation.

### Non-goals
* Introduce new CRD for each application resource. 
* Split one deployment into serval deployments for all locations.
* Replace the native kube-scheduler.
* Impose influence on all of the running pod instances.

## Proposal
### Use Cases
* Define a NodeGroup CR to indicate which nodes belong to the nodegroup.
* Define a PropagationPolicy CR that specifies which nodegroup pods should be scheduled to and how many pods should run in this nodegroup.
* Define a OverridePolicy CR that specifies the differences of pod instances in different node groups.

## Design Details
### Architecture
![image](https://i.bmp.ovh/imgs/2022/01/b4ff2a7749cdae29.png)  

The implementation consists of two new components: `GroupSchedulingExtender` and `GroupManagementControllerManager`. When users apply a deployment, kubernetes will automatically create pods for it. The `kube-scheduler` takes the responsibility of scheduling pods to nodes. Users can specify how do these pods spread among different locations with `PropagationPolicy`.  The `kube-scheduler` will ask `GroupSchedulingExtender` for how to schedule pods, and the `GroupSchedulingExtender` will make the decision according to the policy. During runtime, `GroupManagementControllerManager` will be continuously watching policies(including PropagationPolicy and OverridePolicy), nodegroups and apps(such as deployment), and do the neccessary work to reconcile the current condition with the condition what defined in policies. In this example, the `GroupManagementControllerManager` deletes one pod running in the Beijing NodeGroup to make this pod rescheduled. Then a new pod will be created and be scheduled to the Hangzhou NodeGroup by the `kube-scheduler` and the `GroupSchedulingExtender` to make pod number rate of Beijing:Hangzhou as defined in PropagationPolicy. Also, it will respect the `OverridePolicy` and check pods running in nodegroups is actually the edition they need. It is a typical scenario where the nodegroup wants to use its local image registry. Then can set the image field as `hangzhou.registry.io` for pods in hangzhou nodegroup and `beijing.registry.io` for pods in beijing nodegroup.

In the view of applications, deployment in this case, they should specify which PropagationPolicy and OverridePolicy to use through labels. The `GroupSchedulingExtender` will refer to the `PropoagationPolicy` during scheduling progress, and the `cloudcore`, a native compenent of KubeEdge, will refer to the `OverridePolicy` when sending pods to the relative edge nodes.

### Introduced Labels
New labels with following keys are introduced:

`groupmanagement.kubeedge.io/required-propagationpolicy`  
`groupmanagement.kubeedge.io/required-overridepolicy`  
`groupmanagement.kubeedge.io/binding-propagationpolicy`  
`groupmanagement.kubeedge.io/binding-overridepolicy`

First two, called required labels, are used for applications to specify which PropagationPolicy and OverridePolicy it wants to use. Later two, called binding labels, will be added by controllers if they think it's ok.

### Introduced ConfigMap
A new ConfiMap is introduced, called `groupmanagement-config`. It contains information needed by `GroupMangementControllerManager`, including the GVK of appication API if it's CRD, the PodTemplate field path of application API.

### GroupSchedulingExtender
`GroupSchedulingExtender` is implemented based on [scheduler-extender](https://github.com/kubernetes/enhancements/tree/master/keps/sig-scheduling/1819-scheduler-extender) which is a feature supported by kubernetes to extend the capability of `kube-scheduler`. `GroupSchedulingExtender` takes the responsibilty of filtering out the irrelevant nodes and scoring each condidate node according to the relative policy, so that pods can finally be scheduled to the appropriate nodegroup.  

`GroupSchedulingExtender` only cares about propagationpolicy labels. When a pod comes to it, we can find the app it belongs to(through OwnerReference). There are 4 possible situations:

1. having required label, no binding label
2. having required label and binding label, but their values are different
3. having required label and binding label, and their values are same
4. having neither required label and binding label

For 1 and 2, `GroupSchedulingExtender` will not schedule this pod and make it in pending status.  
For 3, `GroupSchedulingExtender` will schedule this pod according to the binding PropagationPolicy.  
For 4, `GroupSchedulingExtender` won't do anything. This pod will be scheduled by native `kube-scehduler`.   

#### Filter
After `kube-scheduler` running all built-in filter plugins, it will send the filtered nodes to `GroupSchedulingExtender` for further filtering. `GroupSchedulingExtender` will filtered out nodes which the pod should not be placed on according to the policy, then send all condidiate nodes back to `kube-scheduler` and continue the scheduling progress. Nodes that will be filtered out in `GroupSchedulingExtender` are as follows:  
1. Node that is not in any nodegroup which the pod should be scheduled to
2. Node that is in the nodegroup which has already had enough replica number of the pod

#### Score
`GroupSchedulingExtender` will give a score for each candidate node that has passed the filter, and then send the scores back to the `kube-scheduler`. `kube-scheduler` will combine all scores(from built-in score plugins and the `GroupSchedulingExtender`), and finally pick one which has the highest score. The score method of `GroupSchedulingExtender` is mainly based on the difference between current pod replica number and desired pod replica number. In other words, a node will get a higher score if it's in a nodegroup with greater value of `DesiredPodReplicaNumber - CurrentPodReplicaNumber`.  
> Note:  
> The score method of `GroupSchedulingExtender` is at the nodegroup level, which means all nodes in the same nodegroup will get the same score from `GroupSchedulingExtender`. The prioritization among these nodes depends on other score plugins in `kube-scheduler`, such as `NodeAffinityPriority`.

### GroupManagementControllerManager 
`GroupManagementControllerManager` contains three controllers, called `NodeGroupController`, `PropagationPolicyController` and `OverridePolicyController`.

#### NodeGroupController
`NodeGroupController` is responsible for reconciling the NodeGroup Object which will collect nodes belonging to the NodeGroup according to node names and labels and fill the status field. `NodeGroupController` only watches the nodegroup API.

#### PropagationPolicyController
`PropagationPolicyController` is responsible for reconciling the current replica number of pods in each nodegroup with the desired distribution status which is defined in the PropagationPolicy. It will evict pods in nodegroups where they should not run and also evict pods in nodegroups when they are redundant(which means the current replica number of pods in the nodegroup exceeds what the policy desires). 

`PropagationPolicyController` should watch configmap `groupmanagement-config`. It will dynamically add/delete informers according to the GVK set in the config. These informers will react to the modification of labels. If it find `groupmanagement.kubeedge.io/required-propagationpolicy`, it will check if the propagation policy actually exists and add binding label for it. Here, we leave extension space before adding binding label. It will also watch PropogationPolicy and NodeGroup resources and do the reconciliation.

#### OverridePolicyController
`OverridePolicyController` is responsible for reconciling the current pod instance edition in each nodegroup with desired pod instance edition which is defined in the OverridePolicy. If inconsistent, it will modify relative paths of pods at APIServer, and then pods will automatically restart. 

`OverridePolicyController` should watch configmap `groupmanagement-config`. It will dynamically add/delete informers according to the GVK set in the config. These informers will react to the modification of labels. If it find `groupmanagement.kubeedge.io/required-overridepolicy`, it will check if the override policy actually exists and add binding label for it. Here, we leave extension space before adding binding label. Currently, we can apply the required override policy for this pod. It will also watch OverridePolicy and NodeGroup resources and do the reconciliation.

### Filter Function of CloudCore
We respect the design of `CloudCore` and keep it as an communication component. `CloudCore` only needs check the label and decides whether to send it to edge nodes. There are 4 possible situations:

1. having required label, no binding label
2. having required label and binding label, but their values are different
3. having required label and binding label, and their values are same
4. having neither required label and binding label

For 1 and 2, cloudcore should not send these pods to edge nodes.  
For 3 and 4, cloudcore should send these pods to edge nodes.

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

type PropagationStrategy string

const (
	StaticWeightPropagationStrategy PropagationStrategy = "StaticWeight"
	NumRangePropagationStrategy     PropagationStrategy = "NumRange"
)

// PropagationPolicySpec represents the desired behavior of PropagationPolicy.
type PropagationPolicySpec struct {
	// PropagationStrategy defines how to propagate pod instances among nodegroups.
	// It can be either "StaticWeight" or "NumRange". Only the relative field will be used.
	// For example:
	// If set PropagationStrategy as "StaticWeight", the StaticWeight field will be used and
	// other fields will be ignored.
	// +optional
	PropagationStrategy PropagationStrategy `json:"propagationStrategy"`

	// StaticWeightList defines the static nodegroup weight.
	// +optional
	StaticWeight []StaticNodeGroupWeight `json:"staticWeightList"`

	// NumRange defines the pod number range of each nodegroup.
	// +optional 
	NumRange []SpreadConstraint `json:"numRange"`
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
	// +required
	Weight int64 `json:"weight"`
}
```

### OverridePolicy API
OverridePolicy specifies how to override paths of resources. It has especial support for pod API.

```go
// OverridePolicy represents the policy that overrides a group of resources to one or more nodegroups.
type OverridePolicy struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	// Spec represents the desired behavior of OverridePolicy.
	Spec OverrideSpec `json:"spec"`
}

// OverrideSpec defines the desired behavior of OverridePolicy.
type OverrideSpec struct {
	// OverrideRules defines a collection of override rules on target nodegroups.
	// +optional
	OverrideRules []RuleWithNodeGroup `json:"overrideRules,omitempty"`
}

// RuleWithNodeGroup defines the override rules on nodegroups.
type RuleWithNodeGroup struct {
	// TargetNodeGroup defines restrictions on this override policy
	// that only applies to resources propagated to the matching nodegroups.
	// nil means matching all nodegroups.
	// +optional
	TargetNodeGroup []string `json:"targetNodeGroup,omitempty"`

	// Overriders represents the override rules that would apply on resources
	// +required
	Overriders Overriders `json:"overriders"`
}

// Overriders offers various alternatives to represent the override rules.
//
// If more than one alternatives exist, they will be applied with following order:
// - ImageOverrider
// - Plaintext
type Overriders struct {
	// Plaintext represents override rules defined with plaintext overriders.
	// +optional
	Plaintext []PlaintextOverrider `json:"plaintext,omitempty"`

	// ImageOverrider represents the rules dedicated to handling image overrides.
	// +optional
	ImageOverrider []ImageOverrider `json:"imageOverrider,omitempty"`

	// CommandOverrider represents the rules dedicated to handling container command
	// +optional
	CommandOverrider []CommandArgsOverrider `json:"commandOverrider,omitempty"`

	// ArgsOverrider represents the rules dedicated to handling container args
	// +optional
	ArgsOverrider []CommandArgsOverrider `json:"argsOverrider,omitempty"`
}

// ImageOverrider represents the rules dedicated to handling image overrides.
type ImageOverrider struct {
	// Predicate filters images before applying the rule.
	//
	// Defaults to nil, in that case, the system will automatically detect image fields if the resource type is
	// Pod, ReplicaSet, Deployment or StatefulSet by following rule:
	//   - Pod: spec/containers/<N>/image
	//   - ReplicaSet: spec/template/spec/containers/<N>/image
	//   - Deployment: spec/template/spec/containers/<N>/image
	//   - StatefulSet: spec/template/spec/containers/<N>/image
	// In addition, all images will be processed if the resource object has more than one containers.
	//
	// If not nil, only images matches the filters will be processed.
	// +optional
	Predicate *ImagePredicate `json:"predicate,omitempty"`

	// Component is part of image name.
	// Basically we presume an image can be made of '[registry/]repository[:tag]'.
	// The registry could be:
	// - k8s.gcr.io
	// - fictional.registry.example:10443
	// The repository could be:
	// - kube-apiserver
	// - fictional/nginx
	// The tag cloud be:
	// - latest
	// - v1.19.1
	// - @sha256:dbcc1c35ac38df41fd2f5e4130b32ffdb93ebae8b3dbe638c23575912276fc9c
	//
	// +required
	Component ImageComponent `json:"component"`

	// Operator represents the operator which will apply on the image.
	// +required
	Operator OverriderOperator `json:"operator"`

	// Value to be applied to image.
	// Must not be empty when operator is 'add' or 'replace'.
	// Defaults to empty and ignored when operator is 'remove'.
	// +optional
	Value string `json:"value,omitempty"`
}

// ImagePredicate describes images filter.
type ImagePredicate struct {
	// Path indicates the path of target field
	// +required
	Path string `json:"path"`
}

// ImageComponent indicates the components for image.
type ImageComponent string

// CommandArgsOverrider represents the rules dedicated to handling command/args overrides.
type CommandArgsOverrider struct {
	// The name of container
	// +required
	ContainerName string `json:"containerName"`

	// Operator represents the operator which will apply on the command/args.
	// +required
	Operator OverriderOperator `json:"operator"`

	// Value to be applied to command/args.
	// Items in Value which will be appended after command/args when Operator is 'add'.
	// Items in Value which match in command/args will be deleted when Operator is 'remove'.
	// If Value is empty, then the command/args will remain the same.
	// +optional
	Value []string `json:"value,omitempty"`
}

const (
	// Registry is the registry component of an image with format '[registry/]repository[:tag]'.
	Registry ImageComponent = "Registry"

	// Repository is the repository component of an image with format '[registry/]repository[:tag]'.
	Repository ImageComponent = "Repository"

	// Tag is the tag component of an image with format '[registry/]repository[:tag]'.
	Tag ImageComponent = "Tag"
)

// PlaintextOverrider is a simple overrider that overrides target fields
// according to path, operator and value.
type PlaintextOverrider struct {
	// Path indicates the path of target field
	Path string `json:"path"`
	// Operator indicates the operation on target field.
	// Available operators are: add, update and remove.
	Operator OverriderOperator `json:"operator"`
	// Value to be applied to target field.
	// Must be empty when operator is Remove.
	// +optional
	Value apiextensionsv1.JSON `json:"value,omitempty"`
}

// OverriderOperator is the set of operators that can be used in an overrider.
type OverriderOperator string

// These are valid overrider operators.
const (
	OverriderOpAdd     OverriderOperator = "add"
	OverriderOpRemove  OverriderOperator = "remove"
	OverriderOpReplace OverriderOperator = "replace"
)

// OverridePolicyList is a collection of OverridePolicy.
type OverridePolicyList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`

	// Items holds a list of OverridePolicy.
	Items []OverridePolicy `json:"items"`
}
```

### Example
We give an example of how to use NodeGroup and PropagationPolicy APIs. 

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

Second, create the propagation policy. In this case, we want 3 pods to run in Hangzhou and 2 pods to run in Beijing. Also, we want these pods to use their local image registry. Thus, we can apply the PropagationPolicy and OverridePolicy like this:

```yaml
apiVersion: groupmanagement.kubeedge.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: nginx-propagationpolicy 
spec:
  propagationStrategy: "StaticWeight" 
  staticWeightList:
  - nodeGroupNames:
    - Beijing 
    weight: 2
  - nodeGroupNames:
    - Hangzhou
    weight: 3
---
apiVersion: groupmanagement.kubeedge.io/v1alpha1
kind: OverridePolicy
metadata:
  name: nginx-overridepolicy
spec:
  overrideRules:
  - targetNodeGroup:
    - Beijing
    imageOverrider:
    - component: "Registry"
      operator: "replace"
      value: "beijing.registry.io"
  - targetNodeGroup:
    - Hangzhou
    imageOverrider:
    - component: "Registry"
      operator: "replace"
      value: "hangzhou.registry.io"
```

Then, add labels 

`groupmanagement.kubeedge.io/required-propagationpolicy: nginx-propagationpolicy`  
`groupmanagement.kubeedge.io/required-overridepolicy: nginx-overridepolicy` 

to the deployment. Actually, the operation order doesn't matter. You can add labels first and then submit policies as well. In either way, we will get what we want.

## Plan
### Develop Plan
- alpha
  - [ ] Support Deployment
  - [ ] Support NumRange PropagationStrategy
  - [ ] Support ImageOverrider, ArgsOverrider and CommandOverrider
- beta
  - [ ] Support Statefulset, DaemonSet, Job
  - [ ] Support CRD application
  - [ ] Support PlaintextOverrider
  - [ ] Support StaticWeight PropagationStrategy
### Test Plan
- Unit Test covering:
  - [ ] `GroupSchedulingExtender` can filter out nodes which the pod should not be scheduled to.
  - [ ] `GroupSchedulingExtender` can give a suitable score for each candicate node.

- E2E Test covering:
  - [ ] Deploy scheduler-extenders in kubeedge cluster.
  - [ ] Apply NodeGroup and check status.
  - [ ] Apply PropagationPolicy and check the distribution status of pods.