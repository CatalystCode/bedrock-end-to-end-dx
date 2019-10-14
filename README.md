# Bedrock Developer and Operations Experience

A scenario based description of how all the tools and components in Bedrock fit together to easily define, build, deploy, and maintain a workload running in a Kubernetes cluster.

## Getting Started

Dag, who is in an developer role at a company called Fabrikam, has heard about Bedrock from others in his company and would like to use it on a project he is leading to launch a microservice workload called `discovery-service`.

He first installs the `spk` and `fab` tools, which provides helpful automation around defining and operating Kubernetes clusters with Bedrock principles:

```bash
$ wget https://github.com/microsoft/spektate/releases/download/1.0.1/spk-v1.0.1-darwin-amd64.zip
$ unzip spk-v1.0.1-darwin-amd64.zip
$ mv spk ~/bin (or as appropriate to place it in your path)
```

```bash
$ wget https://github.com/microsoft/fabrikate/releases/download/0.16.2/fab-v0.16.2-darwin-amd64.zip
$ unzip fab-v0.16.2-darwin-amd64.zip
$ mv fab ~/bin (or as appropriate to place it in your path)
```

## Initializing spk tool

With `spk` installed he:

- either exports or creates `.env` file with the following variables like Azure Client Id. These variables hold sensitive information hence they are not stored in plain text in `spk` config files. The `.env.example` file provides a complete list of environment varibales. 

    ```bash
    $ export AZURE_SUBSCRIPTION_ID=7060ac3f-7a3c-44bd-b54c-4bb1e9cabcab
    $ export AZURE_CLIENT_ID=50d65587-abcd-4619-1234-f99fb2ac0987
    $ export AZURE_CLIENT_SECRET=3ac38e00-aaaa-bbbb-bb87-7222bc4b1f11
    $ export AZURE_TENANT_ID=72f988bf-1234-abcd-91ab-2d7cd011db47
    ```

- initializes the `spk` tool with non sensitive information:

    ```bash
    $ spk init -f <spk config file path>. For example: ./spk-config.yaml
    ```

    This takes the configuration file at `./spk-config.yaml` that contains configuration details like Azure Storage account name, Azure DevOps project, and validates that the prerequisite tools that `spk` relies on (like  `git`, `az`, etc.) are installed, inventories their versions, validates that the version being run is compatible with `spk`, and configures them as necessary with the values provided in the config file.

## Adopting Bedrock in Existing Application Monorepo

With his `spk` tool initialized, he wants to use it to add an existing monorepo of microservices. `discovery-service` is a service that is already deployed in a non-containerized environment and has been developed in a [monorepo](https://en.wikipedia.org/wiki/Monorepo) style. As we mentioned, Dag wants to use Bedrock to deploy these microservices in a containerized environment, so he navigates to the root of this monorepo that he has cloned on his machine:

```bash
$ cd discovery-monorepo
```

and then uses `spk` to initialize it:

```bash
$ spk project init -m -d services
```

where `-m` indicates to `spk` that this a monorepo with multiple multiple microservices and that all of our microservices are located in a directory called `services`.  This creates a `bedrock.yaml` in the root directory that contains the set of known services in this repo.  Looking at this, we can see that it is currently empty:

```bash
$ cat bedrock.yaml
services: {}
```

## Adding a Service

The core `discovery-service` microservice already exists, so he grandfathers it into the Bedrock workflow with:

```bash
$ spk service create discovery-service -d services
```

This updates the bedrock.yaml file to include this service:

```bash
$ cat bedrock.yaml
services:
    ./services/discovery-service:
        helm:
            chart:
                branch: ''
                git: ''
                path: ''

```

and adds a `azure-pipelines.yaml` file in `services/discovery-service` to make it available for `build` service like [Azure Pipelines](https://docs.microsoft.com/en-us/azure/devops/pipelines/get-started/what-is-azure-pipelines?view=azure-devops).

`spk` also includes automation for creating the Azure pipeline in Azure DevOps project.  To create that, Dag executes:

```bash
$ spk service create-pipeline discovery-service -n discovery-service-ci
```

which uses the Azure Devops credentials he established when he ran `init` to create a pipeline that will automatically build the `discovery-service` microservice on each commit to a container that is then pushed to Azure Container Registry with an Azure Devops Pipeline called `discovery-service-ci

Optionally, he can executes the following command that will create a service with pipeline automation.

```bash
$ spk service create discovery-service -d services --create-pipeline -n discovery-service-ci
```

With all of this setup, Dag commits the files that `spk` created and pushes them to his monorepo:

```bash
$ git add bedrock.yaml
$ git add maintainers.yaml
$ git add services/discovery-service/azure-pipelines.yaml
$ git commit -m 'Add discovery-service build pipeline'
$ git push origin HEAD
```

This automatically kicks off a Azure Devops build and builds a container for the `discovery-service` that is pushed to Azure Container Registry.

Dag repeats the `service create`, `service create-pipeline`, and `git push` steps for each of the microservices that make up the deployment.

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

Olina then moves on to create her infrastructure deployment definition.  She suspects that the project may grow beyond just a single cluster to multiple clusters and wants to be able to scalably add clusters without having to hand manage N sets of nearly identical Terraform scripts -- each deployment will be similar in structure but differ in a few configuration values (region, connection strings, etc). Infrastructure definitions with `spk` are hierarchical, with each layer inheriting from the layer above it, so she starts by creating the globally common definition between all of her infrastructure:

```bash
$ spk infra scaffold --name discovery-cluster --source https://github.com/fabrikam/bedrock --template cluster/environments/fabrikam-common-infra
```

This creates a directory called `discovery-cluster` and places a `definition.json` file with a locked source at the latest version (such that it does not change underneath the infrastructure team without them opting into a change) and a block for setting variables that are globally the same for the discovery-cluster and a Terraform template called `fabrikam-common-infra` that contains all of their common infrastructure items.

```json
{​
    name: "discovery-cluster",
    source: "https://github.com/fabrikam/bedrock",
    template: "cluster/environments/fabrikam-common-infra",
    version: "d7d905e6551",

    variables: {​
    }​
}
```

Since Olina wants all of Fabrikam clusters to be of the same size globally, she edits the variable block to include the number of agent VMs and common location for the GitOps repo for each of those clusters that she gets from Dag:

```jsson
{​
    name: "discovery-cluster",

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


Now that Olina has scaffolded out the globally common configuration for the `discovery-cluster`, she wants to define the first cluster that Fabrikam is deploying in the east region.  To do that, she enters the `discovery-cluster` directory above and issues the command:

```bash
$ spk infra scaffold --name east --source https://github.com/fabrikam/bedrock --template cluster/environments/fabrikam-single-keyvault
```

Like the previous command, this creates a directory called `east` and creates a `definition.json` file in it with the following:

```json
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

```json
{​
    name: "east",

    source: "https://github.com/fabrikam/bedrock",
    template: "cluster/environments/fabrikam-single-keyvault",
    version: "d7d905e6551",

    backend: {
        key: "east"
    },

    variables: {​
        cluster_name: "discovery-cluster-east",​
        gitops_path: "east"
        resource_group_name: "discovery-cluster-east-rg",​
        vnet_name: "discovery-cluster-east-vnet"​
    }​
}
```

Likewise, she wants to create a `west` cluster, which she does in the same manner from the `discovery-cluster` directory:

```bash
$ spk infra scaffold --name west --source https://github.com/fabrikam/bedrock --template cluster/environments/fabrikam-single-keyvault
```

And fills in the `definition.json` file with the following `west` specific variables:

```json
{​
    name: "west",

    source: "https://github.com/fabrikam/bedrock",
    template: "cluster/environments/fabrikam-single-keyvault",
    version: "d7d905e6551",
    ...

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
discovery-cluster/
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

With the above defined and the Terraform scripts generated, Olina can leverage Terraform tools she has installed to deploy (or update) the defined clusters.  To deploy the infrastructure, she first navigates to `discovery-cluster/generated` and applies the `common-infra` scripts.

```bash
$ terraform init
$ terraform apply
```

and likewise, afterwards in the `east/generated` and `west/generated` directories.

## Cluster Scaffolding Management

In the case where Olina might want to take some time off, she needs a process so that her colleague(s) can interact with the scaffolding during her absense.  There are multiple ways to handle this, but one that fits well would be to use some form of a source repository (private Azure DevOps repo or private github repo, etc) which will allow for the maintaining of the directory structure and the sharing of the scaffold information.  One needs to make sure, however, that secrets are not commited to the repository.

TODO: Flesh out this description of how Olina could hand off to another person in an operations role named Odin.

## Monitor GitOps Process

As Dag and his development team make changes that are deployed into the cluster through the whole GitOps process: they want to be able to observe how these changes progress from a commit to the source code repo, to pushing the container from that build to ACR, to updating the high level definition with this container's image tag, to the manifest being generated from this high level definition change, and Flux applying this change within the cluster.

In the absence of tooling, all of this GitOps process is observable, but only through manual navigation of all of the various stages and/or manually collecting logs from Flux in the cluster. This is tedious and leads to lost developer productivity.

Instead, Dag wants to use `spk` to observe the status of these deployments.  From running `spk init`, his `spk` environment variables and config file has the connection details for how to do that, so he can simply type in his CLI:

```bash
$ spk deployment get --service discovery-service

Start Time            Service        Deployment   Commit  Src to ACR Image Tag                  Result ACR to HLD Env Hld Commit Result HLD to Manifest Result Duration  Status   Manifest Commit End Time
10/9/2019, 4:00:32 PM discovery-service  178fdc0bc226 5b54eb4 6342       discovery-service-master-6342  ✓      225        DEV 99ffcec    ✓      6343            ✓      4.23 mins Complete 20d199d         10/9/2019, 4:03:56 PM
10/9/2019, 3:07:57 PM discovery-service  c66bab558257 5b54eb4 6340       discovery-service-master-6340  ✓      224        DEV 80033b7    ✓      6341            ✓      3.69 mins Complete df32861         10/9/2019, 3:10:41 PM
10/9/2019, 2:52:42 PM discovery-service  4099dea7d5ed 5b54eb4 6338       discovery-service-master-6338  ✓      223        DEV 333dc79    ✓      6339            ✓      3.62 mins Complete e8422e0         10/9/2019, 2:55:18 PM
9/26/2019, 3:13:20 PM discovery-service  1e680e920c27 5b54eb4 6178       discovery-service-master-6178  ✓      209        DEV bc341e0    ✓      6182            ✓      4.34 mins Complete a58001d         9/26/2019, 3:16:53 PM
9/26/2019, 3:13:12 PM discovery-service  939dcb6e3464 5b54eb4 6177       discovery-service-master-6177  ✓      208        DEV f007812    ✓      6180            х      3.00 mins Complete                 9/26/2019, 3:15:28 PM
9/26/2019, 3:13:03 PM discovery-service a902f747d4cc a0bca78 6176       discovery-service-master-6176 ✓      207        DEV c15c700    ✓      6181            ✓      4.45 mins Complete a58001d         9/26/2019, 3:16:46 PM
```

and watch his recent deployment flow through the GitOps pipeline.

## Updating to New Infra Template Version

After several weeks, Olina returns to the `discovery-cluster` project upon the request of her lead.  In the meantime, the central infra template they use for their application cluster deployments, `fabrikam-single-keyvault` has added a new piece of Azure infrastructure that they would like to include in the `east` and `west` cluster deployments they currently have in operations.

Because she used `spk` to build her infra deployment definitions, she can do this by simply adjusting the `version` field in her `east` cluster deployment to the new tag `3bfeff7f77` by replacing the old tag `d7d905e6551`, and then regenerating the infra deployment templates with:

```json
{​
    name: "east",

    source: "https://github.com/fabrikam/bedrock",
    template: "cluster/environments/fabrikam-single-keyvault",
    version: "d7d905e6551",
    ...
}
```

```bash
$ spk infra generate
```

This will clone the `fabrikam-single-keyvault` environment template at this new version and use it to generate the new set of Terraform environment files with the existing variables in her `definition.json` files.

She then reapplies the `east` cluster, watches the deployment successfully apply, and then repeats the procedure on the `west` cluster to complete the upgrade to the latest version of the central template.
