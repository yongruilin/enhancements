<!--
**Note:** When your KEP is complete, all of these comment blocks should be removed.

To get started with this template:

- [ ] **Pick a hosting SIG.**
  Make sure that the problem space is something the SIG is interested in taking
  up. KEPs should not be checked in without a sponsoring SIG.
- [ ] **Create an issue in kubernetes/enhancements**
  When filing an enhancement tracking issue, please make sure to complete all
  fields in that template. One of the fields asks for a link to the KEP. You
  can leave that blank until this KEP is filed, and then go back to the
  enhancement and add the link.
- [ ] **Make a copy of this template directory.**
  Copy this template into the owning SIG's directory and name it
  `NNNN-short-descriptive-title`, where `NNNN` is the issue number (with no
  leading-zero padding) assigned to your enhancement above.
- [ ] **Fill out as much of the kep.yaml file as you can.**
  At minimum, you should fill in the "Title", "Authors", "Owning-sig",
  "Status", and date-related fields.
- [ ] **Fill out this file as best you can.**
  At minimum, you should fill in the "Summary" and "Motivation" sections.
  These should be easy if you've preflighted the idea of the KEP with the
  appropriate SIG(s).
- [ ] **Create a PR for this KEP.**
  Assign it to people in the SIG who are sponsoring this process.
- [ ] **Merge early and iterate.**
  Avoid getting hung up on specific details and instead aim to get the goals of
  the KEP clarified and merged quickly. The best way to do this is to just
  start with the high-level sections and fill out details incrementally in
  subsequent PRs.

Just because a KEP is merged does not mean it is complete or approved. Any KEP
marked as `provisional` is a working document and subject to change. You can
denote sections that are under active debate as follows:

```
<<[UNRESOLVED optional short context or usernames ]>>
Stuff that is being argued.
<<[/UNRESOLVED]>>
```

When editing KEPS, aim for tightly-scoped, single-topic PRs to keep discussions
focused. If you disagree with what is already in a document, open a new PR
with suggested changes.

One KEP corresponds to one "feature" or "enhancement" for its whole lifecycle.
You do not need a new KEP to move from beta to GA, for example. If
new details emerge that belong in the KEP, edit the KEP. Once a feature has become
"implemented", major changes should get new KEPs.

The canonical place for the latest set of instructions (and the likely source
of this file) is [here](/keps/NNNN-kep-template/README.md).

**Note:** Any PRs to move a KEP to `implementable`, or significant changes once
it is marked `implementable`, must be approved by each of the KEP approvers.
If none of those approvers are still appropriate, then changes to that list
should be approved by the remaining approvers and/or the owning SIG (or
SIG Architecture for cross-cutting KEPs).
-->
# KEP-753: Sidecar containers

<!--
This is the title of your KEP. Keep it short, simple, and descriptive. A good
title can help communicate what the KEP is and should be considered as part of
any review.
-->

<!--
A table of contents is helpful for quickly jumping to sections of a KEP and for
highlighting any additional information provided beyond the standard KEP
template.

Ensure the TOC is wrapped with
  <code>&lt;!-- toc --&rt;&lt;!-- /toc --&rt;</code>
tags, and then generate with `hack/update-toc.sh`.
-->

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Problems: jobs with sidecar containers](#problems-jobs-with-sidecar-containers)
  - [Problems: log forwarding and metrics sidecar](#problems-log-forwarding-and-metrics-sidecar)
  - [Problems: service mesh](#problems-service-mesh)
  - [Problems: configuration / secrets](#problems-configuration--secrets)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Naming](#naming)
    - [Collection name](#collection-name)
    - [Reuse of <code>restartPolicy</code> field and enum](#reuse-of-restartpolicy-field-and-enum)
    - [Use <code>Always</code> vs. New enum value](#use-always-vs-new-enum-value)
  - [Risks and Mitigations](#risks-and-mitigations)
    - [Scenario 1. User decides to use sidecars as a way to run regular containers](#scenario-1-user-decides-to-use-sidecars-as-a-way-to-run-regular-containers)
    - [Scenario 1.a](#scenario-1a)
    - [Scenario 2. Balloon sidecars](#scenario-2-balloon-sidecars)
    - [Scenario 3. Long initialization tasks running in-parallel](#scenario-3-long-initialization-tasks-running-in-parallel)
    - [Scenario 4. Sidecar that never becomes ready](#scenario-4-sidecar-that-never-becomes-ready)
    - [Scenario 5. Intentional failing or terminating sidecars](#scenario-5-intentional-failing-or-terminating-sidecars)
    - [Scenario 6. Keeping a sidecar alive to keep consuming cycles on termination](#scenario-6-keeping-a-sidecar-alive-to-keep-consuming-cycles-on-termination)
    - [Scenario 7. Risk of porting existing sidecars to the new mechanism naively](#scenario-7-risk-of-porting-existing-sidecars-to-the-new-mechanism-naively)
- [Design Details](#design-details)
  - [Backward compatibility](#backward-compatibility)
  - [kubectl changes](#kubectl-changes)
    - [Without sidecar containers support](#without-sidecar-containers-support)
    - [With sidecar container feature](#with-sidecar-container-feature)
  - [Resources calculation for scheduling and pod admission](#resources-calculation-for-scheduling-and-pod-admission)
  - [Exposing Pod Resource requirements](#exposing-pod-resource-requirements)
    - [Resources calculation and Pod QoS evaluation](#resources-calculation-and-pod-qos-evaluation)
    - [Resource calculation and version skew](#resource-calculation-and-version-skew)
  - [Topology and CPU managers](#topology-and-cpu-managers)
  - [Termination of containers](#termination-of-containers)
  - [Other](#other)
  - [Test Plan](#test-plan)
      - [Prerequisite testing updates](#prerequisite-testing-updates)
      - [Unit tests](#unit-tests)
      - [Integration tests](#integration-tests)
      - [e2e tests](#e2e-tests)
      - [Ready state of a sidecar container is properly used to create/delete endpoints](#ready-state-of-a-sidecar-container-is-properly-used-to-createdelete-endpoints)
      - [Pod lifecycle scenarios without sidecar containers](#pod-lifecycle-scenarios-without-sidecar-containers)
      - [Pod lifecycle scenarios with sidecar containers](#pod-lifecycle-scenarios-with-sidecar-containers)
      - [Kubelet restart test cases](#kubelet-restart-test-cases)
      - [API server is down: failure to update containers status during initialization](#api-server-is-down-failure-to-update-containers-status-during-initialization)
    - [Resource usage testing](#resource-usage-testing)
    - [Upgrade/downgrade testing](#upgradedowngrade-testing)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha](#alpha)
    - [Beta](#beta)
    - [GA](#ga)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
    - [Upgrade strategy](#upgrade-strategy)
    - [Downgrade strategy](#downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Future use of restartPolicy field](#future-use-of-restartpolicy-field)
- [Alternatives](#alternatives)
  - [Pod startup completed condition](#pod-startup-completed-condition)
    - [Readiness probes](#readiness-probes)
    - [Startup probes](#startup-probes)
    - [<code>postStart</code> hook](#poststart-hook)
  - [Alternative 1. Sidecar containers as a separate collection](#alternative-1-sidecar-containers-as-a-separate-collection)
  - [Alternative 2. DependOn semantic between containers](#alternative-2-dependon-semantic-between-containers)
  - [Alternative 3. Phases](#alternative-3-phases)
  - [Alternative 4. TerminatePod on container completion](#alternative-4-terminatepod-on-container-completion)
  - [Alternative 5. Injection of sidecar containers thru the &quot;external&quot; object](#alternative-5-injection-of-sidecar-containers-thru-the-external-object)
- [Infrastructure Needed (Optional)](#infrastructure-needed-optional)
<!-- /toc -->

## Release Signoff Checklist

<!--
**ACTION REQUIRED:** In order to merge code into a release, there must be an
issue in [kubernetes/enhancements] referencing this KEP and targeting a release
milestone **before the [Enhancement Freeze](https://git.k8s.io/sig-release/releases)
of the targeted release**.

For enhancements that make changes to code or processes/procedures in core
Kubernetes—i.e., [kubernetes/kubernetes], we require the following Release
Signoff checklist to be completed.

Check these off as they are completed for the Release Team to track. These
checklist items _must_ be updated for the enhancement to be released.
-->

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [X] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [X] (R) KEP approvers have approved the KEP status as `implementable`
- [X] (R) Design details are appropriately documented
- [X] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input (including test refactors)
  - [X] e2e Tests for all Beta API Operations (endpoints)
  - [X] (R) Ensure GA e2e tests meet requirements for [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md) 
  - [X] (R) Minimum Two Week Window for GA e2e tests to prove flake free
- [X] (R) Graduation criteria is in place
  - [X] (R) [all GA Endpoints](https://github.com/kubernetes/community/pull/1806) must be hit by [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md) 
- [X] (R) Production readiness review completed
- [X] (R) Production readiness review approved
- [X] "Implementation History" section is up-to-date for milestone
- [X] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [X] Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

<!--
**Note:** This checklist is iterative and should be reviewed and updated every time this enhancement is being considered for a milestone.
-->

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

<!--
This section is incredibly important for producing high-quality, user-focused
documentation such as release notes or a development roadmap. It should be
possible to collect this information before implementation begins, in order to
avoid requiring implementors to split their attention between writing release
notes and implementing the feature itself. KEP editors and SIG Docs
should help to ensure that the tone and content of the `Summary` section is
useful for a wide audience.

A good summary is probably at least a paragraph in length.

Both in this section and below, follow the guidelines of the [documentation
style guide]. In particular, wrap lines to a reasonable length, to make it
easier for reviewers to cite specific portions, and to minimize diff churn on
updates.

[documentation style guide]: https://github.com/kubernetes/community/blob/master/contributors/guide/style-guide.md
-->
Sidecar containers are a new type of containers that start among the Init
containers, run through the lifecycle of the Pod and don’t block pod
termination. Kubelet makes a best effort to keep them alive and running while
other containers are running.

## Motivation

<!--
This section is for explicitly listing the motivation, goals, and non-goals of
this KEP.  Describe why the change is important and the benefits to users. The
motivation section can optionally provide links to [experience reports] to
demonstrate the interest in a KEP within the wider Kubernetes community.

[experience reports]: https://github.com/golang/go/wiki/ExperienceReports
-->
The concept of sidecar containers has been around since the early days of
Kubernetes. A clear example is [this Kubernetes blog post](https://kubernetes.io/blog/2015/06/the-distributed-system-toolkit-patterns/#example-1-sidecar-containers)
from 2015 mentioning the sidecar pattern.

Over the years the sidecar pattern has become more common in applications,
gained popularity and the use cases are getting more diverse. The current
Kubernetes primitives handle that well, but they fall short for
several use cases and force weird work-arounds in the applications.

The next sections expand on what the current problems are. But, to give more
context, it is important to highlight that some companies are already using a
fork of Kubernetes with this sidecar functionality added (not all
implementations are the same, but more than one company has a fork for this).

### Problems: jobs with sidecar containers

Imagine you have a Job with two containers: one which does the main processing
of the job and the other is just a sidecar facilitating it. This sidecar could
be a service mesh, a metrics gathering statsd server, etc.

When the main processing finishes, the pod won't terminate until the sidecar
container finishes too. This is problematic for sidecar containers that run
continuously.

There is no simple way to handle this on Kubernetes today. There are
work-arounds for this problem, most of them consist of some form of coupling
between the containers to add some logic where a container that finishes
communicates it so other containers can react. But it gets tricky when you have
more than one sidecar container or want to auto-inject sidecars.

The sidecar will also not be restarted for jobs with `restartPolicy:Never` when
it was OOM killed, which may render the pod unusable if the sidecar provided
secure communication to other containers. The issue gets complicated by the fact
that sidecar containers typically have smaller request, making them the first
target as OOM score adjustment uses the request as a main input for calculation.

### Problems: log forwarding and metrics sidecar

A log forwarding sidecar should start before several other containers, to simplify
getting logs from the startup of other applications and from the Init
containers. Let's call "main" container the app that will log and "logging"
container the sidecar that will facilitate it.

If the logging container starts after the main app, special logic needs to be
implemented to gather logs from the main app. Furthermore, if the logging
container is not yet started and the main app crashes on startup, those logs are
more likely to be lost (depends if logs go to a shared volume or over the
network on localhost, etc.). While you can modify your application to handle
this scenario during startup (as it is probably the change you need to do to
handle sidecar crashes), for shutdown this approach won't work.

On shutdown the ordering behavior is arguably more important: if the logging
container is stopped first, logs for other containers are lost. No matter if
those containers queue them and retry to send them to the logging container, or
if they are persisted to a shared volume. The logging container is already
killed and will not be restarted, as the pod is shutting down. In these cases,
logs are lost.

The same things regarding startup and shutdown apply for a metrics container.

### Problems: service mesh

Service mesh presents a similar problem: you want the service mesh container to
be running and ready before other containers start, so that any inbound/outbound
connections that a container can initiate go through the service mesh.

A similar problem happens for shutdown: if the service mesh container is
terminated prior to the other containers, outgoing traffic from other apps will
be blackholed or not use the service mesh.

However, as none of these are possible to guarantee, most service meshes (like
Linkerd and Istio), use fragile and platform specific workarounds to have the basic functionality.
For example, for termination, projects like
[kubeexit](https://github.com/karlkfi/kubexit) or [custom solutions](https://suraj.io/post/how-to-gracefully-kill-kubernetes-jobs-with-a-sidecar/)
are used. 

Some service meshes depend on secret (like a certificate) downloaded by other
init container to establish secure communication between services. This makes
the problem of ordering of sidecar and init containers harder.

Another complication is between log forwarding and service mesh sidecars.
Service mesh sidecars would provide the networking while log forwarding needs to
be active to upload logs. Startup and tear down sequence of those may be a
complicated problem.

### Problems: configuration / secrets

Some pods use init containers to pull down configuration/secrets and update them
before the main container gets it. Then use sidecars to continue to watch for
changes and perform the updates and push to the main container. This requires
two separate code paths today. Perhaps the same sidecar could be used for both
cases.

### Goals

<!--
List the specific goals of the KEP. What is it trying to achieve? How will we
know that this has succeeded?
-->
This proposal aims to:
- make containers implementing the sidecar pattern first class citizens inside a
  Pod
- solve a Job completion issue when sidecars should run continuously
- allow mixing initContainers and sidecars for a choreographed startup sequence
- allow easy injection of sidecar containers in any Pod
- (to be evaluated after alpha) allow to implement sidecar containers that will
  guaranteed to be running longer than regular containers 

### Non-Goals

<!--
What is out of scope for this KEP? Listing non-goals helps to focus discussion
and make progress.
-->
This proposal doesn't aim to:
- support arbitrary dependencies graphs between containers
- act as a security control to enforce that pod containers only run while the
  sidecar is healthy. Restart of sidecar containers is a best effort
- allow to enforce security boundaries to sidecar containers different that
  other containers. For example, allow Istio to run privileged to configure ip
  tables and disable other containers from doing so
- (alpha) support containers termination ordering

## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation. What is the desired outcome and how do we measure success?.
The "Design Details" section below is for the real
nitty-gritty.
-->
The proposal is to introduce a `restartPolicy` field to init containers and use
it to indicate that an init container is a sidecar container. Kubelet will start
init containers with `restartPolicy=Always` in the order with other init
containers, but instead of waiting for its completion, it will wait for the
container startup completion.

The condition for startup completion will be that the startup probe succeeded
(or if no startup probe defined) and `postStart` handler completed. This
condition is represented with the field `Started` of `ContainerStatus` type. See
the section ["Pod startup completed condition"](#pod-startup-completed-condition)
for considerations on picking this signal.

The field `restartPolicy` will only be accepted on init
containers as part of this KEP. The only supported value proposed in this KEP is `Always`. No other
values will be defined as part of this KEP. Moreover, the field will be
nullable so the default value will be "no value". 

Other values for `restartPolicy` of containers will not be accepted and
containers will follow the logic is currently implemented (documented
[here](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/#:~:text=if%20a%20pod's%20init%20container%20fails%2C%20the%20kubelet%20repeatedly%20restarts%20that%20init%20container%20until%20it%20succeeds.%20however%2C%20if%20the%20pod%20has%20a%20restartpolicy%20of%20never%2C%20and%20an%20init%20container%20fails%20during%20startup%20of%20that%20pod%2C%20kubernetes%20treats%20the%20overall%20pod%20as%20failed.)
and more details can be found in the section
["Future use of restartPolicy field"](#future-use-of-restartpolicy-field)).

Sidecar containers will not block Pod completion - if all regular containers
complete, sidecar containers will be terminated.

The `restartPolicy` field for individual init containers can override the
Pod-level `restartPolicy` for sidecar containers. As a result, even if the Pod's
`restartPolicy` is set to `Never` or `OnFailure`, sidecar containers will still
be restarted.

Note, a separate KEP https://github.com/kubernetes/enhancements/issues/4438 will enable
sidecar containers to be restarted even during Pod termination.

In order to minimize OOM kills of sidecar containers, the OOM adjustment for
these containers will match or fall below the OOM score adjustment of regular
containers in the Pod. This intent to solve the issue
https://github.com/kubernetes/kubernetes/issues/111356 

As part of this KEP we also will be enabling for sidecar containers (those will
not be allowed for other init containers):
- `PostStart` and `PreStop` lifecycle handlers for sidecar containers
- All probes (startup, readiness, liveness)
- Readiness probes of sidecars will contribute to determine the whole Pod
  readiness.

```yaml
kind: Pod
spec:
  initContainers:
  - name: vault-agent
    image: hashicorp/vault:1.12.1
  - name: istio-proxy
    image: istio/proxyv2:1.16.0
    args: ["proxy", "sidecar"]
    restartPolicy: Always
  containers:
  ...
```

### Naming

This section explains the motivation for naming, assuming that sidecar
containers and init containers belong to the same collection and distinguished
using a field. Other alternatives are outlined in other section.

#### Collection name

For this KEP it is important to have sidecar containers be defined among other
init containers to be able to express the initialization order of containers.
The name `initContainers` is not a good fit for sidecar containers as they
typically do more than initialization. The better name can be “infrastructure”
containers. The current idea is to implement sidecars as a part of
`initContainers` and if this introduces too much trouble, the new collection
name may replace the old collection name in future.

*Alternative* is to introduce a new collection `infrastructureContainers` that
replaces semantically `initContainers` and deprecate the `initContainers`. This
collection will allow both - init containers and sidecar containers. Decision on
this alternative can be postponed to after alpha implementation.

Another *alternative* is to instead of containers, insert placeholders to the
`initContainers` collection. Containers themselves are defined in other
collection. This option will likely confuse end users more than will help.

#### Reuse of `restartPolicy` field and enum

The per-container restart policy was a long standing request from the community.
Implementing per-container restart policy introduces a set of challenging
problems for the pod lifecycle. For example, the state keeping for containers
which has already run to completion. Introducing sidecar containers is another
scenario where this field can be semantically used.

Introducing this field on containers opens up opportunities to implement those
long-standing requests from the community.

*Alternative* is to introduce a new field: `ambient: true` on all containers.
This property will make containers be restarted all the time, and will not block
the pod completion.
  - Pros: detaching sidecar proposal from the per-container `restartPolicy`
    proposal
  - Cons: this field is a new concept that will be introduced for the same
    property that is typically controlled by `restartPolicy`.

#### Use `Always` vs. New enum value

The semantic of an `Always` enum value is very close to what sidecars require.
This is why reusing Always to represent sidecars makes a lot of sense for Init
containers.

There are a few pros and cons to reuse `Always` as the value instead of
introducing a new enum value
`UntilPodTermination`/`UntilPodShutdown`/`WithPod`/`Ambient` for `restartPolicy`
on containers. 

Pros:
- Allows the same enum values to be used for the `restartPolicy` of both pods
  and containers, but semantics that are mostly the same.
- Less 
  
Cons:
- There are slight differences between the semantics of `Always` for containers
  and `Always` for pods. The main difference is that a `initContainer` with
  `Always` will be terminated when the pod terminates and has no influence over
  the pod lifecycle. Also, for Pods, the default `restartPolicy` is
  [documented](https://github.com/kubernetes/kubernetes/blob/280473ebc4e45f9001f5f9789c318ff7329bc5f0/staging/src/k8s.io/api/core/v1/types.go#L2753-L2764)
  as `Always` but for `initContainers` it will default to `OnFailure`. We
  believe this can be addressed in the documentation of the existing enum
  fields.

When in future we may support `Always` on regular containers, there will be
interesting case of `Always` having a meaning of non-blocking container for the
Pod with `restartPolicy == Never` or `OnFailure`. This is easy to explain - Pod
lifecycle is controlled by containers with the matching restartPolicy. But it
may be slightly confusing and needs to be carefully reviewed at the time this
feature will be considered.

### Risks and Mitigations

<!--
What are the risks of this proposal, and how do we mitigate? Think broadly.
For example, consider both security and how this will impact the larger
Kubernetes ecosystem.

How will security be reviewed, and by whom?

How will UX be reviewed, and by whom?

Consider including folks who also work outside the SIG or subproject.
-->
The following is the list of hypothetical scenarios how users may decide to
abuse the feature or use it in a non-designed way. This is an exercise to
understand potential side effects from implementing the feature. We are looking
for side effects like abuse of the functionality to implement something
error-prone that sidecars were not designed for. Or causing issues like noisy
neighbors, etc.

#### Scenario 1. User decides to use sidecars as a way to run regular containers

At least one regular container is required by the Pod. Users may decide to run
all other containers as sidecars and have a single container with the Sleep
logic inside with timeout or waiting for some cancellation signal from custom
job orchestration portal. This way users may implement execution with a time
limit defined by that logic. This is something that users cannot express today
“declaratively” and will be possible using new sidecar containers. One may
imagine a scenario when a third party jobs orchestration tool converts all
containers to sidecar containers programmatically and injects a single job
orchestration container into the Pod. This can be seen as an abuse of the
concept, and may lead to issues when pods terminate unexpectedly or Jobs meant
to be run as regular containers start being run as sidecars being restarted on
Completion. However we do not see a major problem with it as this use of sidecar
containers will unlikely lead to "unexpected" behavior - all side effects are
quite clear.

Another reason for users to implement regular containers as sidecars would be to
use any special properties kubelet may apply to sidecar containers. For example,
if restart backoff timeouts will be minimized for sidecar containers (which was
proposed and rejected), users may decide to use this feature for regular
containers by running those as sidecars. Current proposal as written doesn’t
introduce any special properties for sidecars that users may start abusing.
Special OOM score adjustment will unlikely be useful as this kind of abuse will
likely be needed for bigger containers that does not have the problem of OOM
score adjustment to be too low.

#### Scenario 1.a

User decides to run regular container as a sidecar to simply guarantee the
startup sequence of the containers. Users can already implement the startup
sequencing by using blocking nature of `postStart` hook. So using the sidecar
containers instead is not adding any benefits.

As for injecting sidecars into Pods with regular containers run as sidecars -
webhooks will likely inject sidecar containers to be first, so the risk is
minimal here as well.

#### Scenario 2. Balloon sidecars

Users may decide to start the “large request” sidecar containers early to
pre-allocate resources for other containers. The same time asking for less
resources for a regular containers hoping to reuse what is allocated by the
“balloon” sidecar. This was previously impossible to pre-allocate resources like
CPU before any init containers run and may be more critical when resource
managers like CPU or topology managers are being used. If this pattern have any
benefits, people may be incentivized to abuse sidecars this way. However it’s
unclear if this has any benefits for users at all. 

#### Scenario 3. Long initialization tasks running in-parallel

Users may decide to implement long initialization tasks that will run
in-parallel with other initialization tasks. Users may also decide to run new
type of initialization tasks  like image preloading for workload containers from
the sidecar container, which will make it run in parallel with the
Initialization tasks. This is impossible to do today as only a single Init
container is run at a time today. For containers with lengthy initialization
this pattern may be abused and can lead to the pattern of sidecars
synchronization when the first workload container will wait for all Init sidecar
containers to complete. This pattern may lead to race conditions and be error
prone.

Today similar behavior can be achieved by running initialization tasks as
regular containers, with the special container that blocks workload from
execution using synchronization logic in the `PostStart` handler. Sidecars support
makes implementation of this feature easier.

There is not much risk, however, even if the user abuses this pattern by
converting all init containers in sidecars. Whenever this pattern is useful, the
user will most likely need to spend time understanding the logic of different
Init Containers and making sure it is not conflicting. This pattern also likely
prohibited for large scale customers because sidecar-based initialization will
cost resources even after initialization is complete.

#### Scenario 4. Sidecar that never becomes ready 

This is the failure mode when the sidecar never becomes ready. This is
functionally equivalent to the init container that never completes and not
allowing users to implement any other ways to abuse kubelet.

#### Scenario 5. Intentional failing or terminating sidecars

Users may implement a sidecar container that intentionally crashes or terminates
often. This scenario functionally similar to a regular container restarts often
so not much additional overhead or side effects will be created. 

#### Scenario 6. Keeping a sidecar alive to keep consuming cycles on termination

On a multitenant cluster, you may have time limits placed on jobs to enable fair
usage across tenants. If a sidecar can prevent termination indefinitely, it
could be used to perform computation outside the allowed limit.

#### Scenario 7. Risk of porting existing sidecars to the new mechanism naively

There is risk associated with people moving sidecars as implemented today to use
the new pattern. We didn’t receive any feedback on potential downsides. One
scenarios that may be affected is if sidecars decided to terminate itself and
kubelet keeps trying to restart it as one of the main containers are still being
terminated. Based on current patterns for sidecar containers, this is not likely
the problem. 

Another potential problem may be that sidecars will wait for some condition to
mark itself “started” that cannot be met with the new pattern. For example, wait
for other containers to fully start. As sidecar will not be marked as startup
completed, other init containers will not run and Pod will be stuck on
initialization. For example, Knative's sidecar does aggressive probing of the
user's container to ensure they're ready prior to marking the sidecar ready
itself. This prevents the Pod from being included K8s ready endpoints. See the
section ["Pod startup completed condition"](#pod-startup-completed-condition)
for more details.

Switching to the new sidecars approach will slow down Pod start for Istio. Istio
today is not blocking other containers to start during its initialization. With
the switch to the new model, the separation of Initialization stage and main
containers running stage will be more explicit and many implementations will
likely wait for sidecar initialization, effectively slowing down Pod startup
comparing to current approach. This can be eliminated by slight redesign of a
sidecars.


## Design Details

<!--
This section should contain enough information that the specifics of your
change are understandable. This may include API specs (though not always
required) or even code snippets. If there's any ambiguity about HOW your
proposal will be implemented, this is the place to discuss them.
-->
### Backward compatibility

The new field means that any Pod that is not using this field will behave the
same way as before.

The new field will only work with the proper control plane and kubelet. Upgrade
and downgrade scenarios will be covered in the further sections.

Outside of Kubernetes-controlled code, there might be third party controllers or
existing containers relying on the current behavior of init containers.
Behaviors they can rely on:

- Assuming nothing is running in the Pod when init container starts. For
  instance taking PID of the process assuming there are no sidecars.
- Incorrectly calculating the resource usage of a Pod not taking the new type of
  containers into account (e.g. some grafana dashboards may need modification)
- Stripping the restartPolicy:Always from init container as an unknown field
  rendering Pod unfunctional as it will not pass the init stage after this
- OPA rules may reject the Pod with new unknown fields because of failure to
  parse the new field

These potential incompatibilities will be documented.

### kubectl changes

The `kubectl get pods` filters all the Init containers from output when Pod is Running.
As part of this KEP, the output will be extended to include status of sidecar Containers.
#### Without sidecar containers support

For the Pod:

```
initContainers:
  - name: init-config
containers:
  - name: sidecar-1
  - name: sidecar-2
  - name: main
```

Initialization (Waiting)

```
NAME      READY   STATUS     RESTARTS   AGE
test      0/3     Init:0/1   0          0s
```
Running

```
NAME      READY   STATUS     RESTARTS   AGE
test      3/3     Running    0          35s
```

#### With sidecar container feature

For the Pod:

```
initContainers:
  - name: init-config
  - name: sidecar-1
    restartPolicy: Always
  - name: sidecar-2
    restartPolicy: Always
containers:
  - name: main
```

What we have today:

Initialization (Waiting)

```
NAME      READY   STATUS     RESTARTS   AGE
test      0/1     Init:0/3   0          0s
NAME      READY   STATUS     RESTARTS   AGE
test      0/1     Init:1/3   0          5s
NAME      READY   STATUS     RESTARTS   AGE
test      0/1     Init:2/3   0          10s
```

Running

```
NAME      READY   STATUS     RESTARTS   AGE
test      1/1     Running    0          35s
```

What will be returned as part of the KEP implementation:

Initialization (Waiting)

```
NAME      READY   STATUS     RESTARTS   AGE
test      0/3     Init:0/3   0          0s
NAME      READY   STATUS     RESTARTS   AGE
test      0/3     Init:1/3   0          5s
NAME      READY   STATUS     RESTARTS   AGE
test      0/3     Init:2/3   0          10s
```

Running

```
NAME      READY   STATUS     RESTARTS   AGE
test      3/3     Running    0          35s
```

### Resources calculation for scheduling and pod admission

When calculating whether Pod will fit the Node, resource limits and requests are
being calculated.

Resources calculation will change for Pod with sidecar containers. Today
resources are calculated as a maximum between the maximum use of an
InitContainer and Sum of all regular containers:

`Max ( Max(initContainers), Sum(Containers) )`

With the sidecar containers the formula will change.

Easiest formula would be to assume they are running the duration of the init
stage as well as regular containers.

`Max ( Max(nonSidecarInitContainers) + Sum(Sidecar Containers), Sum(Sidecar
Containers) + Sum(Containers) )`

However the true calculations will be different as all init containers that
completed before the first sidecar containers will not need to account for any
sidecar containers for the maximum value calculation.

So the formula, assuming the function:

```
InitContainerUse(i) = Sum(sidecar containers with index < i) + InitContainer(i)
```

Is this:

```
Max ( Max( each InitContainerUse ) , Sum(Sidecar Containers) + Sum(Containers) ) 
```

There is also a [Pod overhead](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-overhead/)
that is being added to the resource usage. This section assumes it will be added
by kubelet independently from this formulae computation.

### Exposing Pod Resource requirements

It’s currently not straightforward for users to know the effective resource
requirements for a pod. The formula for this is:

`Max ( Max(initContainers), Sum(Containers)) + pod overhead`.

This is derived from the fact that init containers run serially and to
completion before non-init containers.  The effective request for each resource
is then the maximum of the largest request for a resource in any init container,
and the sum of that resources across all non-init containers.

The introduction of in place pod updates of resource requirement in
[KEP#1287](https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/1287-in-place-update-pod-resources)
further complicates effective resource requirement calculation as
`Pod.Spec.Containers[i].Resources` becomes a desired state field and may not
represent the actual resources in use.  The KEP notes that:

> Schedulers should use the larger of `Spec.Containers[i].Resources` and
> `Status.ContainerStatuses[i].ResourcesAllocated` when considering available
> space on a node.

We will introduce `ContainerUse` to represent this value:

```
ContainerUse(i) = Max(Spec.Containers[i].Resources, Status.ContainerStatuses[i].ResourcesAllocated)
```

In the absence of KEP 1287, or if the feature is disabled, `ContainerUse` is
simply:

```
ContainerUse(i) = Spec.Containers[i].Resources
```

The sidecar KEP also changes that calculation to be more complicated as sidecar
containers are init containers that do not terminate.  Since init containers
start in order, sidecar resource usage needs to be summed into those init
containers that start after the sidecar.  Defining `InitContainerUse` as:

```
InitContainerUse(i) = Sum(sidecar containers with index < i) + Max(Spec.InitContainers[i].Resources, Status.InitContainerStatuses[i].ResourcesAllocated)
```

allows representing the new formula for a pods resource usage

```
Max ( Max( each InitContainerUse ) , Sum(Sidecar Containers) + Sum(each ContainerUse) ) + pod overhead
```

Even now, users still sometimes find how a pod's effective resource requirements
are calculated confusing or are just unaware of the formula.  The mitigating
quality to this is that init container resource requests are usually lower than
the sum of non-init container resource requests, and can be ignored by users in
those cases.  Software that requires accurate pod resource requirement
information (e.g. kube-scheduler, kubelet, autoscalers) don't have that luxury.
It is too much to ask of users to perform this even more complex calculation
simply to know the amount of free capacity they need for a given resource to
allow a pod to schedule.

This KEP will not expose the total resource requests field to end user
as many decisions on this field need to be made from other KEPs: InPlace pod update 
and Pod Level resources. We do not want to make it harder for those new KEPs
to be implemented by exposing this field prematurely.

#### Resources calculation and Pod QoS evaluation

Sidecar containers will be used for Pod QoS calculation as all other containers.

The logic in
[`GetPodQOS`](https://github.com/kubernetes/kubernetes/blob/release-1.26/pkg/apis/core/helper/qos/qos.go#L38-L101)
not likely will need changes, but needs to be tested with the sidecar
containers.

#### Resource calculation and version skew

In case of a version skew between scheduler and kubelet, or in cases when
scheduler and kubelet has a different value set for the `SidecarContainers` feature gate,
calculation of resources required for a Pod will differ between the scheduler
and a kubelet when the sidecar container created.

In case when scheduler "knows" about the sidecar and kubelet doesn't, there
unlikely be any issues. Scheduler will calculate resources usage for a Pod that
will be equal or more than kubelet will require to run the Pod. So there will be
no overbooking.

If scheduler has the `SidecarContainers` feature gate disabled, the Pod that has a Sidecar
container will not be admitted as validation of the new field will fail.

We will recommend in documentation to not disable the feature gate on scheduler,
while there are any Pods with Sidecar container is running.

### Topology and CPU managers

[NodeResourcesFit scheduler plugin](https://github.com/kubernetes/kubernetes/blob/release-1.26/pkg/scheduler/framework/plugins/noderesources/fit.go#L160-L176)
will need to be updated take sidecar container resource request into
consideration.

Preliminary code analysis didn't expose any issues introducing sidecar
containers. The biggest question is resources reuse for sidecar containers and
other init containers, especially in cases of single NUMA node requirements and
such. This may be non-trivial. The decision on this is not blocking the KEP
though.

From the code, it appears that init containers are treated exactly like
application containers so we don't need to change anything from resource
management point of view. I found references in the code where all the
containers (init containers and application containers) were coalesced before
resources (CPUs, memory and devices) are allocated to them. Here are a few
examples:

- Container Manager:
  https://github.com/kubernetes/kubernetes/blob/release-1.26/pkg/kubelet/cm/container_manager_linux.go#L708
- CPU Manager:
  https://github.com/kubernetes/kubernetes/blob/release-1.26/pkg/kubelet/cm/cpumanager/policy_static.go#L490
- Memory Manager:
  https://github.com/kubernetes/kubernetes/blob/release-1.26/pkg/kubelet/cm/memorymanager/policy_static.go#L372
- Topology Manager:
  - https://github.com/kubernetes/kubernetes/blob/release-1.26/pkg/kubelet/cm/topologymanager/scope.go#L137
  - https://github.com/kubernetes/kubernetes/blob/release-1.26/pkg/kubelet/cm/topologymanager/scope_container.go#L52
  - https://github.com/kubernetes/kubernetes/blob/release-1.26/pkg/kubelet/cm/topologymanager/scope_pod.go#L58


### Termination of containers

In Alpha sidecar containers will be terminated as regular containers. No special
or additional signals will be supported.

In Beta we have thought about introducing additional termination grace period fields
to manage termination duration
([draft proposal](https://docs.google.com/document/d/1B01EdgWJAfkT3l6CIwNwoskQ71eyuwet3mjiQrMQbU8))
and leverage these fields to add reverse order termination of sidecar containers
after the primary containers terminate.

However, we decided on an alternative that doesn't require additional fields or hooks while keeping
the desired behaviors when Pods with sidecars are terminated. While original approach works better
with truly graceful termination where consistency is more important than time taken, proposed approach
works for that scenario as well as a more and more popular scenario of limited time to terminate when
graceful termination is set by external requirement and Pods needs to do best to gracefully terminate
as much as possible (think of a Spot Instances with 30 seconds notification).

Here is the proposed approach:
1. Sidecar containers that have a `PreStop` hook will be notified when the Pod has begun terminating
   by executing the `PreStop` hook. This happens at the same time as regular containers, and begins
   the Pod's termination grace period countdown.
2. Once the last primary container terminates, the last started sidecar container is notified by
   sending a `SIGTERM` signal.
3. The next sidecar (in reverse order) is notified by sending a `SIGTERM` signal after the previous
   sidecar container terminates.
4. This continues until all sidecar containers have terminated, or the Pod's termination grace period
   expires.
5. In the latter case, all remaining containers are notified by a `SIGTERM`, followed by a fixed
   grace period of 2s and finally terminated.
6. The Pod will be terminated after that.

Pseudocode for the above:

```
func terminatePod() {
  // notify all sidecar containers with preStop hook, asynchronously
  for sidecar in sidecarContainers {
    if sidecar has preStop hook {
      go execute preStop hook // async
    }
  }
  // notify all containers with preStop hook and then SIGTERM, asynchronously
  for container in containers {
    if container has preStop hook {
      go func(container) { // async
        execute preStop hook
        send SIGTERM
      }
    }
  }
  for {
    switch {
      case grace period expired:
        for anyContainer in sidecarContainers + containers {
          if anyContainer is running {
            send SIGTERM
          }
        }
        sleep 2s
        for anyContainer in sidecarContainer + containers {
          if anyContainer is running {
            send SIGKILL
          }
        }
        return
      case all containers are terminated:
        // sidecars are terminated in reverse order
        for sidecar in reverse(sidecarContainers) {
          // sidecar is already terminating, let it finish
          if sidecar is terminating {
            break
          }
          // next sidecar to terminate
          else if sidecar is running {
            send SIGTERM
            break
          }
        }
        sleep 1s
      case all sidecarContainers are terminated:
        return
      default:
        sleep 1s
    }
  }
}
```

It is worth noting that, like with regular containers, `PreStop` hook must complete before the `SIGTERM`
signal to stop the sidecar container can be sent. Therefore, ordering and graceful termination of sidecars
can only be guaranteed if the `PreStop` hook completes within the Pod's termination grace period.

Sidecars continue to be restarted until they enter the `Terminated` state which they are notified
by a `SIGTERM` signal. This is to ensure that sidecars that fail are restarted until the TGPS expires.

We might postpone running the `livenessProbe` for restarted sidecar containers during termination
until GA, depending on the implementation complexity.

If we compare this to the initial proposal, the following behaviors are preserved:
- Sidecars should not begin termination until all primary containers have
  terminated.
  - Implicit in this is that sidecars should continue to be restarted until all
    primary containers have terminated.
- Sidecars should terminate serially and in reverse order. I.e. the first
  sidecar to initialize should be the last sidecar to terminate.

The additional benefits of this approach comparing to initial proposal:
- If graceful termination period is short, and mostly taken by the main container, the sidecar containers
  has more time to gracefully terminate, for example, clear up buffers of logging container.
- There is absolutely no change in behavior of main containers - they start graceful termination at exact
  same time as before and can utilize as much of the graceful termination period as they need. The Pod graceful
  termination period semantic also stay unchanged.

### Other

This behavior needs to be adjusted:
https://github.com/kubernetes/enhancements/issues/3676 


### Test Plan

<!--
**Note:** *Not required until targeted at a release.*
The goal is to ensure that we don't accept enhancements with inadequate testing.

All code is expected to have adequate tests (eventually with coverage
expectations). Please adhere to the [Kubernetes testing guidelines][testing-guidelines]
when drafting this test plan.

[testing-guidelines]: https://git.k8s.io/community/contributors/devel/sig-testing/testing.md
-->

[ ] I/we understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes necessary
to implement this enhancement.

##### Prerequisite testing updates

<!--
Based on reviewers feedback describe what additional tests need to be added prior
implementing this enhancement to ensure the enhancements have also solid foundations.
-->

##### Unit tests

<!--
In principle every added code should have complete unit test coverage, so providing
the exact set of tests will not bring additional value.
However, if complete unit test coverage is not possible, explain the reason of it
together with explanation why this is acceptable.
-->

<!--
Additionally, for Alpha try to enumerate the core package you will be touching
to implement this enhancement and provide the current unit coverage for those
in the form of:
- <package>: <date> - <current test coverage>
The data can be easily read from:
https://testgrid.k8s.io/sig-testing-canaries#ci-kubernetes-coverage-unit

This can inform certain test coverage improvements that we want to do before
extending the production code to implement this enhancement.
-->

- `<package>`: `<date>` - `<test coverage>`

There will be many packages touched in process. A few that easy to identify by
areas of a change:

Admission:
- `k8s.io/kubernetes/pkg/kubelet/lifecycle`: `61.7`

Enable probes for sidecar containers
- `k8s.io/kubernetes/pkg/kubelet/prober`: `02/07/2023` - `79.9`

Include sidecars into QoS decision:
- `k8s.io/kubernetes/pkg/kubelet/qos`: `02/07/2023` - `100`

Include sidecar in resources calculation and policy decisions:

- `k8s.io/kubernetes/pkg/kubelet/cm/topologymanager`: `02/07/2023` - `93.2`
- `k8s.io/kubernetes/pkg/kubelet/cm/memorymanager`: `02/07/2023` - `81.2`
- `k8s.io/kubernetes/pkg/kubelet/cm/cpumanager`: `02/07/2023` - `86.4`

Update OOM score adjustment:
-  `k8s.io/kubernetes/pkg/kubelet/oom`: `02/07/2023` - ` 57.1`

##### Integration tests

<!--
Integration tests are contained in k8s.io/kubernetes/test/integration.
Integration tests allow control of the configuration parameters used to start the binaries under test.
This is different from e2e tests which do not allow configuration of parameters.
Doing this allows testing non-default options and multiple different and potentially conflicting command line options.
-->

<!--
This question should be filled when targeting a release.
For Alpha, describe what tests will be added to ensure proper quality of the enhancement.

For Beta and GA, add links to added tests together with links to k8s-triage for those tests:
https://storage.googleapis.com/k8s-triage/index.html
-->

No integration tests are planned. We'll cover this with e2e_node tests.

##### e2e tests

<!--
This question should be filled when targeting a release.
For Alpha, describe what tests will be added to ensure proper quality of the enhancement.

For Beta and GA, add links to added tests together with links to k8s-triage for those tests:
https://storage.googleapis.com/k8s-triage/index.html

We expect no non-infra related flakes in the last month as a GA graduation criteria.
-->

- Test failures: https://storage.googleapis.com/k8s-triage/index.html?pr=1&test=SidecarContainers
- All related tests can be filtered with the SidecarContainers

##### Ready state of a sidecar container is properly used to create/delete endpoints

The sidecar container can expose ports and can be used to handle external traffic.

- Pod with sidecar container exposing the port can receive Service traffic to this Port
- Pod with sidecar container exposing the port with readiness probe marks the Endpoint not ready when probe fails and switched back on when readiness probe succeed
- Pod with sidecar container exposing the port can receive Service traffic to this Port during Pod termination (during graceful termination period)

##### Pod lifecycle scenarios without sidecar containers

- Init containers should start after the previous init container has completed
- Regular containers should start in parallel after all init containers have completed
- Restart behavior of the init containers

##### Pod lifecycle scenarios with sidecar containers

- Init containers should start after the previous restartable init container has started
- Regular containers should start in parallel after all regular init containers have completed and all restartable init containers have started
- Restartable init containers should restart always

##### Kubelet restart test cases

- It should restart the containers in the right order after the node reboot
- It should not restart any completed init containers after the kubelet restart

##### API server is down: failure to update containers status during initialization

From @rata. Interesting test cases where the API server is down and kubelet
wants to start a pod with sidecars. That was problematic in the previous
sidecar KEP iterations.

Let say the API server is up and a pod is being started by the kubelet
- The kubelet still needs to start 3 sidecars in the pod
- The API server crashes (it is not restarted yet)
- The kubelet continues to try to start the sidecars
- The pod is started correctly
- The API server becomes reachable again

I think testing this scenario works is important in early phases (alpha), as in
the past that proved to be tricky. The kubelet is authoritative on which
containers are started and sends it to the API server, but I hit some bugs in
the past where if the API server was down, we couldn't read that a new container
was ready from the kubelet. The code to do it was there even if API server was
down, but something was not working and I didn't debug it. And the end result
was that the pod startup didn't finish until the API server was up again, as
that is when we realized from the kubelet that the sidecar was ready.

I think this code has changed since 1.17 when I tested this, but I don't know if
this issue is fixed. It is a non-trivial scenario that, if it happens to need
more serious code changes in the kubelet to handle it correctly, it will be good
to know in early stages of the KEP IMHO.

#### Resource usage testing

1. Validate that the pod overhead will be accounted for when scheduling a Pod
   with sidecar container.
2. Validate that LimitRanger will apply defaults and consider limits for the Pod
   with the sidecar containers.

#### Upgrade/downgrade testing

1. Kubelet and control plane reject the Pod with the new field if the feature
  gate is disabled.
2. kubelet and control plane reject the Pod with the new field if the feature
  gate was disabled AFTER the Pod with the new field was added.

### Graduation Criteria

#### Alpha

- Feature implemented behind a feature flag
- Initial e2e tests completed and enabled
- E2e testing of the existing scenarios with and without the feature gate turned
  on

#### Beta

- Implement proper termination ordering.
- Add tests with feature activation and deactivation (see [Feature Enablement and Rollback](#feature-enablement-and-rollback)).

#### GA

- All known issues are fixed
- Production use feedback addressed

### Upgrade / Downgrade Strategy

#### Upgrade strategy

Existing sidecars (implemented as regular containers) will still work as
intended, as in the past we don't recognize the new field `restartPolicy` today.

Upgrade will not change any current behaviors.

#### Downgrade strategy

First, there will be no effect on any workload that doesn't use a new field. Any
combination of feature gate enabled/disabled or version skew will work as usual
for that workload.

When the new functionality wasn't yet used, downgrade will not be affected.

Versions of Kubernetes that doesn't have this feature implemented will ignore
and strip out the new field `initContainers`. 

Pods that has already been created will stay being scheduled after the downgrade - 
not be rejected by control
plane nor by kubelet. Both will treat the sidecar container as an Init container.
This may render the Pod unusable as it will stuck in initialization forever -
sidecar container are never exiting.
This behavior has been documented for Alpha release, but we don't see it as a
major issue requiring to wait for 3 releases so kubelet will have the logic
to reject such Pods when the feature gate is disabled to keep Downgrade safe.

**Note**, We have implemented logic for the
[kubelet](https://github.com/kubernetes/kubernetes/blob/f19b62fc0914b38941922afefd1e34eb55f87ee7/pkg/kubelet/lifecycle/predicate.go#L78-L91)
to reject Pods with sidecar containers when feature gate is turned off.
For the control plane -
[kube-apiserver](https://github.com/kubernetes/kubernetes/blob/f19b62fc0914b38941922afefd1e34eb55f87ee7/pkg/api/pod/util.go#L554-L560)
is dropping the field (if it wasn't set before) and
[kube-scheduler](https://github.com/kubernetes/kubernetes/blob/f19b62fc0914b38941922afefd1e34eb55f87ee7/pkg/scheduler/framework/plugins/noderesources/fit.go#L256-L262)
is keeping pods with the field set unschedulable.
See [Upgrade/downgrade testing](#upgradedowngrade-testing) section.

Workloads will have to be deleted and recreated with the old way of handling
sidecars.  Once there is no more Pods using sidecars, node can be downgraded
without side effects.

If downgrade happening from the version with the feature enabled to the previous
version that has this feature support, but feature gate is disabled, kubelet
and control place will reject these Pods.

**Note**, downgrade requires node drain. So we will not support scenarios when
Pod already running on the node will need to be handled by the restarted
kubelet that doesn't know about the sidecar containers.


### Version Skew Strategy

<!--
If applicable, how will the component handle version skew with other
components? What are the guarantees? Make sure this is in the test plan.

Consider the following in developing a version skew strategy for this
enhancement:
- Does this enhancement involve coordinating behavior in the control plane and nodes?
- How does an n-3 kubelet or kube-proxy without this feature available behave when this feature is used?
- How does an n-1 kube-controller-manager or kube-scheduler without this feature available behave when this feature is used?
- Will any other components on the node change? For example, changes to CSI,
  CRI or CNI may require updating that component before the kubelet.
-->
Version skew is possible between the control plane and worker nodes as both
should be aware of the new field used to flag sidecars inside `initContainers`.

Therefore all cluster nodes, including control plane nodes, must be upgraded
before the user can deploy sidecars using the new syntax.

Also, since the feature flag applies to both kubelet and the control plane,
similarly all cluster nodes need to have it enabled before deploying Pods with
sidecars.

For the scenarios when the feature gate is disabled on control plane, but not
disabled on kubelet, users will not be able to schedule Pods with the new field.

For the scenarios when the feature gate is disabled on kubelet, but enabled on
control plane, users will be able to create these Pods, but kubelet will reject
those.

## Production Readiness Review Questionnaire

<!--

Production readiness reviews are intended to ensure that features merging into
Kubernetes are observable, scalable and supportable; can be safely operated in
production environments, and can be disabled or rolled back in the event they
cause increased failures in production. See more in the PRR KEP at
https://git.k8s.io/enhancements/keps/sig-architecture/1194-prod-readiness.

The production readiness review questionnaire must be completed and approved
for the KEP to move to `implementable` status and be included in the release.

In some cases, the questions below should also have answers in `kep.yaml`. This
is to enable automation to verify the presence of the review, and to reduce review
burden and latency.

The KEP must have a approver from the
[`prod-readiness-approvers`](http://git.k8s.io/enhancements/OWNERS_ALIASES)
team. Please reach out on the
[#prod-readiness](https://kubernetes.slack.com/archives/CPNHUMN74) channel if
you need any help or guidance.
-->

### Feature Enablement and Rollback

<!--
This section must be completed when targeting alpha to a release.
-->

###### How can this feature be enabled / disabled in a live cluster?

<!--
Pick one of these and delete the rest.

Documentation is available on [feature gate lifecycle] and expectations, as
well as the [existing list] of feature gates.

[feature gate lifecycle]: https://git.k8s.io/community/contributors/devel/sig-architecture/feature-gates.md
[existing list]: https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/
-->

- [X] Feature gate (also fill in values in `kep.yaml`)
  - Feature gate name: `SidecarContainers`
  - Components depending on the feature gate:
    - kubelet
    - kube-apiserver
    - kube-controller-manager
    - kube-scheduler

###### Does enabling the feature change any default behavior?

<!--
Any change of default behavior may be surprising to users or break existing
automations, so be extremely careful here.
-->

No.

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

<!--
Describe the consequences on existing workloads (e.g., if this is a runtime
feature, can it break the existing applications?).

Feature gates are typically disabled by setting the flag to `false` and
restarting the component. No other changes should be necessary to disable the
feature.

NOTE: Also set `disable-supported` to `true` or `false` in `kep.yaml`.
-->

Yes. Pods that had sidecars will need to be deleted and recreated without them.

The feature gate disablement will require the kubelet restart. When kubelet will
start, it will fail to reconcile the Pod with the new fields and will clean up
all running containers.

If version downgrade is involved, the node must be drained. All Pods with the
new field will not be accepted by kubelet once feature was disabled.

###### What happens if we reenable the feature if it was previously rolled back?

If Pods were in Pending State rejected by kubelet due to "unknown" field to be
scheduled, they may become scheduleable again and will work as expected.

###### Are there any tests for feature enablement/disablement?

<!--
The e2e framework does not currently support enabling or disabling feature
gates. However, unit tests in each component dealing with managing data, created
with and without the feature, are necessary. At the very least, think about
conversion tests if API types are being modified.

Additionally, for features that are introducing a new API field, unit tests that
are exercising the `switch` of feature gate itself (what happens if I disable a
feature gate after having objects written with the new field) are also critical.
You can take a look at one potential example of such test in:
https://github.com/kubernetes/kubernetes/pull/97058/files#diff-7826f7adbc1996a05ab52e3f5f02429e94b68ce6bce0dc534d1be636154fded3R246-R282
-->

See https://github.com/kubernetes/kubernetes/pull/129731/ introducing this test with the emulated version.

### Rollout, Upgrade and Rollback Planning

<!--
This section must be completed when targeting beta to a release.
-->

###### How can a rollout or rollback fail? Can it impact already running workloads?

<!--
Try to be as paranoid as possible - e.g., what if some components will restart
mid-rollout?

Be sure to consider highly-available clusters, where, for example,
feature flags will be enabled on some API servers and not others during the
rollout. Similarly, consider large clusters and how enablement/disablement
will rollout across nodes.
-->

Rollout could fail for multiple reasons:

- webhooks that are not recompiled with the new field will strip it out
- bug in the resource calculation or CPU reservation logic could render the Pod unschedulable
- bug in the kubelet affecting the pod lifecycle could cause the Pod to be stuck in initialization

However, we have tried to maintain a high coverage of unit tests to ensure we catch these.

Rollback can fail if a Pod with sidecars is scheduled on a node where the feature
is disabled.
In that case the Pod will be rejected by kubelet and will be stuck in Pending state.
Therefore, we advise to first disable the feature gate on the control plane and then
proceed with the nodes.

Running workloads are not impacted.

Pods with sidecars might take a long time to exit and exceed the TGPS, a new
event should be added in beta to help administrators diagnose this issue.
Rather than rolling back the feature, they should work on the graceful termination
of their main containers to ensure sidecars have enough time to be notified
and exit on their own.

###### What specific metrics should inform a rollback?

<!--
What signals should users be paying attention to when the feature is young
that might indicate a serious problem?
-->

- [X] Metrics
  - Metric name: kubelet_started_containers_errors_total
    - Type: Counter
    - Labels:code, container_type (should be `init_container`)
    - Components exposing the metric: `kubelet-metrics`
    - Symptoms: high number of errors indicates that the kubelet is unable to start the sidecar containers
- [X] API objects
  - Pods stuck in Pending state of Init container running.
    - Type: API objects
    - Symptoms: when the new field `restartPolicy:Always` was mistakenly stripped out by a webhook, Pod will get stuck.

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

<!--
Describe manual testing that was done and the outcomes.
Longer term, we may want to require automated upgrade/rollback tests, but we
are missing a bunch of machinery and tooling and can't do that now.
-->

Upgrade->downgrade->upgrade testing was done manually using the following steps:

Kubelet specific:
1. Deploy k8s 1.29-alpha
2. Enable the `SidecarContainers` feature gate on the control plane and kubelet
3. Deploy a Pod with sidecar containers using a Deployment
4. Disable the `SidecarContainers` feature gate on the kubelet (requires a restart)
5. Drain the node
6. Pod is rejected by kubelet
7. Enable the `SidecarContainers` feature gate on the kubelet (requires a restart)
8. Pod is scheduled and works as expected

Control plane specific:
1. Deploy k8s 1.29-alpha
2. Enable the `SidecarContainers` feature gate on the control plane and kubelet
3. Deploy a Pod with sidecar containers using a Deployment
4. Disable the `SidecarContainers` feature gate on the control plane
5. Delete the Pod
6. Pod is created without the new field - init containers are not recognized as sidecars and block the Pod in initialization
7. Modify the Deployment by moving the sidecar containers to the regular containers section
8. Pod is scheduled and works (without the sidecar support)
9. Enable the `SidecarContainers` feature gate on the control plane
10. Delete the Pod
11. Pod is scheduled and works (without the sidecar support)
12. Modify the Deployment by moving the sidecar containers to the init containers section
13. Pod is scheduled and works (with the sidecar support)

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

<!--
Even if applying deprecation policies, they may still surprise some users.
-->

No.

### Monitoring Requirements

<!--
This section must be completed when targeting beta to a release.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.
-->

###### How can an operator determine if the feature is in use by workloads?

<!--
Ideally, this should be a metric. Operations against the Kubernetes API (e.g.,
checking if there are objects with field X set) may be a last resort. Avoid
logs or events for this purpose.
-->

By checking if `.spec.initContainers[i].restartPolicy` is set to `OnFailure` or `Always`.

###### How can someone using this feature know that it is working for their instance?

<!--
For instance, if this is a pod-related feature, it should be possible to determine if the feature is functioning properly
for each individual pod.
Pick one more of these and delete the rest.
Please describe all items visible to end users below with sufficient detail so that they can verify correct enablement
and operation of this feature.
Recall that end users cannot usually observe component logs or access metrics.
-->

End users can check components that are using the new feature, such as Istio, if istio-proxy runs as a sidecar container:

```
$ kubectl get pod -o "custom-columns="\
"NAME:.metadata.name,"\
"INIT:.spec.initContainers[*].name,"\
"CONTAINERS:.spec.containers[*].name"

NAME                     INIT                     CONTAINERS
sleep-7656cf8794-8fhdk   istio-init,istio-proxy   sleep
```

###### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

<!--
This is your opportunity to define what "normal" quality of service looks like
for a feature.

It's impossible to provide comprehensive guidance, but at the very
high level (needs more precise definitions) those may be things like:
  - per-day percentage of API calls finishing with 5XX errors <= 1%
  - 99% percentile over day of absolute value from (job creation time minus expected
    job creation time) for cron job <= 10%
  - 99.9% of /health requests per day finish with 200 code

These goals will help you determine what you need to measure (SLIs) in the next
question.
-->

- number of running containers should not change by more than 10% throughout the day,
  as measured by the number of running containers at the beginning and end of the day
- error rate for containers of type init_container should be less than 1%,
  as measured by the number of errors divided by the total number of init_container containers
- number of events indicating that TGPS has been exceeded should be less than 10 per day,
  as measured by the number of events logged in the kubelet log
- 99% of jobs with sidecars should complete successfully,
  as measured by the number of jobs that complete successfully divided by the total number of jobs with sidecars

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

<!--
Pick one more of these and delete the rest.
-->

- [X] Metrics
  - Metric name: kubelet_running_containers
    - Type: Gauge 
    - Labels:container_state
    - Components exposing the metric: `kubelet-metrics`
  - Metric name: kubelet_started_containers_errors_total
    - Type: Counter 
    - Labels:code, container_type (should be `init_container`)
    - Components exposing the metric: `kubelet-metrics`

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

<!--
Describe the metrics themselves and the reasons why they weren't added (e.g., cost,
implementation difficulties, etc.).
-->

No.

### Dependencies

<!--
This section must be completed when targeting beta to a release.
-->

###### Does this feature depend on any specific services running in the cluster?

<!--
Think about both cluster-level services (e.g. metrics-server) as well
as node-level agents (e.g. specific version of CRI). Focus on external or
optional services that are needed. For example, if this feature depends on
a cloud provider API, or upon an external software-defined storage or network
control plane.

For each of these, fill in the following—thinking about running existing user workloads
and creating new ones, as well as about cluster-level services (e.g. DNS):
  - [Dependency name]
    - Usage description:
      - Impact of its outage on the feature:
      - Impact of its degraded performance or high-error rates on the feature:
-->

No.

### Scalability

<!--
For alpha, this section is encouraged: reviewers should consider these questions
and attempt to answer them.

For beta, this section is required: reviewers must answer these questions.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.
-->

###### Will enabling / using this feature result in any new API calls?

<!--
Describe them, providing:
  - API call type (e.g. PATCH pods)
  - estimated throughput
  - originating component(s) (e.g. Kubelet, Feature-X-controller)
Focusing mostly on:
  - components listing and/or watching resources they didn't before
  - API calls that may be triggered by changes of some Kubernetes resources
    (e.g. update of object X triggers new updates of object Y)
  - periodic API calls to reconcile state (e.g. periodic fetching state,
    heartbeats, leader election, etc.)
-->

No.

###### Will enabling / using this feature result in introducing new API types?

<!--
Describe them, providing:
  - API type
  - Supported number of objects per cluster
  - Supported number of objects per namespace (for namespace-scoped objects)
-->

No.

###### Will enabling / using this feature result in any new calls to the cloud provider?

<!--
Describe them, providing:
  - Which API(s):
  - Estimated increase:
-->

No.

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

<!--
Describe them, providing:
  - API type(s):
  - Estimated increase in size: (e.g., new annotation of size 32B)
  - Estimated amount of new objects: (e.g., new Object X for every existing Pod)
-->

No.

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

<!--
Look at the [existing SLIs/SLOs].

Think about adding additional work or introducing new steps in between
(e.g. need to do X to start a container), etc. Please describe the details.

[existing SLIs/SLOs]: https://git.k8s.io/community/sig-scalability/slos/slos.md#kubernetes-slisslos
-->

Graceful Pod termination might take longer with sidecars since their exit sequence starts after the
last main container has stopped.
The impact should be negligible because the TGPS is enforced in all cases.

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

<!--
Things to keep in mind include: additional in-memory state, additional
non-trivial computations, excessive access to disks (including increased log
volume), significant amount of data sent and/or received over network, etc.
This through this both in small and large cases, again with respect to the
[supported limits].

[supported limits]: https://git.k8s.io/community//sig-scalability/configs-and-limits/thresholds.md
-->

No.

###### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?

<!--
Focus not just on happy cases, but primarily on more pathological cases
(e.g. probes taking a minute instead of milliseconds, failed pods consuming resources, etc.).
If any of the resources can be exhausted, how this is mitigated with the existing limits
(e.g. pods per node) or new limits added by this KEP?

Are there any tests that were run/should be run to understand performance characteristics better
and validate the declared limits?
-->

No, since the KEP only enable a new way to run containers as sidecars instead of regular containers.
Resource consumption can even be lower since various tricks using emptyDir volumes to perform synchronization
(as with istio-proxy) are no longer needed.

### Troubleshooting

<!--
This section must be completed when targeting beta to a release.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.

The Troubleshooting section currently serves the `Playbook` role. We may consider
splitting it into a dedicated `Playbook` document (potentially with some monitoring
details). For now, we leave it here.
-->

###### How does this feature react if the API server and/or etcd is unavailable?

Nothing changes compared to the current kubelet behavior.

###### What are other known failure modes?

<!--
For each of them, fill in the following information by copying the below template:
  - [Failure mode brief description]
    - Detection: How can it be detected via metrics? Stated another way:
      how can an operator troubleshoot without logging into a master or worker node?
    - Mitigations: What can be done to stop the bleeding, especially for already
      running user workloads?
    - Diagnostics: What are the useful log messages and their required logging
      levels that could help debug the issue?
      Not required until feature graduated to beta.
    - Testing: Are there any tests for failure mode? If not, describe why.
-->

- Main containers don't exit within TGPS, leading to sidecars being terminated
  - Detection: high number of events indicating TGPS has been exceeded
  - Mitigations: ensure timely termination of main containers
  - Diagnostics: Events
  - Testing: https://github.com/kubernetes/kubernetes/blob/b4f902f0371485505ff4eda39975e67bfa9b0727/test/e2e_node/container_lifecycle_test.go#L4977-L5077
- Main container or sidecar use a preStop hook consuming TGPS, leading to remaining sidecars being terminated
  - Detection: high number of events indicating TGPS has been exceeded
  - Mitigations: ensure preStop hooks are not delaying termination
  - Diagnostics: Events
  - Testing: https://github.com/kubernetes/kubernetes/blob/b4f902f0371485505ff4eda39975e67bfa9b0727/test/e2e_node/container_lifecycle_test.go#L4272-L4408
- Sidecar container uses a preStop hook that make the container exit during Pod shutdown, sidecar is restarted, leading
to a CrashLoopBackOff
  - Detection: sidecar in CrashLoopBackOff during termination
  - Mitigations: ensure preStop hooks are not making the container to exit, document best practices
  - Diagnostics: Events
  - Testing: no testing needed as this is a best practice implementing sidecars

###### What steps should be taken if SLOs are not being met to determine the problem?

None.

## Implementation History

<!--
Major milestones in the lifecycle of a KEP should be tracked in this section.
Major milestones might include:
- the `Summary` and `Motivation` sections being merged, signaling SIG acceptance
- the `Proposal` section being merged, signaling agreement on a proposed design
- the date implementation started
- the first Kubernetes release where an initial version of the KEP was available
- the version of Kubernetes where the KEP graduated to general availability
- when the KEP was retired or superseded
-->

- 2018-05-14: First proposal.
- 2023-06-09: Target 1.28 for Alpha.
- 2023-07-08: Alpha implementation merged.
- 1.29: feature is in Beta
- 1.33: feature is graduated to Stable

## Drawbacks

<!--
Why should this KEP _not_ be implemented?
-->

## Future use of restartPolicy field

In the Alpha version, no other values beyond Always will be acceptable. Going
forward the following semantic may be supported for other values of
restartPolicy field of init containers. The `restartPolicy` may be set to one of:
`Never`, `OnFailure` and `Always`.  The meaning of these values is similar to
the Pod 'restartPolicy' flag:
- `Never` indicates that if the container returns a non-zero exit code, that pod
  initialization has failed. This value can be supported for Pods with Never and
  OnFailure `restartPolicy` indicating the critical Initialization step.
  Supporting this value on Pods with the the `restartPolicy` `Always` is not
  planned as it changes the semantic of this policy and may affect how
  deployments control the # of Pods in this deployment.
- `OnFailure` indicates the init container will be restarted until it completes
  successfully. This value may be used for Pods with the `restartPolicy` `Never` ,
  indicating that the Initialization step is flaky and should be retried.
- `Always` indicates that the container will be considered initialized the
  moment the startup probe indicates it is started and will be restarted if it
  exits for any reason until the pod terminates.
- (If the initContainer's `restartPolicy` is unset, the pod's `restartPolicy` is
  used to determine the init container restart policy. For Always, OnFailure is
  used, for OnFailure - OnFailure, for Never - Never).

Since init container and regular containers share the schema, `restartPolicy`
field will be available for regular containers as well. From scenarios
perspective, the first set of scenarios would be for containers that can only
override the `restartPolicy` to “higher” value. From `Never` to `OnFailure` or `Always`,
from `OnFailure` to `Always`. An override from `Always` to `Never` will not be
allowed.This will eliminate situations when kubelet restarts and cannot decide
whether the container overridden value to `Never` will need to be restarted or not
for the Pod with the restart policy `Always`. It is also likely that we will
require at least one container to have a `restartPolicy` matching the Pod
`restartPolicy`. We can re-evaluate this in future if scenarios will arise.

Note, by implementing these scenarios, the meaning of `restartPolicy:Always` for
containers on a Pod with the restartPolicy:Never, will not exactly be the
meaning of the `restartPolicy:Always` for the Pod with the `restartPolicy:Always`.
For the first case, containers will not block the Pod termination. For the
second, they will. In other words, the container with `restartPolicy:Always` will
effectively become a sidecar container similar to init container with the same
field value.

Additional considerations can be found in the older document that was a previous
attempt to implement the sidecar containers:
https://docs.google.com/document/d/1gX_SOZXCNMcIe9d8CkekiT0GU9htH_my8OB7HSm3fM8/edit#heading=h.71o8fcesvgba.

## Alternatives

<!--
What other approaches did you consider, and why did you rule them out? These do
not need to be as detailed as the proposal, but should include enough
information to express the idea and why it was not acceptable.
-->

### Pod startup completed condition

As part of this KEP, we needed to decide on a condition that Pod needs to meet to
be recognized as "started". This will be used to sequence init containers
properly and give enough time for sidecar container to complete it's
initialization.

#### Readiness probes

The easiest signal we have today is a Container Ready state. Semantically this signal means that the container is fully prepared to handle traffic.

The risk of using of Ready state as a startup completed condition, is:
- today the ready state is used to control traffic, but not for container
  lifecycle (as oppose to startup and liveness probes). If Container's readiness
  probe depends on external services, and those services are not yet available
  or depend on other containers in the Pod, the sidecar container will never be
  started.
- it may be harder semantically to understand what does the not Ready Sidecar
  container mean if it became not ready after it has started, while other Init
  containers are being run.
- the scenario when sidecar container is started, but wants to delay the
  readiness for later would not be possible to implement.

#### Startup probes

The second possible signal we can use is Startup probes. Kubelet will be waiting
for startup probe to succeed to mark the container as started. Readiness probes
will only start when Pod's initialization stage will complete and regular
containers will start. There is a `ContainerStatus` field `Started` indicating
this state.

#### `postStart` hook

The easiest approach would be to simply wait for postStart hook to complete.
This is already implemented for regular containers - they are started
sequentially with the `postStart` hook being a blocking call.

Using `postStart` has some debugging challenges in how logs are reported (they
aren't) and the status is displayed. If this option will be used, we should
consider improving this situation.

With `postStart` simple sidecars will start faster than if any probes are used.
However it brings additional challenges for sidecar authors as majority of logic
may be duplicated and needs to be authored, e.g. a shell binary + a loop script
to retry `curl`.

### Alternative 1. Sidecar containers as a separate collection

This alternative was rejected for the following reasons:
- it introduces a whole new collection of containers, which is a big change
doesn’t allow different startup sequencing - before or after init containers.
Before or after a specific init container. Making a single decision for the
entire collection of containers and all scenarios was not possible.
- A single semantic for the entire collection of containers may make it harder
  to introduce new properties to containers in this collection without changing
  its semantic. It was preferable to introduce new properties representing
  specific behaviors to the existing containers
- Each time a new collection of containers is added to Pod, it increases the
  size of the OpenAPI schema of pods and all resources containing PodTemplate
  significantly, which has negative downstream implications on systems consuming
  OpenAPI schemas and on CRDs that embed PodTemplate.

### Alternative 2. DependOn semantic between containers

This alternative is trying to replicate systemd dependencies. The alternative
was rejected for the following reasons:
- Universal sidecars injection is not possible for such a system.
- There were lack of scenarios requiring more than general containers sequencing
  and two stages - Init and Main execution.

### Alternative 3. Phases

This alternative introduces semantically meaningful phases like Network
unavailable, Network configured, Storage configured, etc. and fit containers in
these phases. This alternative was rejected because it was hard to impossible to
fit all containers and scenarios into predefined buckets like this. And the
opposite was also true - many scenarios didn’t need this level of complexity and
separation.

### Alternative 4. TerminatePod on container completion

This alternative (https://github.com/kubernetes/enhancements/issues/3582) was
one of the attempts to scope a problem to the absolute minimum needed to solve
Jobs with sidecars termination problems. The idea is to implement a specific
property on a container that will terminate the whole Pod when a specific
container exits. This idea was rejected:
- Automatic injection is hard
- Only a subset of scenarios is being solved and introduction of built-in
  sidecar containers support will disregard most of the work
- There is a desire to allow to distinguish sidecar containers from other
  containers

### Alternative 5. Injection of sidecar containers thru the "external" object

The idea of this alternative is to allow configuring some containers for the Pod
using another object, so kubelet will combine a resulting Pod definition as a
multi-part object. This alternative offers great properties of security
boundaries between objects allowing different permissions sets to different
containers. 

Scenario this and similar alternative may cover are:

> More effectively separate access to privileged / secure resources between
> containers. Filesystem resources, secrets, or more privileged process
> behaviors can be separated into a container that has higher access and is
> available before the regular serving/job container and until it has completed.

This is somewhat part of the service mesh use case (especially those that
perform privileged iptables manipulation and hide access to secrets).

Other scenarios:

- Run an agent that exchanges pod identity information to gain a time or space
  limited security credential (such as an SSL cert, remote token, whatever)
  alongside a web serving container that has no access to the pod identity
  credential, and thus can only expose the bound credential if compromised.
- Start a FUSE daemon in a sidecar that mounts an object storage bucket into one
  of the shared pod emptydir volumes - have another container running with much
  lower privileges that can use that
- Run a container image build with privileges in a sidecar container that
  responds to commands on a unix socket shared in a pod emptydir and takes
  commands from an unprivileged container to snapshot the state of the
  container.

However this alternative was rejected as the change introduced is beyond the
scope of the sidecar KEP and will be needed for many more scenarios.
Implementing sidecar containers as proposed in this KEP does not make this
alternative harder to implement in future.

A little more context can be found in this document:
https://docs.google.com/document/d/10xqIf19HPPUa58GXtKGO7D1LGX3t4wfmqiJUN4PJfSA/edit#heading=h.xdsy4mbwha7q 


## Infrastructure Needed (Optional)

<!--
Use this section if you need things from the project/SIG. Examples include a
new subproject, repos requested, or GitHub details. Listing these here allows a
SIG to get the process for these resources started right away.
-->
