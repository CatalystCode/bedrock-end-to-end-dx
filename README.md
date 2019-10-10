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

This takes a configuration file located by convention at `~/.spk/config`, validates that the prerequisite tools are installed that `spk` relies on, inventories their versions, validates that the version being run is compatible with `spk`, and configures them as necessary with the values provided in the config file.

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

## Building High Level Definition

Once Dag finishes onboarding the services onto Bedrock, he reaches out to Olina, a colleague in an operations role at Fabrikam, to have her start work on a high level definition of the workload in the cluster.  To do this, she first clones the high level definition repo `discovery-cluster-definition` into a local directory.

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
- name: azure-native
  type: component
  source: https://github.com/fabrikam/fabrikate-definitions
  method: git
  path: definitions/azure-native
  branch: master
```

which she then commits back to the repo. This triggers the generation process and this deployment definition will be built into resource manifests that are committed to `github.com/fabrikam/discovery-cluster-manifests`.

## Building Cluster Definition

Olina then moves on to create her infrastructure deployment definition.  She suspects that the project may grow beyond just a single cluster to multiple clusters and wants to be able to scalably add clusters without having to hand manage N sets of Terraform scripts. Each deployment will be similar in structure but differ in a few configuration values (region, connection strings, etc). Infrastructure definitions with `spk` are hierarchical, with each layer inheriting from the layer above it, so she starts by creating the globally common definition between all of her infrastructure:

```bash
$ spk infra scaffold --name discovery-service --source https://github.com/fabrikam/bedrock --template cluster/environments/fabrikam-common-infra
```

This creates a directory called `discovery-service` and places a `definition.json` file with a locked source at the latest version (such that it does not change underneath the infrastructure team without them opting into a change) and a block for setting variables that are globally the same for the discovery-service and a Terraform template called `fabrikam-common-infra` that contains all of their common infrastructure items.

```js
{​
    name: "discovery-service",
    source: "https://github.com/fabrikam/bedrock",
    template: "cluster/environments/fabrikam-common-infra",
    version: "d7d905e6551",

    variables: {​
    }​
}
```

Since they want all of their clusters to be of the same size globally, she edits the variable block to include the number of agent VMs and common location for the GitOps repo for each of those clusters that she gets from Dag:

```js
{​
    name: "discovery-service",

    source: "https://github.com/fabrikam/bedrock",
    template: "cluster/environments/fabrikam-common-infra",
    version: "d7d905e6551",

    backend: {​
        storage_account_name: "terraformstate"
        container_name: "discoveryservice"
    }​,

    variables: {​
        agent_vm_count: 3,
        gitops_ssh_url: "git@github.com:fabrikam/discovery-cluster-manifests.git"
    }​
}
```


Now that Olina has scaffolded out the globally common configuration for the `discovery-service`, she wants to define the first cluster that Fabrikam is deploying in the east region.  To do that, she enters the `discovery-service` directory above and issues the command:

```bash
$ spk infra scaffold --name east --source https://github.com/fabrikam/bedrock --template cluster/environments/fabrikam-single-keyvault
```

Like the previous command, this creates a directory called `east` and creates a `definition.json` file in it with the following:

```js
{​
    name: "east",

    source: "https://github.com/fabrikam/bedrock",
    template: "cluster/environments/fabrikam-single-keyvault",
    version: "d7d905e6551",

    backend: {
        key: "east"
    },

    variables: {​
    }​
}
```

She then fills in the east specific variables for this cluster:

```js
{​
    name: "east",

    source: "https://github.com/fabrikam/bedrock",
    template: "cluster/environments/fabrikam-single-keyvault",
    version: "d7d905e6551",

    backend: {
        key: "west"
    },

    variables: {​
        cluster_name: "discovery-cluster-east",​
        gitops_path: "east"
        resource_group_name: "discovery-cluster-east-rg",​
        vnet_name: "discovery-cluster-east-vnet"​
    }​
}
```

Likewise, she wants to create a `west` cluster, which she does in the same manner from the `discovery-service` directory:

```bash
$ spk infra scaffold --name west --source https://github.com/fabrikam/bedrock --template cluster/environments/fabrikam-single-keyvault
```

And fills in the `definition.json` file with the following `west` specific variables:

```js
{​
    name: "west",

    source: "https://github.com/fabrikam/bedrock",
    template: "cluster/environments/fabrikam-single-keyvault",
    version: "d7d905e6551",

    variables: {​
        cluster_name: "discovery-cluster-west",​
        gitops_path: "west"
        resource_group_name: "discovery-cluster-west-rg",​
        vnet_name: "discovery-cluster-west-vnet"​
    }​
}
```

With this, she now has a directory structure resembling:


```bash
discovery-service/
    definition.json
    east/
        definition.json
    west/
        definition.json
```


## Generating Cluster Terraform Templates

With her cluster infrastructure now defined, she can now generate the Terraform scripts for all the infrastructure for this deployment by navigating to the `discovery-cluster` top level directory and running:

```bash
$ spk infra generate
```

This command recursively reads in the definition at the current directory level, applies any variables there to the currently running dictionary for the directory scope, and then, if there is a `source` and `template` defined in the current definition, creates a `generated` directory and fills the Terraform definition using that source and template at the specified version and with the accumulated variables.

In Olina's scenario above, this means that `spk` generates three `generated` directories like this with Terraform scripts ready for deployment.

```bash
discovery-service/
    definition.json
    generated/
        (... common-infra environment template filled with variables from definition.json)
    east/
        definition.json
        generated/
            (... single-keyvault environment template filled with variables from definition.json in east and the root)
    west/
        definition.json
        generated/
            (... single-keyvault environment template filled with variables from definition.json in west and the root)
```

## Deploying Cluster

With the above defined and the Terraform scripts generated, Olina can leverage Terraform tools she has installed to deploy (or update) the defined clusters.  To deploy the infrastructure, she first navigates to `discovery-service/generated` and applies the `common-infra` scripts.

```bash
$ terraform init
$ terraform apply
```

and likewise, afterwards in the `east/generated` and `west/generated` directories.

## Cluster Scaffolding Management

In the case where Olina might want to take some time off, she needs a process so that her colleague(s) can interact with the scaffolding during her absense.  There are multiple ways to handle this, but one that fits well would be to use some form of a source repository (private VSTS repository, private github repo, etc) which will allow for the maintaining of the directory structure and the sharing of the scaffold information.  One needs to make sure, however, that secrets are not commited to the repository.

TODO: Flesh out this description of how Olina could hand off to another person in an operations role named Odin.

## Introspect Deployments

As Dag and his development team make changes that are deployed into the cluster, they want to be able to observe how these changes progress from a commit to the source code repo, to pushing the container from that build to ACR, to updating the high level definition with this container's image tag, the manifest being generated from this high level definition change, and Flux applying this change within the cluster.

In the absence of tooling, all of this GitOps pipeline is observable, but only through manual navigation to all of these various stages and/or manually collecting logs from Flux in the cluster. This is tedious and less than ideal.

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

## Updating to New Infra Template Version

After several weeks, Olina returns to the `discovery-service` project upon the request of her lead.  In the meantime, the central infra template they use for their application cluster deployments, `fabrikam-single-keyvault` has added a new piece of Azure infrastructure that they would like to include in the `east` and `west` cluster deployments they currently have in operations.

Because she used `spk` to build her infra deployment definitions, she can do this by simply adjusting the `version` field in her `east` cluster deployment to the new tag `3bfeff7f77`, and then regenerating the infra deployment templates with:

```bash
$ spk infra generate
```

This will clone the `fabrikam-single-keyvault` environment template at this new version and use it to generate the new set of Terraform environment files with the existing variables in her `definition.json` files.

She then reapplies the `east` cluster, watches the deployment successfully apply, and then repeats the procedure on the `west` cluster to complete the upgrade to the latest version of the central template.
