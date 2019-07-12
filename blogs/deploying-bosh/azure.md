# Deploying a BOSH to Microsoft Azure Using Genesis and Terraform

## Introduction

BOSH can be an effective way to manage your infrastructure, but before you
have a BOSH director, you've got to bootstrap your infrastructure to a point
where you can deploy a BOSH Director to it. In this post, we'll cover setting
up an automation account, using Terraform to configure Azure all the way up
to making a network for your new BOSH director, and using Genesis to quickly
configure and deploy your BOSH director.

## Prerequisites

This post does not cover the installation of the Genesis toolchain (genesis,
spruce, etc.). It is expected that you have these things installed already.

You'll need to create your own Azure account and subscription (or have your
organization create one for you). Your account will need ownership privilege
over the subscription in order to make automation credentials. If you don't
have these privileges, you'll need somebody to take the steps in the next
section for you.

## Getting Automation Credentials

App registrations in Azure are automation accounts. We're going to need one of
these so that Terraform and BOSH can authenticate and manipulate resources. To
create one, first log in to the Azure portal (portal.azure.com).

Once logged in, navigate to the `Azure Active Directory` on your sidebar. This
presents a new sidebar, on which you should navigate to `App registrations`.
This will open up yet another bar - this time a horizontal one - and you should
click on `New registration` therein.

Now you have a small form to complete. It will ask you to provide a display
name for the application, the account type, and a redirect URI. The display
name can be whatever you want it to be. The account type will almost always
be set to `Accounts in this organizational directory only`, and the redirect
URI can be left blank for our use cases. The Redirect URI type can be set to
web.

Once you apply the settings, you should be brought to the app's overview page.
If you should navigate away for any reason, you can come back to this page by
finding it in the list of app registrations in the Azure Active Directory panel.
This page contains a few pieces of information we need. To authenticate as an
application, we need a set of credentials that Azure refers to as a "service
principal". This includes the following:

* Application ID
* Client Secret
* Tenant ID

This page contains the application ID and tenant ID. Take note of them, and
then navigate to `Certificates & secrets`. Create a new client secret; set the
expiry to something you're comfortable with (secrets can be rotated at a later
time). Then, take the output value and note that down as well.

We will also need the ID of the subscription that resources will be created in
(and billed to). To find the ID, search `Subscriptions` in the top bar, go to
the list of subscriptions and select the desired subscription from the list.
Copy the ID and note it down.

While you're still on the subscription page, you can give the application
you created access to create resources in the target subscription. To d
so, click on `Access Control (IAM)` on the sidebar, and then `Add a role
assignment`. Search for and select the name of the application you previously
created, and give it the role of `Contributor`.

## Setting up Terraform

Now that we have automation credentials, we can start standing up
infrastructure using Terraform. Terraform binaries and documentation can be
found at [https://terraform.io](https://terraform.io).

Terraform works using a series of plugins they call "providers." For Azure, we
will be primarily using the `azurerm` (Azure Resource Manager) provider.

First, make a file called `terraform.tfvars`. A `tfvars` file is a place where
we can store secrets separate from the main configuration file. At a later time,
you may want to store these in a more secure location (such as Vault). For now,
though, just make sure this file does not get checked into source control.

In this `terraform.tfvars` file, define the following variables:

```hcl
subscription_id = "01234567-89ab-cdef-0123-456789abcdef"
tenant_id       = "01234567-89ab-cdef-0123-456789abcdef"
client_id       = "01234567-89ab-cdef-0123-456789abcdef"
client_secret   = "supersecret"
```

You should substitute the placeholder values with the values you noted down
earlier.

Now, create another file - `azure.tf`. This will be our main configuration file.
Here, we need to define our provider. We give it the information it needs to
authenticate and target our subscription. The following block can likely be
pasted in verbatim.

```hcl
variable "subscription_id" {}
variable "tenant_id"       {}
variable "client_id"       {}
variable "client_secret"   {}

provider "azurerm" {
	subscription_id = "${var.subscription_id}"
	tenant_id       = "${var.tenant_id}"
	client_id       = "${var.client_id}"
	client_secret   = "${var.client_secret}"
}
```
If you're unfamiliar with Terraform, take note of the `${foo}` syntax. This is how
you reference values from elsewhere in the document. The `var` prefix means
that the value was defined in a variable block, but you may also see `data`, if
something is referenced in a data block.

## Making a Resource Group

A resource group is a organizational tool for all of the objects you want to
make in Azure. VMs, Virtual Networks, IPs, and just about anything else you can
create get associated with a resource group. We need to make one. 

You can call yours whatever you want, but I'm going to call mine `genesis-doc`.
Append the resource group your `azure.tf` file:

```hcl
resource "azurerm_resource_group" "genesis" {
	name      = "genesis-doc"
	location  = "East US"
}
```

The name should be whatever you want to call it. The location should be whereever
you want your objects to exist. The Azure reference for available region names
can be found [here](https://azure.microsoft.com/en-us/global-infrastructure/regions/).

## Setting Up Your Network

A virtual network in Azure is a virtual address space that can be split into
subnets.  I am going to create mine at `10.0.0.0/16`. Depending on your
anticipated needs, you may need to make a larger or smaller address space.
Additionally, if somebody in your workspace is already using a particular
range, you may want to start your range at a different point in order to avoid
address collisions should you ever peer the networks.

Here's an example of a virtual network definition:

```hcl
resource "azurerm_virtual_network" "genesis-bosh-east" {
  name          = "genesis-bosh-east"
  location      = "${azurerm_resource_group.genesis-doc.location}"
  resource_group_name = "${azurerm_resource_group.genesis-doc.name}"
  address_space = ["10.0.0.0/16"]
  dns_servers   = ["1.1.1.1", "1.0.0.1"]
}
```

You can, again, give it any name you want. The address space should be set
according to what your needs are.  For DNS servers, I've opted for Cloudflare's
public DNS, but if you have a preference or some DNS servers internal to your
network already, you can set the DNS settings to point to those instead.

We now need to slice a subnet out of that address space for our control plane. 
A `/24` is usually suitable, so that's what I'll be making. Again, this can be
changed to your unique needs.

```hcl
resource "azurerm_subnet" "genesis-bosh-east-controlplane" {
  name                 = "genesis-bosh-east-controlplane-subnet"
  resource_group_name  = "${azurerm_resource_group.genesis-doc.name}"
  virtual_network_name = "${azurerm_virtual_network.genesis-bosh-east.name}"
  address_prefix       = "10.0.0.0/24"
}
```

And for now, we'll also need to give the subnet a security group. The default
rules block all access from the internet, but allow all traffic outbound to
the internet. They also allow bidirectional traffic within the virtual network.
There's a fair chance that your current jumpbox is not located in the virtual
network, so we'll need to make a special rule that allows traffic from your
jumpbox in the network security group we create.

After defining the network security group, you'll create an association between
the security group and the subnet to apply it.

```hcl
resource "azurerm_network_security_group" "controlplane" {
  name     = "genesis-controlplane-sg"
  location = "${azurerm_resource_group.genesis-doc.location}"
  resource_group_name = "${azurerm_resource_group.genesis-doc.name}"

  security_rule {
    name                       = "allow_jumpbox_access"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "*"
    source_port_range          = "*"
    destination_port_range     = "*"
    source_address_prefix      = "1.2.3.4/32"
    destination_address_prefix = "*"
  }
}

resource "azurerm_subnet_network_security_group_association" "genesis-bosh-east-controlplane-sg-a" {
  subnet_id   = "${azurerm_subnet.genesis-bosh-east-controlplane.id}"
  network_security_group_id = "${azurerm_network_security_group.controlplane.id}"
}
```

The source address prefix uses `1.2.3.4` as a placeholder; you'll need to provide your own jumpbox
IP address instead.

Once done, if everything was configured properly, you can run `terraform apply`.
Carefully read the changelog that is presented by Terraform, and enter `yes` if
you find everything suitable. Terraform will then create all of the things that
were defined in the configuration file.

## Starting a Local Vault Instance

We use the [Safe CLI](https://github.com/starkandwayne/safe) to interact with
Vault in general. In this case, we're going to use it to quickly set up a
temporary local Vault instance that Genesis can use until we set up a permanent
Vault instance later.

To do this, open a separate shell session / tmux pane on the jumpbox and run 

```
safe local --memory
```

Safe will output the target name of the new Vault and authenticate to it
automatically.

## Configuring a New BOSH Installation

Navigate to the directory in which you would like to store the Genesis
deployment directory and run 

```
genesis init bosh -k bosh
```

Newer versions of genesis will prompt you at this point about which Vault
you would like to use. Select the target that was created by `safe local`.

`cd` into the newly created directory. Now we need to tell Genesis to
initialize the configuration file for a new proto-BOSH director deployment.
Decide what you would like to call this environment. We typically recommend
an `<org>-<site>-<env>` naming scheme. I'll call mine `snw-eastus-controlplane`.
Once this is determined, run `genesis new` with the environment name.

```
genesis new snw-eastus-controlplane
```

You'll now be walked through a series of questions about the deployment you're
about to make.

Of note, if this is your first BOSH, then this _is_ a Proto-BOSH director. 

You'll then be prompted for the desired static IP. When choosing the static IP,
know that the first (x.x.x.0) and last (x.x.x.255) IP of the subnet are
reserved for the network address and broadcast addresses respectively. The
second (x.x.x.1) is reserved for your subnet gateway (take note of this -
you'll need it for one of the prompts). Additionally, the x.x.x.2 and x.x.x.3
address are reserved as well for Azure services.  This means that the first
available IP in our example subnet is `10.0.0.4`. We will use that for our BOSH
director in our example. 

When prompted for which IaaS this BOSH will use - if you've read this far, it's
probably Microsoft Azure.

You will then be prompted for the same service principal information that you
put into your `tfvars` file earlier (client id, client secret, tenant id, and
subscription id). Take each of those values as your answers as prompted by
Genesis.

You will then be prompted for the names of the resource group, network security group,
virtual network, and subnet. You can get each of these from the name field for the
`azurerm_resource_group`, `azurerm_network_security_group`, `azurerm_virtual_network`, and
`azurerm_subnet` blocks respectively in your Terraform configuration file.

## The Environment File

Genesis will create an environment file that looks roughly like this:

```yml
kit:
  name:    bosh
  version: 1.4.0
  features:
    - azure
    - proto

params:
  env:   snw-eastus-controlplane

  # These properties definte the host-level networking for a proto-BOSH.
  # Environmental BOSHes can depend on their parent BOSH cloud-config,
  # but for proto- environments, we have to specify these.
  #
  static_ip:       10.0.0.4
  subnet_addr:     10.0.0.0/24
  default_gateway: 10.0.0.1
  dns:
    - 1.1.1.1
    - 1.0.0.1

  # BOSH on Azure needs to know what resource groups to use when
  # deploying VMs, as well as what default security group to apply
  # to VMs that do not otherwise specify them.
  #
  # Azure credentials are stored in the Vault at
  #   secret/snw/eastus/controlplane/bosh/azure
  #
  azure_resource_group: genesis-doc
  azure_default_sg:     genesis-controlplane-sg

  # The following configuration is only necessary for proto-BOSH
  # deployments, since environment BOSHes will derive their networking
  # and resource groups from their parent BOSH cloud-config.
  #
  azure_virtual_network: genesis-bosh-east
  azure_subnet_name:     genesis-bosh-east-controlplane-subnet
```

This is the configuration file that tells genesis how you want your BOSH deployed.
Some of the answers you gave to the prompts will end up in here. Some of the more
security-sensitive answers you gave will be stored in the Vault instead and only
pulled down at deployment time. As a result, this file should be safe to store
in source control.

## Generating Secrets

After you finish answering all of the prompts, Genesis will take some time to
generate numerous credentials for you. These include various passwords and
certificates that the BOSH director will use. These can be viewed in the Vault
by using `safe get`. To view a layout of created secrets, you can use the
`safe tree` command.

## Deploying BOSH

```
genesis deploy snw-eastus-controlplane
```

Substitute `snw-eastus-controlplane` with the name of your environment. Genesis
should handle the rest of the deployment for you.
