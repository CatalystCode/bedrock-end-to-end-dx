# Bedrock Developer and Operations Experience "North Star"

A scenario based description of how all the tools and components in Bedrock fit together to easily define, build, deploy, and maintain a workload running in a Kubernetes cluster.

THERE ARE NO SACRED COWS IN THIS DOCUMENT. Please point out misunderstandings, problems, and places that the experience could be better.

## Getting Started

Dag, who is in an developer role at a company called Fabrikam, has heard about Bedrock from others in his company and would like to use it on a project he is leading to launch a microservice workload called `discovery-service`.

He first installs the `spk` tool, which provides helpful automation around defining and operating Kubernetes clusters with Bedrock principles:

TODO: Confirm this is how we are going to install `spk`

```bash
$ wget https://github.com/microsoft/spektate/releases/download/1.0.1/spk-v1.0.1-darwin-amd64.zip
$ unzip spk-v1.0.1-darwin-amd64.zip
$ mv spk ~/bin (or as appropriate to place it in your path)
```

## Initializing spk tool

With `spk` installed, he initializes the `spk` tool with:

TODO: Is this the case?

```bash
$ spk init
```

This creates a `~/.spk/config` file with auth details, validates that prereqs are installed (`git`, `terraform`, `helm`, `az` cli tools), inventories their versions, and validates that they are compatible with `spk`. The tool notices that she does not have a compatible version of `helm` and asks her to update it, which he does with his favorite package manager manually, and then reruns `spk init`.

## Adopting Bedrock in Existing Application Monorepo

With his `spk` tool initialized, he wants to add an existing `discovery-service` service to the project. `discovery-service` is a service that is already deployed in a non-containerized environment and has been developed in a [monorepo](https://en.wikipedia.org/wiki/Monorepo) style. As we mentioned, Dag wants to use Bedrock to deploy these microservices, so he navigates to the root of this monorepo that he has cloned on his machine:

```bash
$ cd discovery-service
```

and then uses `spk` to initialize it:

TODO: Is this still the case?

```bash
$ spk project init -m
```

where `-m` indicates to `spk` that this a monorepo for multiple microservices.

## Adding an Existing Service

The core `discovery-service` microservice already exists, so he grandfathers it into the Bedrock workflow with:

```bash
$ spk service create discovery-service
```

This service has an existing `azure-pipelines.yaml` file, so the tool is smart enough to skip adding a scaffolded version, but it does scaffold out both a `maintainers` and `Bedrockconfig` file.

TODO: Is this correct?

`service create` also uses cached credentials to create a Azure Devops build pipeline for the `azure-pipelines.yaml` file to process on each commit to build the application, and in this case, push a container to Azure Container Registry.

Since the high level definition and resource manifest repos specified don't currently exist, it prints a warning that it is creating them, and then creates them with a high level Fabriakte definition that includes this service. It then creates a release pipeline step to automatically update the image in the Fabrikate high level definition with each push to ACR and also creates the build pipeline step to build the high level definition and check the results into resource manifest repo.

## Adding a New Service

Dag moves on to a second, and new, microservice called the `proxy-service`. He starts by creating a development branch called `add_proxy_service` on the overall monorepo, and then scaffolds the project with:

```bash
$ spk service create proxy-service
```

Since this is a completely new service, this creates a directory called `proxy-service` in the overall monorepo and scaffolds it out with a `maintainers`, `Bedrockconfig`, and `azure-pipeline.yaml` file.

Like the previous service, it also creates the source to Azure Container Registry Azure Devops pipeline build step and release step to update the image in the Fabrikate high level definition. However, it checks and notices that the previous service has created all of the other required repos and infrastructure, and so does not need to create them, and finishes.

## Building High Level Definition for Workload

Once Dag finishes onboarding the services onto Bedrock, he reaches out to Olina, a colleague in an operations role at Fabrikam, to have her start work on a high level definition of the workload in the cluster.  To do this, she first creates a high level definition repo `discovery-cluster-definition`, and clones it into a local directory.

```bash
$ git clone https://github.com/fabrikam/discovery-cluster-definition
```

She then uses Fabrikate to add the common azure-native monitoring and operations stack her company uses across Kubenetes deployments:

```bash
$ fab add azure-native --source https://github.com/fabrikam/fabrikate-definitions --path definitions/azure-native
```

this creates a `component.yaml` file that is the root  of her Fabikate definition that looks like this:

```yaml
name: discovery-cluster-definition
subcomponents:
- name: discovery-service
  ...
- name: new-service
  ...
- name: azrure-native
  type: component
  source: https://github.com/fabrikam/fabrikate-definitions
  method: git
  path: definitions/azure-native
  branch: master
```

which she then commits back to the repo. This triggers the generation process and this deployment definition will be built into resource manifests that are committed to `github.com/fabrikam/discovery-cluster-manifests`.

## Scaffolding a Cluster Definition

Olina then moves on to wanting to create her infrastructure deployment definition.  She suspects that the project may grow beyond just a single infra deployment.  Each deployment will be similar in structure but differ in settings (region, connection strings, etc)

To scaffold the skeleton of the project, she issued the `spk infra scaffold` command:

```bash
$ spk infra scaffold discovery-cluster-infra --bedrock-source https://github.com/fabrikam/bedrock –-container-name discovery-cluster –-backend-key <key>
```

This creates a `discovery-cluster-infra.json` file with a locked source at the latest version (such that it does not change underneath the infrastructure team) and a set of variables (with defaults where appropriate) that must be defined for each deployment.

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

## Creating a Specific Cluster Definition

Now that Olina has scaffolded out her desired template for infrascture deployment, she must define one or more specific deployments.  To do this, she issues the command:

```bash
$ spk infra define --definition discovery-cluster-infra --name discovery-cluster-west
```

What this does is, using the `discovery-cluster-infra.json` file generated using `spk scaffold`, a cluster definition json `discovery-cluster-one.json` is generated along side the `discovery-cluster-infra.json` file with specific information filled in to the cluster.  These may be handled by either a set of `key:value` pairs on the command line or hand editted once the `discovery-cluster-west.json` file is generated.  The file resembles:

```js
{​
    name: 'discovery-cluster-west',
    base_config: 'discovery-cluster-infra',
    type: 'cluster-definition',
​
    variables: {​
        resource_group_name: '<resource-group-name>',​
        cluster_name: 'discovery-cluster-west,​
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

Olina can repeat this process for additional deployment definitions, say for `east` as `discovery-cluster-east`.

TODO: Should service principal details be in this file? (probably not, since it will be checked in)

## Generating Cluster Terraform Templates

When Olina is done defining her cluster definitions, she will need to generate Terraform sctips (and variables) in order to deploy the cluster(s).  This is handled with the command:

```bash
$ spk infra generate discovery-cluster-infra
```

This command will parse through each of the defined clusters, and using the base template create a Terraform template and corresponding `tfvar` file for each of the clusters defined.

If for some reason, Olina was just interested in updating a single cluster definition, she could use the same command as follows:

```bash
$ spk infra generate discovery-cluster-infra --name discovery-cluster-east
```

## Deploying Cluster

With the above defined and the Terraform scripts generated, Olina can leverage Terraform tools she has installed to deploy (or update) the defined clusters.

```bash
$ terraform init
$ terraform apply -var-file=discovery-cluster-west.tfvars
```

## Introspect Deployments

As Dag and his development team make changes that are deployed into the cluster, they want to be able to observe how these changes progress from a commit to the source code repo, to pushing the container from that build to ACR, to updating the high level definition with this container's image tag, the manifest being generated from this high level definition change, and Flux applying this change within the cluster.

In the absence of tooling, all of this GitOps pipeline is observable, but only through manual navigation to all of these various stages and/or manually collecting logs from Flux in the cluster.

Instead, Dag wants to use `spk` to introspect the status of these deployments. He first initializes the deployment with all of config needed to enable `spk` to have access to various pieces of infrastructure it needs to understand the current state:

```bash
$ spk deployment init --storage-account=xxxx --storage-key=xxxx ...
```

Next, since his `discovery-service` microservice wasn't initially created with introspection enabled, he adds these hooks with:

```bash
$ spk deployment add --name discovery-service
```

This adds introspection to the `discovery-service` pipeline.  With that, on future commits, he can observe deployments with:

```bash
$ spk deployment get --service proxy-service
```

which displays a new log line in her console as each step is completed that looks like

TODO: Insert table view from output
