# Bedrock Developer Experience "North Star"

A scenario based description of how someone might use all of the tools and components in Bedrock in the glorous future to easily define, build, deploy, and maintain workload in Kubernetes.  Eventually, the goal is to turn this document into a walk through of the platform in our documentation, and a video of how the operational process we have defined works end to end.

THERE ARE NO SACRED COWS IN THIS DOCUMENT.  In fact, quite the opposite, I suspect there are better ideas out there for many parts of this experience.  This document is meant to stimulate that discussion, tease out the details, and describe the best end to end coherent developer/ops experience we can.

## Creating First Cluster

Oddveig, a person in an operations role, has heard about Bedrock from others in her company using it and would like to try it out.

She heads over to the Bedrock repo, and after reading the overview description, tries the Quickstart with Azure.

The Quickstart first asks her to install the `spk` tool:
```bash
$ ???
```

She has a new development machine with this role, so she uses `spk` to install all of the prerequisites:

```bash
$ spk prereqs install
```

This installs all of the prereqs on her machine that she needs.

The next step in the quickstep is to create her own local cluster environment definition based off of the `azure-simple` template, so she first creates a directory for that environment:

```bash
$ mkdir my-azure-simple
$ cd my-azure-simple
```

And then uses `spk` to scaffold an environment  based off of the `azure-simple` template.

```bash
$ spk infra scaffold https://github.com/microsoft/bedrock/azure-simple
```

This creates a `infra.json` file with the following:

```js
{
	name: 'my-azure-simple',
    source: 'https://github.com/microsoft/bedrock/tree/master/cluster/environments/azure-simple',
    version: 'd7d905e6551',
    variables: {
		resource_group_name: '<resource-group-name>',
        cluster_name: '<cluster-name>',
        agent_vm_count: 3,
        service_principal_id: '<client-id>',
 		service_principal_secret: '<client-secret>',
        ssh_public_key: "public-key"
        gitops_ssh_url: "git@github.com:timfpark/fabrikate-cloud-native-manifests.git"
	    gitops_ssh_key: "<path to private gitops repo key>"
		vnet_name: "<vnet name>"
    }
}
```

She fills in all of the variables for her particular cluster and then generates the environment:

```bash
$ spk infra generate
```

This creates a Terraform template from the base template at the specified version from the definition in the current directory.  She can later bump the version and regenerate the Terraform template.   (In some workflows, this could also be used in a CI/CD orchestration to generate the Terraform template that is then applied.)

One implication of this is that the Terraform code is generated and therefore should be regarded as immutable output.

She’s happy with the generated Terraform code, so she inits and applies it with Terraform:

```bash
$ terraform init
$ terraform apply
```

The cluster spins up and deploys the stock workload manifests in the repo  `gitops_ssh_url`

## Creating first manifest repo

After trying out the ‘hello-world’ workload that was provided as part of the template, she wants to use her own workload.

She creates a new manifest repo in Github, adds the gitops ssh key to the repo, and then adjusts `gitops_ssh_url` to `git@github.com:oddveig/cluster-resource-manifests.git`.

She then regenerates the Terraform templates with `spk`:

```bash
$ spk infra generate
```

And recreates the cluster with:

```bash
$ terraform destroy
$ terraform apply
```

The cluster now uses her repo instead and she can check in Kubernetes manifests to it and have them deployed to the cluster.

## Building a first Fabrikate definition

She does this for a bit, and quickly realizes that committing YAML directly is difficult and error prone and that she wants to use Helm to template these deployments.

Furthermore, she wants to be able to compose the config for these deployments eventually across a number of different clusters and also utilize a preconfigured stack component that her company maintains and so decides to build a high level definition with Fabrikate for it.

To do this, she first creates a directory for her high level definition project:

```bash
$ mkdir my-cluster-hld
$ cd my-cluster-hld
```

She then uses fabrikate to add the common stack her company uses across Kubenetes deployments:

```bash
$ fab add cloud-native --source https://github.com/microsoft/fabrikate-definitions --path definitions/fabrikate-cloud-native
```

this creates a `component.yaml` file that is the root  of her Fabikate definition that looks like this:

```yaml
name: my-cluster-hld
subcomponents:
- name: cloud-native
  type: component
  source: https://github.com/microsoft/fabrikate-definitions
  method: git
  path: definitions/fabrikate-cloud-native
  branch: master
```

which she checks into `github.com/oddveig/cluster-fabrikate-defintion`

She wants to generate the resource manifests for her repo above using this Fabrikate definition so she scaffolds out a Gitops pipeline:

```bash
$ spk gitops scaffold
```

This creates a gitops pipeline definition that looks like this:

```yaml
orchestrator: azure-devops
repos:
  fabrikateDefinition:
  resourceManifests:
```

which she edits to include the relevant and necessary details:

```yaml
orchestrator: azure-devops
repos:
  fabrikateDefinition: git@github.com:oddveig/cluster-fabrikate-definition.git
  resourceManifests: git@github.com:oddveig/cluster-resource-manifests.git
```

She then creates the gitops pipeline with:

```bash
$ spk gitops create
```

This creates the repos (if necessary) and the Azure Devops pipeline to automatically build the Fabrikate definition in `github.com/oddveig/cluster-fabrikate-definition` into resource manifests that are stored in `github.com/oddveig/cluster-resource-manifests`.

## Adding a Service

With her cluster created and her GitOps pipeline operational, she tells Dana, a developer in the team.

Dana is excited to hear the cluster is in place and wants to deploy her `search` microservice into it.  Her team uses the rings service maturity model and three rings: `dev`, `qa`, and `prod`.

She starts by creating the service itself.  She checks out the Fabrikate definition that Oddveig established and creates a development branch called `search_service`.

She then scaffolds out a new service with:

```bash
??? $ spk service scaffold search-service  ???
```

TODO: How do we tie the service repo into this?
TODO: How does the pipeline get deployed for service repo -> Fabrikate definition?

## Adding a Service Revision Ring

With the service created, she next wants to add the rings for the service.  She starts with the `dev` ring:

```bash
??? $ spk ring scaffold search-service dev ???
```

She has now created the `search` service with three rings as a set of changes in the Fabrikate deployment.  As part of the development and ops team's workflow, they always use pull requests and code reviews of these changes.

She creates a PR for the changes and asks Oddveig to review.  Oddveig approves the PR, merges it to master.

## Observing Deployments

The merge to master that Oddveig makes kicks off the GitOps pipeline that we built earlier.  In the absence of tooling, this GitOps pipeline is observable, but only through manual navigation to various Azure Devops pages and manually collecting logs from Flux in the cluster.

Instead Dana wants to observe the process in her command line and she uses the `spk` command line tool to do this:

```bash
$ spk service get --service search-service --watch  ???
```

which displays a new log line in her console as each step is completed that looks like:

???
