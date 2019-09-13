# Initial Developer Experience

Oddveig, a person in an operations role, has heard about Bedrock and would like to try it out.

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

```json
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

After trying out the ‘hello-world’ stock workload that was provided as part of the template, she wants to use her own workload.  She creates a new manifest repo in Github, adds the gitops ssh key to the repo, and then adjusts `gitops_ssh_url`.

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

She does this for a bit, and quickly realizes that checking in YAML directly has lead to several serious deployment failures and wants to use a higher level definition of the deployment to address this.

She first creates a directory for her high level definition project:

```bash
$ mkdir my-cluster-hld
$ cd my-cluster-hld
```

She then uses fabrikate to scaffold out a top level component for the project
