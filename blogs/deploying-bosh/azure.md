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

We've already made a basic Terraform file for you that can stand up your
control plane if given a few extra parameters. You can find that file
[here](https://github.com/genesis-community/terraforms/tree/master/controlplane/azure).

There is also a `terraform.tfvars.example` file in that directory that can be
used as a basis for creating your `terraform.tfvars` file. It contains all of
the possible parameters you can specify for your deployment.

The `subscription_id`, `tenant_id`, `client_id`, and `client_secret` were all
values that you should have taken note of while making your app registration.
The `resource_group_name` is the name of the resource group that you'd like
Terraform to create. 

The `location` is the Azure location that you wish resources to be created in. 

The `starting_address` is the first address of a `/16` range you'd like the
network to use. Try to select a range that isn't in use elsewhere in your
subscription, in case you ever decide to peer the networks.

`dns_servers` are the default dns servers that will be used by resources in the
network. By default, its set to use the public Cloudflare servers.

`ssh_keys` are the public keys that will be put on in the `authorized_keys`
file of the bastion box that gets created. You'll need to provide one in order
to log in.

Once configured, run the following from the directory with your `azure.tf` and
`terraform.tfvars` file.

```
terraform apply
```

Take note of the output variables after a successful apply, as they contain the
answers to many of the questions that Genesis will ask you later.

## Setting Up Your Bastion Host

Once your bastion box is created, you should be able to ssh into it if you
configured your Terraform file properly. Once inside, you'll need to install
the tools needed for Genesis. For instructions on how to install these tools,
see [this document](https://github.com/genesis-community/website/blob/master/blogs/toolchain/install.md).

## Starting a Local Vault Instance

We use the [Safe CLI](https://github.com/starkandwayne/safe) to interact with
Vault in general. You should have it installed after after following the
instructions on setting up your bastion host. We're going to use Safe to
quickly set up a temporary local Vault instance that Genesis can use until we
are able to set up a permanent Vault through BOSH at a later time.

To do this, open a separate shell session / tmux pane on the jumpbox and run 

```
safe local --memory
```

Safe will output the target name of the new Vault, target it, and authenticate
to it automatically.

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
about to make. The prompts will look something like the following:

```
ubuntu@bastion:~/deployments/bosh-deployments$ genesis new snw-eastus-controlplane
Setting up new environment snw-eastus-controlplane...

Using vault at http://127.0.0.1:8201.
Verifying availability...ok


Is this a proto-BOSH director?
[y|n] > y

What static IP do you want to deploy this BOSH director on?
> 10.0.0.5

What network should this BOSH director exist in (in CIDR notation)?
> 10.0.0.0/24

What default gateway (IP address) should this BOSH director use?
> 10.0.0.1

What DNS servers should BOSH use? (leave value empty to end)
1st value > 1.1.1.1
2nd value > 1.0.0.1
3rd value >

What IaaS will this BOSH director orchestrate?
  1) VMWare vSphere
  2) Amazon Web Services
  3) Microsoft Azure
  4) Google Cloud Platform
  5) OpenStack
  6) BOSH Warden

Select choice > Microsoft Azure

What is your Azure Client ID?
> 01234567-89ab-cdef-0123-456789abcdef
What is your Azure Client Secret?
client_secret [hidden]:
client_secret [confirm]:


What is your Azure Tenant ID?
> 01234567-89ab-cdef-0123-456789abcdef

What is your Azure Subscription ID?
> 01234567-89ab-cdef-0123-456789abcdef

What Azure Resource Group will BOSH be deploying VMs into?
> genesis

What security group should be used as the BOSH default security group?
> genesis-sg

What is the name of your Azure Virtual Network?
> genesis-network

What is the name of the Azure subnet that the BOSH will be placed in?
> genesis-controlplane-subnet

Would you like to edit the environment file?
[y|n] > n
 - auto-generating credentials (in secret/snw/eastus/controlplane/bosh)...
 - auto-generating certificates (in secret/snw/eastus/controlplane/bosh)...
New environment snw-eastus-controlplane provisioned!

To deploy, run this:

  genesis deploy 'snw-eastus-controlplane'

```

Of note, if this is your first BOSH, then this _is_ a Proto-BOSH director.
Also, if you've read this far, the IaaS you're targeting is likely Microsoft
Azure. The remaining answers can be found in the outputs of your Terraform
apply or in your `terraform.tfvars` file.

The command will then spend some time generating certificates and secrets for
the deployment to use. These will get stored in the Vault and made accessible
by the `safe` utility.

An environment file named after your environment name (e.g.
`snw-eastus-controlplane.yml`) will be created. This contains all non-sensitive
information about your deployment. It is safe to store in version control. If
you needed to change anything about your deployment, you would make your changes
in this file. For the purpose of this tutorial, we will assume that the defaults
are fine and that you don't need to make changes to this file.

## Deploying BOSH

```
genesis deploy snw-eastus-controlplane
```

Substitute `snw-eastus-controlplane` with the name of your environment (the
argument you gave to `genesis new`). Genesis should handle the rest of the
deployment for you.

To log in to the BOSH director, you can run:

```
genesis do snw-eastus-controlplane -- alias
genesis do snw-eastus-controlplane -- login
```

Substitute snw-eastus-controlplane with your own environment name.

## Next Steps

If you're building out a full environment, at this point you should be looking
to deploy a permanent Vault. With your new BOSH. We're working on making a document
on this in-particular, and this section will be updated with a link to that once it
is completed.
