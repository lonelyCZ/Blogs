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

creation-date: yyyy-mm-dd

---

# Fake Cluster On Edge

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

以 Deployment 为例，如下图所示，传统的做法是先将相同地域的计算节点设置成相同的标签，然后创建多个 Deployment，不同 Deployment 通过 NodeSelectors 选定不同的标签，从而实现将相同的应用部署到不同地域的需求。

但是随着地域分布越来越多，使得运维变得越来越复杂，具体表现在以下几个方面：

- 当镜像版本升级，需要修改大量相关的 Deployment 的镜像版本配置。
- 需要自定义 Deployment 的命名规范来表明相同的应用。
- 缺少一个更高的视角对这些 Deployment 进行统一管理和运维。运维的复杂性随着应用和地域分布增多出现线性增长。

基于以上情况，需要将边缘集群中的节点进行分组管理，在karmada中，可以将该分组定义为一种虚假的集群，可以让karmada无感知地进行调度。

  

## Motivation

<!--
This section is for explicitly listing the motivation, goals, and non-goals of
this KEP.  Describe why the change is important and the benefits to users.
-->

现在Karmada已经实现了使用K8s原生API进行资源的部署，用户可以不修改资源模板就进行资源的多集群部署。可以将该功能应用到边缘节点分组的场景中，只需要将边缘节点分组当成一个虚假的Cluster，就可以直接将应用部署在该分组的节点之上。

### Goals

<!--
List the specific goals of the KEP. What is it trying to achieve? How will we
know that this has succeeded?
-->

- 在pull模式下，karmada-agent可以为同一个APIServer创建多个Cluster
- 可以为Cluster添加一个selector标签，用于匹配同一个分组中的节点
- 在下发应用时，将匹配的Cluster的selector标签注入到应用模板的nodeSelector字段中
- 在边缘集群中为每个Cluster创建namespace实现资源隔离
- 应用状态的收集与karmada兼容

### Non-Goals

<!--
What is out of scope for this KEP? Listing non-goals helps to focus discussion
and make progress.
-->

- 不会修改karmada原本的逻辑，只是修改pull模式下的karmada-agent，使其支持节点分组管理。

## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation. What is the desired outcome and how do we measure success?.
The "Design Details" section below is for the real
nitty-gritty.
-->

### User Stories (Optional)

<!--
Detail the things that people will be able to do if this KEP is implemented.
Include as much detail as possible so that people can understand the "how" of
the system. The goal here is to make this feel real for users without getting
bogged down.
-->

#### Story 1

#### Story 2

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