---
authors:
- kkelani
categories:
- Infrastructure
- Elastic Runtime Tile
- Release Engineering
- Terraform
date: 2017-06-01T19:52:30-07:00
draft: true
short: |
  A look at how Release Engineering deploys and tests the Elastic Runtime Tile
title: "Continuously Deploying Pivotal Cloudfoundry: Elastic Runtime Tile"
---

The Release Engineering team maintains the [Elastic Runtime Tile](https://network.pivotal.io/products/elastic-runtime). The ERT is a product of Pivotal Cloudfoundry that contains open-source Cloudfoundry components and proprietary components, built by Pivotal, that improve management of the platform. [Ops Manager](https://docs.pivotal.io/pivotalcf/1-11/customizing/) is a web application that operators use to configure and deploy tiles, such as the ERT, MySQL, Redis, RabbitMQ, etc.

{{< responsive-figure src="/images/release-engineering-deploying-ert/ops-mgr.png" class="center small" caption="Ops Manager Web App" >}}

Conceptually, our team resides between platform component teams and the customer. Our core responsibilities are to consume platform components, expose configuration options to operators in a user-friendly manner via Ops Manager, and test the platform.

{{< responsive-figure src="/images/release-engineering-deploying-ert/components.png" class="center small" caption="OSS and proprietary components flow into the ERT" >}}

Given our proximity to the customer and our long-term support agreements, it is critical that we deploy and test the product in production-like environments on the infrastructures our customers use. Our testing matrix below details the versions, upgrade paths, and infrastructures we test. 

{{< responsive-figure src="/images/release-engineering-deploying-ert/ert-testing-matrix.png" class="center small" caption="ERT Testing Matrix" >}}

This post will explain the [Concourse](https://www.concourse.ci)-driven system we’ve put in place to deploy and test the ERT.  We use [`terraform`](https://www.terraform.io) to create infrastructure and an Ops Manager instance. We built a tool called `om` to interact with Ops Manager to configure and deploy the ERT. To see the pipelines in action go to https://releng.ci.cf-app.com.

{{< figure src="/images/release-engineering-deploying-ert/pipeline.png" class="center" caption="ERT 1.11 pipeline">}}

## <a name="creating-infrastructure" href="#creating-infrastructure">Creating Infrastructure</a>
[Terraform](https://www.terraform.io) is a tool for creating and modifying infrastructure in an automated manner. It currently supports all major cloud providers. Users define their desired infrastructure components such as load balancer, DNS, network configuration, etc, in template `*.tf` files. After running `terraform apply`, the user is given a state `*.tfstate` file, which contains information about the infrastructure. This is an important file as it allows you to manipulate and destroy your infrastructure in a painless, reentrant way.

Our team maintains a series of repos that contain terraform templates that stand up the required infrastructure to deploy the ERT. These template files exist for [Google Cloud](https://github.com/pivotal-cf/terraforming-gcp), [AWS](https://github.com/pivotal-cf/terraforming-aws), and [Azure](https://github.com/pivotal-cf/terraforming-azure). Since template files do not contain any credentials, they are easy to distribute. Having the ability to describe your infrastructure in template files instead of documentation or writing a client program to interact with the IaaS, is one of the major benefits of terraform.

To create the infrastructure, our pipeline pulls templates from one of the aforementioned repos, creates a terraform variables file `*.tfvars` containing environment specific values (credentials, SSL certs, etc.), and executes terraform. The [terraform concourse resource](https://github.com/ljfranklin/terraform-resource) provides an easy way to safely run terraform templates. As mentioned earlier, the state file is important and confidential. The resource takes templates and variables as inputs, performs the `terraform apply` command, and outputs your state file to an Amazon S3 bucket of your choice. This puts the state file in a safe secure location for further operations.

Check out the README.md in the terraforming repo of your choice for specific info on how to get started.

## <a name="configuring-ops-man-authentication" href="#configuring-ops-man-authentication">Configuring Ops Manager Authentication</a>
After creating the necessary infrastructure, authentication needs to be configured for interacting with Ops Manager. Since we are focused on automation through Concourse, we use `om` (https://github.com/pivotal-cf/om) to interact with the Ops Manager API. Some of you may be familiar with `opsmgr`, which was a similar tool but used Capybara to automate form submission. Unfortunately, `opsmgr` was susceptible to a high rate of false negatives, like failing to find elements on a page because they were overlapping. As a result, we created `om` to improve the reliability of our pipelines.

{{< figure src="/images/release-engineering-deploying-ert/opsmgr.png" class="center" caption="Configuring Ops Manager authentication" >}}

In order to begin interacting with Ops Manager via the API, we set up authentication using the following command:

~~~bash
$ om --target https://pcf.example.com configure-authentication \
  --user desired-username --password desired-password \
  --decryption-passphrase desired-passphrase
~~~

## <a name="uploading-artifacts" href="#uploading-artifacts">Uploading Artifacts</a>
Next, the pipeline uploads the ERT and its stemcell. As mentioned earlier, the ERT contains compiled releases of open-source Cloudfoundry components like loggregator, consul, and diego, as well as Pivotal Cloudfoundry components like autoscaling, apps manager, etc. The tile also contains metadata that describes tile properties and a manifest template that gets filled in with user-provided configuration (see “Configuring ERT”).

{{< figure src="/images/release-engineering-deploying-ert/opsmgr-upload.png" class="center" caption="Uploading a tile" >}}

Once again, since our focus is on automation, uploading is performed using `om`. The ERT is found on [Pivotal Network](https://network.pivotal.io), a site that contains many Pivotal products like the Redis Tile, Spring Cloud Services, and Runtime for Windows.

Stemcells can be found on [bosh.io](https://www.bosh.io). The required stemcell version for an ERT is found on the tile’s download page. For automation purposes, our pipeline unzips the tile, finds the metadata file, and extracts the required stemcell version from the `stemcell_criteria` section of the metadata.

Once we have the ERT and stemcell, they are uploaded to the Ops Manager via the following `om` commands:

~~~bash
$ om --target https://pcf.example.com --user some-user --password password upload-product \
  --product /path/to/product/file.pivotal

$ om --target https://pcf.example.com --user some-user --password password stage-product \
  --product-name cf --product-version 1.10.12

$ om --target https://pcf.example.com --user some-user --password password upload-stemcell \
  --stemcell /path/to/stemcell/file.tgz
~~~

## <a name="configuring-bosh" href="#configuring-bosh">Configuring BOSH</a>
Like deploying the open-source Cloudfoundry, Ops Manager uses BOSH. Much of the same configuration that is required when using [bosh-init](https://bosh.io/docs/using-bosh-init.html) or [bbl](https://github.com/cloudfoundry/bosh-bootloader) is applicable to configuring BOSH via Ops Manager.

{{< figure src="/images/release-engineering-deploying-ert/opsmgr-configure-bosh.png" class="center" caption="BOSH configuration" >}}

To configure the BOSH director, Ops Manager must be provided with details about the underlying infrastructure that it will be deploying VMs into. This is another place where `terraform` is handy. Our pipeline determines this information from the terraform state file `*.tfstate` via the following command:

~~~bash
$ terraform output -state terraform.tfstate | jq -r ‘map_values(.value)’
{ 
    "vm_tag": "some-id-tag-for-vms",
    "project": "some-gcp-project",
    "network_name": "some-network",
    "azs": ["us-central1-a", "us-central1-b", "us-central1-c"],
    ...
}
~~~

The `om configure-bosh` command is populated with values from the above command. Here is an example that sets the IaaS and network configuration for the director:

~~~bash
$ om --target https://pcf.example.com --username some-user --password some-password configure-bosh \
  --iaas-configuration '{"default_deployment_tag": "some-id-tag-for-vms", "project": "some-gcp-project"}'

$ om --target https://pcf.example.com --username some-user --password some-password configure-bosh \
  --az-configuration '{"availability_zones": [{"name": "us-central1-a"}, {"name": "us-central1-b"}, {"name": "us-central1-c"}]}'
...
~~~

To fully configure the BOSH director, check out the examples in `configure-bosh` [command documentation](https://github.com/pivotal-cf/om/tree/master/docs/configure-bosh).

## <a name="configuring-ert" href="#configuring-ert">Configuring ERT</a>

Configuration for open-source Cloudfoundry is provided via a manifest file. However, configuration of the ERT is exposed by a series of forms on Ops Manager and these values are populated into the manifest. The forms allow operators to enable features like container networking, tcp routing, and specify values like SSL certificates for the routers, a destination for external system logging, configuring the Cloud Controller database, etc. Values that are provided in these forms are translated into properties for their respective BOSH jobs.

{{< figure src="/images/release-engineering-deploying-ert/opsmgr-configure-ert.png" class="center" caption="ERT configuration" >}}

Here's an example of part of an `om configure-product` command that is used to configure the “system domain” value that would be provided to the `cloud_controller` job:

~~~bash
$ om --target https://pcf.example.com --username some-user --password some-password configure-product \
  --product_properties='{".cloud_controller.system_domain": {"value": "sys.example.com"}, ... }

...
~~~

To fully configure the ERT, check out the examples in the `configure-product` [command documentation](https://github.com/pivotal-cf/om/tree/master/docs/configure-product).

## <a name="deploying-ert" href="#deploying-ert">Deploying ERT</a>
Now that BOSH is ready to deploy VMs and the ERT is configured, the final step is to deploy the platform. Under the hood, Ops Manager uses `bosh-init` to deploy the director with the configuration provided to the `om configure-bosh` command in the <a href="#configuring-bosh">Configuring BOSH</a> section. It issues a `bosh deploy` to deploy a manifest with the configuration provided to the `om configure-product` command in <a href="#configuring-ert">Configuring the ERT</a> section. The lengthy compilation step is skipped since the BOSH releases in the ERT have already been compiled.

The pipeline applies changes by issuing the following command:

~~~bash
$ om --target https://pcf.example.com --user some-user --password some-password apply-changes
~~~

Running the `apply-changes` command will tail the Ops Manager installation output and exit `0` for a successful deployment and `1` for a failed deployment. The command is also reentrant meaning it will re-attach to an installation in progress.

## <a name="conclusion" href="#conclusion">Conclusion</a>
Automation is critical to testing the Elastic Runtime Tile. To ensure we discover problems before releasing the tile to our customers, it is imperative that we deploy and test different versions, upgrade paths, and configurations on numerous infrastructures. Tools like `terraform` make creating and managing infrastructure state simple. We created `om` to automate a stable and reentrant interaction with Ops Manager. Tooling is at the heart of Release Engineering and it has allowed us to maintain long-term support and confidence in the products we ship to customers.
