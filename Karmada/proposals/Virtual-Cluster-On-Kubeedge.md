---
title: Your short, descriptive title
authors:
- "@robot" # Authors' github accounts here.
reviewers:
- "@robot"
- TBD
approvers:
- "@robot"
- TBD

creation-date: 2021-09-27

---

# Virtual Cluster On Kubeedge

<!--
This is the title of your KEP. Keep it short, simple, and descriptive. A good
title can help communicate what the KEP is and should be considered as part of
any review.
-->

## Summary

<!--
This section is incredibly important for producing high-quality, user-focused
documentation such as release notes or a development roadmap. 

A good summary is probably at least a paragraph in length.
-->

In edge computing scenarios, compute nodes are geographically distributed. The same application may be deployed on compute nodes in different regions.  

Taking Deployment as an example, the traditional practice is to first set the same label for edge nodes in the same region, and then create multiple Deployments. Different deployments select different labels through NodeSelector, so as to meet the requirements of deploying the same application to different regions.

![image](https://i.bmp.ovh/imgs/2021/09/0b4fb492df90b35a.png)

However, with the increasing geographical distribution, operation and maintenance becomes more and more complex, which is embodied in the following aspects:

- When the image version is upgraded, a number of related Deployment image version configurations need to be changed.
- A custom Deployment naming convention is required to indicate the same application.
- Lack of a higher perspective for the unified management and operation of these edge groups.
- The complexity of operation increases linearly with the increase of applications and geographical distribution.

Based on the above situation, the nodes in the edge cluster need to be managed in groups. In Karmada, this group can be defined as a kind of virtual cluster and scheduled by Karmada.

  

## Motivation

<!--
This section is for explicitly listing the motivation, goals, and non-goals of
this KEP.  Describe why the change is important and the benefits to users.
-->

Now Karmada has implemented the deployment of resources using K8s native API, and users can deploy resources in multiple clusters without modifying resource templates. This function can be applied to the scenario of edge node group. The edge group is regarded as a Virtual Cluster, and applications can be directly deployed on the nodes of the group.

### Goals

<!--
List the specific goals of the KEP. What is it trying to achieve? How will we
know that this has succeeded?
-->

- In pull mode, Karmada-Agent can create multiple clusters for the same APIServer.
- Add a selector tag to the Cluster to match the nodes in the same group.
- Before delivering an application, inject the matched Cluster selector label into the nodeSelector field of the application template.
- Create namespaces for each Virtual Cluster in Kubeedge to achieve resource isolation.
- Application status collection is compatible with Karmada.

### Non-Goals

<!--
What is out of scope for this KEP? Listing non-goals helps to focus discussion
and make progress.
-->

- The original logic of Karmada will not be modified, but a special agent will be created in Pull mode to support node group management.

## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation. What is the desired outcome and how do we measure success?.
The "Design Details" section below is for the real
nitty-gritty.
-->

### User Stories

<!--
Detail the things that people will be able to do if this KEP is implemented.
Include as much detail as possible so that people can understand the "how" of
the system. The goal here is to make this feel real for users without getting
bogged down.
-->

#### Story1

A bank has dozens of branches, and several nodes in each branch are connected to the K8S control.

The branches are distributed all over the country and are connected to the head office through a variety of network ways, so the network situation is difficult to guarantee and the traditional K8S cluster is difficult to apply. Therefore, edge computing framework like KubeEdge can be selected.

If the nodes of each branch are managed through labels, it will be difficult to maintain later, so each branch can be regarded as a cluster and unified management can be carried out using Karmada.



### Notes/Constraints/Caveats (Optional)

<!--
What are the caveats to the proposal?
What are some important details that didn't come across above?
Go in to as much detail as necessary here.
This might be a good place to talk about core concepts and how they relate.
-->

### Risks and Mitigations

<!--
What are the risks of this proposal, and how do we mitigate? 

How will security be reviewed, and by whom?

How will UX be reviewed, and by whom?

Consider including folks who also work outside the SIG or subproject.
-->

## Design Details

<!--
This section should contain enough information that the specifics of your
change are understandable. This may include API specs (though not always
required) or even code snippets. If there's any ambiguity about HOW your
proposal will be implemented, this is the place to discuss them.
-->

**Agent**: In the original Karmada architecture, an Agent corresponding to an ApiServer can only manage one Cluster, but the whole KubeEdge has only one ApiServer. After dividing KubeEdge into multiple virtual clusters, An Agent needs to be transformed to manage multiple virtual clusters.

![image](https://i.bmp.ovh/imgs/2021/09/ac977055beebeefe.png)

**namespace**: When the native Karmada deploy resources, it deploy resources with the same namespace and name in the corresponding cluster. However, there is actually only one APIServer in KubeEdge, so namespace is needed to isolate resources deployed in the virtual cluster.

For example, an Nginx application is deployed in Karmada's Edge namespace, but in KubeEdge it should be deployed in edge-V-ClusterA namespace and Edge-V-ClusterB namespace.

![image](https://i.bmp.ovh/imgs/2021/09/2f6afb5af1b7e703.png)

#### Realization effect

Namespace

 ![image](https://i.bmp.ovh/imgs/2021/09/9c0937f7805451ae.png)

Deployment

![image](https://i.bmp.ovh/imgs/2021/09/cff002dc3fe1ffbc.png)

Deployment Detail

 ![image](https://i.bmp.ovh/imgs/2021/09/4a96d18a1bbe1f43.png)

Pod

![image](https://i.bmp.ovh/imgs/2021/09/6ba2c5ed40a3788a.png)

Service

 ![image](https://i.bmp.ovh/imgs/2021/09/3d2b9bc855e587ce.png)

### Test Plan

<!--
**Note:** *Not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?

No need to outline all of the test cases, just the general strategy. Anything
that would count as tricky in the implementation, and anything particularly
challenging to test, should be called out.

-->

- Unit Test covering:
  - The agent registers multiple clusters and the Cluster is in the Ready state of True.
  - Agent creates namespace correctly in kubeedge cluster and inject the matched Cluster selector label into the nodeSelector field of the application template.
- E2E Test covering:
  - Deploy agent in kubeedge cluster.
  - Deploy applications to different virtual cluster in kubeedge cluster.

## Alternatives

<!--
What other approaches did you consider, and why did you rule them out? These do
not need to be as detailed as the proposal, but should include enough
information to express the idea and why it was not acceptable.
-->

<!--
Note: This is a simplified version of kubernetes enhancement proposal template.
https://github.com/kubernetes/enhancements/tree/3317d4cb548c396a430d1c1ac6625226018adf6a/keps/NNNN-kep-template
-->