# Ansible Provisioning Appliance

## Overview

This is a natural development of the Red Hat [Validated Patterns](https://validatedpatterns.io) initiative,
which we have communicated as reference architectures that can be run through CI/CD, and thus must have
real software deliverables.

The essence of this idea is to provide a stable and reliable platform for instantiating a larger
infrastructure that can be used in multiple cloud scenarios (that is, hyperscalers and bare metal).

Design goals for what this appliance framework should do:
* Be able to inject this "appliance" into an environment (i.e. hyperscaler availability zone, or an on-prem data center) and be able, from this appliance, to execute a GitOps workflow for provisioning other workloads.
* When the provisioning is complete, it should be possible to archive the logs from doing the provisioning work as a single artifact for future reference or troubleshooting.
* Once provisioning is complete and logs have been retrieved, it should be possible to decommission the appliance and
remove it from the network environment. (That is - the appliance is not essential to the ongoing maintenance of the
environment that it provisions and in fact should provide suitable solutions for the ongoing maintenance of provisioned
environments as part of the provisioning process.)

Non-functional design goals for how this framework should work:
* It should support as many hosting environments as possible. Initially, it should support hyperscalers (i.e. AWS) and bare metal. It should have line of sight to supporting disconnected networks, but that may not be a day-1 deliverable. Support of a particular hosting environment should not be predicated on the appliance being hosted in that environment directly. (Thus, it should be possible to build the appliance on baremetal and have it provision infrastructure in AWS.)
* Rather than being a solution for managing a particular provisioning workflow, it should strive to enable arbitrary
provisioning workflows, and be expandable to support tooling and workflows beyond what has initially been considered.
* Scalability of the provisioning system itself is secondary to architectural flexibility. That is, scaling the provisioning system vertically (i.e. by adding more processing, memory, storage) is preferable to the complexity that would be incurred by scaling the system horizontally.
* The value of this kind of systems lies in the aggregation and integration of functionality rather than in the
creation of new installers for existing products.

The offering should focus on being the container and execution environment for defining and building
larger components of infrastructure in a repeatable mechanism. The offering must provide tooling suitable
for bootstrapping the appliance environment, and then the workload that runs on the appliance environment
should be a black box.

## Workflow

The "process" here will be a guided, opinionated workflow that is self-contained (to the extent that it
includes references to all of the code and artifacts that it needs to function and run to completion;
such that it can be tested. In a very real sense, it is an extended Validated Pattern that has both
OpenShift and Ansible (and possibly other) components. A critical difference, though, is that one should
always think of this Appliance solution as being inherently temporary - while the user may wish to keep it
after its job is done, its role is not to serve as an essential infrastructure component in perpetuity;
rather its role is to provide a platform to run repeatable, GitOps workflows for populating and building
environments.

The main workflow should look like this:

1. User kicks off the process with suitable parameters for configuration
2. Process creates Appliance
3. Process runs Pattern(s) on Appliance
* The OpenShift components are managed as an OpenShift Validated Pattern hosted on the cluster
* The Ansible components are managed as Ansible Validated Patterns and are hosted on the AAP instance
inside the cluster.
4. (Optional) Process provides mechanism to bundle and download logs
5. (Optional) Process provides mechanism to uninstall itself

## Solution Overview

The solution will consist of a single SNO node, which will be running at minimum:

* OpenShift GitOps (ArgoCD)
* Ansible Automation Platform
* A Git Hosting Solution (probably Gitea)
* HashiCorp Vault
* External Secrets Operator

Other operators may be included, and the architecture is flexible to allow other additions. Other
components that may be useful to include would be Red Hat Advanced Cluster Manager (for provisioning
additional OpenShift clusters), OpenShift Virtualization (for hosting VMs, if needed/desired), and so on.
These can be accomodated by running on a larger single node, either in a hyperscaler or on bare metal.

The architecture envisions the usage of this sort of appliance in a disconnected environment as well, and
provides for the mechanisms to include the content needed to service such an environment in a disconnected
fashion.

## Solution Essential Components

### OpenShift (specifically Single Node OpenShift)

OpenShift provides a useful, and extensible, set of platform primitives that meet the need to make a process
repeatable and still open-ended enough to be extended should needs arise that have not been previously considered.

Single Node OpenShift is specified here for these reasons:

* It is simpler and easier to install initially than other kinds of OpenShift clusters, and supports the
entire range a user may desire from the OpenShift ecosystem (that is, operators and other platform capabilities).
* It is considerably cheaper to run a single node than multiples in hyperscaler environments
* Specifying SNO at the outset emphasizes the temporary nature of the provisioning appliance.

Meanwhile, OpenShift provides a fully supported and supportable, reasonably future-proof way of hosting various
kinds of application workloads, including especially the infrastructure needed to provision other infrastructure,
and with proper prior planning, can reasonably expand to include much more exotic and complex scenarios. Need to
run some provisioning services on virtual machines for some reason? Plan to include OpenShift Virtualization.

OpenShift provides mechanisms to install on Bare Metal and common hyperscalers, and even includes mechanisms to
install on very exotic and custom setups. That said, given the other dynamics in play (and particularly the desire
to run on bare metal, and the amount of compute that a single bare metal node now represents), limiting the
architecture initially to SNO seems reasonable.

It is certainly possible to run all of these and other services on a very large single RHEL node, but OpenShift
provides the necessary integrations into DNS and other cloud services, and the ability to manage DNS and load
balancing internally, that would have to be worked out in a purely RHEL environment. That said, we recognize that
some services can/should only run on RHEL and suggest OpenShift Virtualization as a mechanism to run those should
they be needed.

### OpenShift GitOps (ArgoCD)

ArgoCD is essential because it is the common denominator of repeatability for the Validated Patterns initiative.
Because of the Validated Patterns effort, there is considerable prior art available to provide installation schemes
for most of the named components. The GitOps operator also provides a convenient mechanism for extending the
installation if needed.

### Ansible Automation Platform

AAP is essential as the glue and primary engine for declaring desired state on anything that is not kubernetes-native
infrastructure, including (but not necessarily limited to) cloud APIs, network gear (switches and routers). AAP also
includes the primitives for hosting its own content in this scenario.

### A Git Hosting Platform (Gitea, possibly GitLab or other)

OpenShift GitOps will require a git service, and it is very likely that Ansible Automation Platform will want to use
one as well. Providing git hosting inside the appliance reduces outside dependencies (such as having to support the
different APIs in GitHub versus GitLab versus Gitea, and crucially allows for local customization of a public pattern
without requiring the user to fork the pattern.

Architecturally, this includes the other components that would be needed for disconnected use, exluding that which
would be needed for Ansible specifically because AAP's Automation Hub component provides those. Automation Hub may be
useful in this capacity, for example as a container registry, as well.

## Technically Non-Essential Components

These components are not essential in the sense that they can be swapped out with some effort, but we expect the
benefits of having them will in most cases outweigh the desire to not have them.

### HashiCorp Vault (Community)

Vault becomes the standard secret store for both the components that run in OpenShift and the workflows in Ansible.
The OpenShift integrations can be done through the External Secrets Operator, while the integration with AAP is direct.
This allows the use of the same secret material in both OpenShift and Ansible pattern components.

### External Secrets Operator

The External Secrets Operator is used to project secrets from Vault into OpenShift applications where they are needed.

## Optional Components

### Red Hat Advanced Cluster Management

RHACM has a number of capabilities for deploying and managing OpenShift clusters.
The primary usage of RHACM in this context is expected to be the creation of clusters,

### OpenShift Virtualization

OCP Virtualization is most useful if the patterns included require VMs to run as part of the provisioning workload.
For example, if the user wants to run Satellite as part of the provisioning process (i.e. building a Satellite, then
using that Satellite to build other RHEL nodes as part of the "production" workload), this would be the way to enable
that. It would be architecturally defensible to provision additional VMs as part of the provisioning environment in a
hyperscaler, but to keep the process the same in as many environments as possible, the architecture specifies the use
of OpenShift Virtualization, as it works in both hyperscalers and on bare metal.

### Red Hat Satellite

Satellite, for example, may be useful to have to provision additional RHEL nodes. It has long-established capabilities
for assisting with bare metal provisioning (including DHCP/PXE) as well, which may be needed in some environments.

### Red Hat Identity Manager (IdM)

Red Hat Identity Manager provides integrated DNS, user management, and certificate management. It is more likely that
IdM would be part of the infrastructure that the appliance is being used to provision, rather than as part of the
appliance itself.

## Initial Targets

### Bare Metal

Bare metal installation is a target for the first phase of this effort because it is something that we have seen
customers struggle with. We simplify the process as much as possibly by specifying that the provisioner will be
Single Node OpenShift, which eliminates much of the complexity. Bare metal installation also seems essential to
consider for the disconnected network case.

In general, the tooling to install on bare metal should be either openshift-install or the assisted installer.

### AWS

AWS installation should be the other first-phase installation target. This should help establish the general shape
of hyperscaler installation, and many (while not all) of the lessons we learn there will help us move to, for example,
Azure, GCP, Alibaba, and IBM Cloud (given enough success).

## Feasible Future Directions

### Azure

The next hyperscaler we tackle after AWS should be Azure.

### Terraform

Since this idea originated in the idea of building public cloud zones, users may very well want to use
Terraform to build some of those elements. With the (at time of writing) pending acquisition of HashiCorp
by IBM, there are a number of intriguing possibilities for interoperability among Ansible, Terraform, and
OpenShift. As the parameters of the acquisition become clearer, this framework would be a good way and
vehicle for exploring those possibilities.
