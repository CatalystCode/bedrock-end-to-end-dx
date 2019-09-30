# Bedrock Developer and Operations Experience "North Star"

A scenario based description how all the tools and components in Bedrock fit together to easily define, build, deploy, and maintain a workload running in a Kubernetes cluster.

THERE ARE NO SACRED COWS IN THIS DOCUMENT. Please point out misunderstandings, problems, and places that the experience could be better.

## Getting Started

Olina, who is in an operations role at a company called Fabrikam, has heard about Bedrock from others in her company and would like to use it on a project he is leading to launch a microservice workload called `discovery-service`.

He first installs the `spk` tool, which provides helpful automation around defining and operating Kubernetes clusters with Bedrock  principles:

TODO: Confirm this is how we are going to install `spk`

```bash
$ wget https://github.com/microsoft/spektate/releases/download/1.0.1/spk-v1.0.1-darwin-amd64.zip
$ unzip spk-v1.0.1-darwin-amd64.zip
$ mv spk ~/bin (or as appropriate to place it in your path)
```

## Initializing Infrastructure

With `spk` installed, she initializes the `spk` tool with:

```bash
$ spk init
```

This creates a `~/.spk/config` file with auth details, validates that prereqs are installed (`git`, `terraform`, `helm`, `az` cli tools), inventories their versions, and validates that they are compatible with `spk`. The tool notices that she does not have a compatible version of `helm` and asks her to update it, which she does with her favorite package manager manually, and then reruns `spk init`.

## Creating Cluster Definition

With her `spk` tool initialized, she moves on to creating her infrastructure deployment project.  She first scaffolds the project with `spk infra scaffold`:

```bash
$ spk infra scaffold discovery-cluster-infra --bedrock-source https://github.com/fabrikam/bedrock –-container-name discovery-cluster –-backend-key <key>
```

This creates a `discovery-cluster-infra.json` file with a locked source at the latest version (such that it does not change underneath the infrastructure team) and a prefilled set of configuration variables with defaults (if applicable).

```js
{​
    name: 'discovery-cluster-infra',
    source: 'https://github.com/fabrikam/bedrock/tree/master/cluster/environments/fabrikam-azure-single-keyvault',
    version: 'd7d905e6551',

    resources: [​
        resource_group_name, vnet_name​
    ],
    ​
    variables: {​
        resource_group_name: '<resource-group-name>',​
        cluster_name: '<cluster-name>',​
        agent_vm_count: 3,​
        service_principal_id: '<client-id>',
        service_principal_secret: '<client-secret>',​
        ssh_public_key: "public-key"​
        gitops_ssh_url: "git@github.com:timfpark/fabrikate-cloud-native-manifests.git"​
        gitops_ssh_key: "<path to private gitops repo key>"​
        vnet_name: "<vnet name>"​
    }​
}
```

TODO: Should service principal details be in this file? (probably not, since it will be checked in)

She fills in all of the variables for her particular cluster and then generates the environment:

```bash
$ spk infra generate discovery-cluster-infra
```

This creates a Terraform template from the base template at the specified version from the definition in the current directory.  She can later bump the version and regenerate the Terraform template, and when in the future `discovery-service` grows to be a service that her company deploys in multiple clusters in multiple regions, she can also use this to easily stamp out multiple clusters with largely common, but when necessary, differentiated, config.

## Scaffolding GitOps Pipeline

In the bedrock GitOps deployment methodology, code commits in the application repo trigger building a container, which in turn triggers a release step that commits the new image tag for this container to a Fabrikate high level definition repo.  This commit to the high level definition triggers a build that generates and commits the resource manifests into a repo that the cluster then uses as its store of record for what should be deployed into the cluster.

TODO: Add diagram to make this more concrete.

Setting all of this up by hand is hard, error prone, and tedious.  Instead, she wants to generate the resource manifests for her repo above using this Fabrikate definition so she scaffolds out a Gitops pipeline definition:

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

which she edits to include the relevant and necessary details of the repos she would like to create.

```yaml
orchestrator: azure-devops
repos:
  fabrikateDefinition: git@github.com:fabrikam/discovery-cluster-definition.git
    fromTemplate: git@github.com:fabrikam/default-cluster-definition.git
  resourceManifests: git@github.com:fabrikam/discovery-cluster-manifests.git
```

She then creates the gitops pipeline with:

```bash
$ spk gitops create
```

This creates the repos (if they do not currently exist) and the Azure Devops pipeline to automatically build the Fabrikate definition in `github.com/oddveig/cluster-fabrikate-definition` into resource manifests that are committed in `github.com/oddveig/cluster-resource-manifests`. It also instruments those pipelines such that they can be introspected to understand the current state of the deployment.

## Building High Level Definition for Workload

This automation creates the plumbing for building and deploying a cluster but she still needs to define what workload should be deployed in these clusters.

To do this, she first clones the high level definition repo that the automation created above:

```bash
$ git clone https://github.com/fabrikam/discovery-cluster-definition
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

which she then commits back to the repo. This triggers the generation process and this deployment definition will be built into resource manifests that are committed to `github.com:fabrikam/discovery-cluster-manifests`.

## Adopting Bedrock in Existing Application Monorepo

With her cluster definition created, her GitOps pipeline operational, and a base deployment definition in place, she tells Dag, a developer in the team that he can start adding the `discovery-service` services to the deployment.

As we mentioned at the outset, `discovery-service` is a service that is already in deployment in a virtualized environment and has been developed in a [monorepo](https://en.wikipedia.org/wiki/Monorepo) style. As we mentioned, Dag wants to use Bedrock to deploy these microservices, so he navigates to the root of this monorepo that he has cloned on his machine:

```bash
$ cd discovery-service
```

and then uses `spk` to initialize it:

```bash
$ spk project init -m
```

where `-m` indicates to `spk` that this a monorepo for multiple microservices.

TODO: What files does this define and what are their purposes?

## Defining Rings for Services

The `discovery-service` team also uses the Rings service deployment model with three rings: `dev`, `qa`, and `prod`. In this deployment model, requests are routed to specific instances of a deployment based on specific group membership that is attached to the request. In the case of Fabrikam and the `discovery-service`, this group membership is applied to requests as part of a header that is then routed into the cluster.

TODO: How do we see the definition of rings for services in the long term?

## Adding a Service

She starts by creating the service itself. She checks out the Fabrikate definition that Olina established and creates a development branch called `add_proxy_service` to add `proxy-service`, an individual microservice that composes in part the overall `discovery-service`

She then initializes this new service with:

TODO: How does this work?
```bash
$ spk service create proxy-service
```

This creates a directory called `proxy-service` in the overall monorepo and scaffolds it out with a `maintainers` and `azure-pipeline.yaml` file.

TODO: What is created?

## Adding a Service Revision Ring

TODO: How does all of this work?

With the service created, she next wants to add the rings for the service.  She starts with the `dev` ring:

```bash
??? $ spk ring scaffold search-service dev ???
```

She has now created `proxy-service` with three rings as a set of changes in the Fabrikate deployment.  As part of the development and ops team's workflow, they always use pull requests and code reviews of these changes.

She creates a PR for the changes and asks Olina to review.  Olina approves the PR, merges it to master.

## Observing Deployments

The merge to master that Olina makes kicks off the GitOps pipeline that we built earlier. In the absence of tooling, this GitOps pipeline is observable, but only through manual navigation to various Azure Devops stages and manually collecting logs from Flux in the cluster.

```bash
$ spk deployment init --storage-account=xxxx --storage-key=xxxx ...
```

Instead Dana wants to observe the process in her command line and she uses the `spk` command line tool to do this:

```bash
$ spk deployment get --service proxy-service
```

which displays a new log line in her console as each step is completed that looks like

TODO: Insert table view from output
