+++
title = "The Chef way of managing a hybrid fleet"
description = "Your estate spans on-prem and several clouds, each with its own tooling. Use one workflow and one view of configuration across all of it: write the desired state once, detect where each node runs, enroll nodes everywhere, and roll changes out in controlled waves."
draft = false
gh_repo = "chef-web-docs"

[menu.overview]
  identifier = "overview/solutions/compliance/hybrid_fleet"
  parent = "overview/solutions/compliance"
  title = "The Chef way of managing a hybrid fleet"
  weight = 14
+++


Your servers don't all live in one place.
Some run in your own data centers, and others run in one or more public clouds.
Each place comes with its own tooling: its own image builders, its own scripts, its own console, and its own idea of what "configured correctly" means.
Teams end up maintaining several ways of doing the same job, and the answer to "which servers are out of line?" depends on which console you ask.

You need one workflow for configuring servers and one place to see the result, wherever the servers run.
Chef does this by letting you describe the state you want as code, once.
An agent on each server, Chef Infra Client, makes the server match that description and reports what it did.
The same code, the same central service, and the same reports apply to a server in your data center and to a virtual machine in a cloud.

## Who this page is for

**If you're accountable for the outcome**, such as an infrastructure, operations, or platform owner, read this section and [How Chef manages a hybrid fleet](#how-chef-manages-a-hybrid-fleet).
You'll learn what one workflow across on-premises and cloud means in practice, what Chef doesn't do for you, and where the decisions are.

**If you run the servers**, work through the rest of the page.
It builds a small cookbook that detects where a node runs, converges it, shows how to test the branches, and then explains how nodes in every location connect to one central service, how you enroll them, how you roll changes out, and how you see the whole fleet.
If you haven't used Chef before, you don't need to read anything else first.

### What you get

- **One definition.** A server's desired state is written once, in code, and applied the same way everywhere.
- **Location-aware settings.** The code can detect which environment a node runs in and apply the setting that belongs there, such as a different log endpoint for each location.
- **One central service.** Nodes in your data centers and in the clouds retrieve their cookbooks and policy from the same place.
- **One view.** Run reports and node lists come together in Chef Automate and in Chef 360 Platform, so you don't have to check each environment separately.
- **Controlled change.** The same code is tested on a workstation, promoted through groups of nodes, and rolled out in waves.

### What you don't get

Chef configures the servers it runs on.
It doesn't provision the virtual machines, networks, or accounts in your clouds, and it doesn't replace the tools you use to build them.
It doesn't translate between clouds either: if two clouds need different settings, you write down both, as the example on this page does.
A node is in a known state only for the resources you wrote code for, and only as of its last successful run.

## How Chef manages a hybrid fleet

Five ideas make up the Chef way.
Each one maps to a piece of the platform.

| Idea | What it means | Chef piece |
| --- | --- | --- |
| Describe the state you want | You write what a server should look like, not the steps to get there. A `package` resource says a package should be installed, and Chef works out how on each operating system. | [Chef Infra](/client/latest/) cookbooks, recipes, and resources |
| Detect where you run | Chef Infra Client collects facts about each node, including whether it runs in AWS, Azure, or Google Compute Engine, so one recipe can branch on location. | [Ohai](/client/latest/features/ohai/) and the [cloud helper methods](/client/latest/infra_language/checking_clouds/) |
| Keep the code in one place | Every node, wherever it runs, asks the same central service for the cookbooks and policy that apply to it. | Chef Declarative State Management (DSM) in [Chef 360 Platform](/360/latest/), or Chef Infra Server |
| Test before you ship, and roll out in waves | You test on your workstation first, then promote the same revision through groups of nodes. | [Chef Workstation](/workstation/latest/), policy groups, and [Chef Courier](/360/latest/understand/courier/) jobs |
| See everything in one place | Run reports and node lists from every location are collected centrally. | [Chef Automate](/automate/) and Chef 360 Node Management |

A node pulls its configuration.
Chef Infra Client contacts the central service on a schedule, downloads the cookbooks that apply, and, only if the node doesn't match what the cookbook says, makes the change.
Because the work happens on the nodes, the central service doesn't have to run your configuration for you, and a node needs only network access to it.
That's why the same model works in a data center and in a cloud.

{{< warning >}}

**Check the lifecycle before you design.**
Chef Infra Server is deprecated, and Chef 360 Platform is its replacement.
The [supported versions](/versions/) page lists the end-of-life dates for Chef Infra Server, Chef Automate, and Chef Infra Client.
Dates can change, so confirm them on that page and with your Chef account team before you commit to a design.

{{< /warning >}}

The central service is a choice, and it's the one decision that every location shares:

| If you | Then |
| --- | --- |
| Run Chef Infra Server today | Plan a move to DSM in Chef 360 Platform. See [Migrate from Chef Infra Server](#migrate-from-chef-infra-server). |
| Are building a new estate | Use Chef 360 Platform, and decide whether Progress hosts it (SaaS) or you run it. See [Choose where the platform runs](#choose-where-the-platform-runs). |
| Have servers you can reach over SSH but don't want an agent on | Use Agentless Mode for Linux nodes. See [Bring nodes under management](#bring-nodes-under-management). |

## What you'll build

A Chef Infra cookbook that does two things on a node:

- Detects whether the node runs on-premises, in AWS, in Azure, or in Google Cloud.
- Writes one configuration file whose contents depend on the location, so a hand edit is reverted at the next run.

It's a small stand-in for real work, such as pointing every server at the log collector closest to it.
You'll run it on a node that isn't in a cloud, and then check that the cloud branches pick the right value.
Then the page shows how the same cookbook reaches nodes in every location.

## Before you begin

You need:

- [Chef Workstation](/workstation/latest/install/) on the machine where you write and test code.
  It includes the `chef` and `chef-client` commands and the testing tools.
- A test node or a scratch directory.
  The example runs in your workstation's local mode and writes to a temporary directory, so it doesn't change the system.
- A Progress Chef license, for bootstrapping nodes and for commercial downloads.
  Chef Infra Client might ask you to accept the license terms the first time you run it.

{{< note >}}

Chef Infra Client 19 and Chef Workstation 26 are the current releases.
To bootstrap a node with Chef Infra Client 19 or later, you need Chef Workstation 26 or later.
The output on this page comes from Chef Infra Client 18.10, so output from 19 might differ slightly.

{{< /note >}}

## Write the cookbook

To create a cookbook that applies a location-specific setting, follow these steps:

1. Create a repository and a cookbook inside it:

   ```shell
   chef generate repo chef-repo
   cd chef-repo
   chef generate cookbook cookbooks/hybrid_baseline
   ```

   The cookbook contains a `recipes/default.rb` for your recipe, a `metadata.rb`, a `Policyfile.rb`, a `kitchen.yml` for testing, and a `test` directory.
   See [cookbooks](/client/latest/cookbooks/) for what each part is for.

1. Create `cookbooks/hybrid_baseline/attributes/default.rb`:

   ```ruby
   default['hybrid_baseline']['config_dir'] = '/etc/hybrid-baseline'

   # One value per location. Replace the hostnames with your own.
   default['hybrid_baseline']['log_endpoints'] = {
     'on-prem' => 'logs.dc1.example.com',
     'aws' => 'logs.aws.example.com',
     'azure' => 'logs.azure.example.com',
     'gcp' => 'logs.gcp.example.com',
   }
   ```

   An attribute holds a value that changes between nodes or environments, so the recipe doesn't have to.
   Keeping the per-location values here means you change a hostname in one place, and you don't touch the logic.

1. Replace `cookbooks/hybrid_baseline/recipes/default.rb` with:

   ```ruby
   location =
     if ec2?
       'aws'
     elsif azure?
       'azure'
     elsif gce?
       'gcp'
     else
       'on-prem'
     end

   directory node['hybrid_baseline']['config_dir'] do
     mode '0755'
     recursive true
   end

   file "#{node['hybrid_baseline']['config_dir']}/site.conf" do
     content <<~CONF
       # Managed by Chef. Local edits are replaced on the next Chef Infra Client run.
       location=#{location}
       log_endpoint=#{node['hybrid_baseline']['log_endpoints'][location]}
       PermitRootLogin no
     CONF
     mode '0644'
   end
   ```

   `ec2?`, `azure?`, and `gce?` are [helper methods](/client/latest/infra_language/checking_clouds/) that report whether the node is running in AWS EC2, Microsoft Azure, or Google Compute Engine.
   Chef Infra Client 15.8 and later includes them, and they read the data that Ohai collects about the node.
   A node that matches none of them is treated as on-premises, which is a choice this recipe makes, not something Chef decides.

   The `directory` and `file` blocks are [resources](/client/latest/resources/bundled/).
   They state what should exist and how.
   They don't say how to create it, because Chef Infra Client works that out for the platform it's running on.

## Converge a node

To run the cookbook on a node that isn't in a cloud, follow these steps:

1. Create `attrs.json` in the repository, so the example writes to a temporary directory instead of `/etc`:

   ```json
   {
     "hybrid_baseline": {
       "config_dir": "/tmp/hybrid-baseline-demo"
     }
   }
   ```

   On a real node, leave the default and don't use this file.

1. Run the recipe in local mode, which runs a lightweight Chef Infra Server on your workstation for the length of the run:

   ```shell
   chef-client -z -o 'recipe[hybrid_baseline]' -j attrs.json
   ```

   Accept the license prompt if you see one.
   Because this node isn't in a cloud, the recipe chooses the on-prem branch:

   ```text
   Converging 2 resources
   Recipe: hybrid_baseline::default
     * directory[/tmp/hybrid-baseline-demo] action create
       - create new directory /tmp/hybrid-baseline-demo
       - change mode from '' to '0755'
     * file[/tmp/hybrid-baseline-demo/site.conf] action create
       - create new file /tmp/hybrid-baseline-demo/site.conf
       @@ -1,4 +1,8 @@
       +# Managed by Chef. Local edits are replaced on the next Chef Infra Client run.
       +location=on-prem
       +log_endpoint=logs.dc1.example.com
       +PermitRootLogin no
       - change mode from '' to '0644'
   Infra Phase complete, 2/2 resources updated in 07 seconds
   ```

1. Run it again.
   Nothing has drifted, so nothing changes, and the run reports `0/2 resources updated`.
   This is idempotence, and it's why you can run Chef on a schedule on every node, in every location, without worrying about nodes that are already correct.

   If someone edits the file by hand, the next run reverts the edit and shows the difference.
   The [on-premises fleet page](/solutions/compliance/on_prem_fleet/) walks through that drift correction step by step.

The output here is from a test run, trimmed, with paths shortened.
Your timings differ.

## Check the cloud branches

You can't run a cloud node on a laptop, but you can still check that each branch picks the right value.
The helpers read the data Ohai collects, so you can supply that data yourself in a test.
These two checks use the same cookbook, so they test your logic and not your cloud account.

1. Create `cookbooks/hybrid_baseline/spec/unit/recipes/default_spec.rb`:

   ```ruby
   require 'chefspec'

   describe 'hybrid_baseline::default' do
     platform 'ubuntu'

     context 'on a node that is not in a cloud' do
       it 'sets the on-prem location' do
         expect(chef_run).to render_file('/etc/hybrid-baseline/site.conf')
           .with_content(/location=on-prem/)
       end
     end

     context 'on an EC2 node' do
       override_attributes['ec2'] = { 'instance_id' => 'i-0abc' }

       it 'sets the aws location' do
         expect(chef_run).to render_file('/etc/hybrid-baseline/site.conf')
           .with_content(/location=aws/)
       end
     end
   end
   ```

   [ChefSpec](/workstation/latest/tools/chefspec/) runs the recipe in memory and checks what it would do, without changing a system.
   The `override_attributes` line stands in for the data Ohai would collect on a real EC2 node.

1. Run the test from the cookbook directory:

   ```shell
   chef exec rspec spec
   ```

   ```text
   ..

   2 examples, 0 failures
   ```

A local run of the same cookbook, with fake `ec2`, `azure`, or `gce` data passed in through the `-j` file, produced these results:

| Data supplied to the run | `location` | `log_endpoint` |
| --- | --- | --- |
| None | `on-prem` | `logs.dc1.example.com` |
| `ec2` | `aws` | `logs.aws.example.com` |
| `azure` | `azure` | `logs.azure.example.com` |
| `gce` | `gcp` | `logs.gcp.example.com` |

On a real cloud node, you don't supply any data: Ohai collects it during the run.
The Ohai plugins for EC2, Azure, and Google Compute Engine are among the [default plugins](/client/latest/features/ohai/).
The documentation doesn't list the individual attributes each plugin sets, so the safest checks are the helper methods, as this recipe uses.

## Test before you ship

A cookbook that changes servers needs testing before it runs on them.
Chef Workstation includes the tools for that, so a change can be checked on your workstation or in a disposable virtual machine first.

| Tool | What it checks |
| --- | --- |
| Cookstyle | Style and common mistakes in your Chef Infra code |
| ChefSpec | What a recipe would do, without running it on a system |
| Test Kitchen | Whether the cookbook converges on a real, throwaway machine, and whether the result passes your tests |
| [Chef InSpec](/inspec/latest/) | Whether the finished system meets requirements, such as a security baseline |

The `kitchen.yml` that `chef generate cookbook` creates is the starting point for Test Kitchen.
See [Chef Workstation](/workstation/latest/) and [Test Kitchen](/workstation/latest/tools/kitchen/) for how to run them.

## Connect every node to one central service

A hybrid fleet works because every node, in every location, points at the same central service.
That service stores the cookbooks, policies, and node metadata, so the code you test once is the code every node retrieves.

| Location | What the node needs |
| --- | --- |
| Data center | A network path to the central service, and a bootstrap or enrollment method. |
| Each cloud | A network path from that cloud's networks to the central service, and a bootstrap or enrollment method. |

On the node, the `chef_server_url` setting in `client.rb` tells Chef Infra Client where the central service is.
With Chef 360 Platform, that's the DSM address, and Chef Workstation and `knife` connect to it the same way.
See [connect to DSM](/360/latest/use/operators/dsm_operations/connect_to_dsm/).

Plan the network, because it's the part Chef doesn't do for you.
Check each location's firewalls and security groups, and any proxy, against the requirements for the product you use.
Chef Infra Client reads proxy settings such as `http_proxy`, `https_proxy`, and `no_proxy`.
See [proxies](/client/latest/install/proxies/).

### Choose where the platform runs

Chef 360 Platform runs in three ways.
You choose one, and every location connects to it.

| Deployment model | Who runs the platform | Best for |
| --- | --- | --- |
| SaaS | Progress hosts and operates the platform | Teams that don't want to run any platform infrastructure. The docs describe it as operating across hybrid environments, from on-premises data centers through to all major public clouds. |
| Embedded cluster | You run Chef 360 Platform Server, and Chef provisions and manages the Kubernetes environment for it | A quicker self-managed setup with fewer cluster tasks |
| Bring your own Kubernetes (BYOK) | You run Chef 360 Platform Server on a Kubernetes cluster you manage | Teams that already run Kubernetes, or that need tighter control over networking and security |

For a hybrid estate, the choice mostly comes down to who operates the platform and how nodes in each location reach it.
SaaS nodes need outbound access on ports 443 and 31050, and the node requirements list more, such as the `bldr.habitat.sh` address and a node address range that doesn't overlap the SaaS service ranges.
See [deployment models](/360/latest/chef_360_product/deployment_models/) and the [SaaS system requirements](/360/saas/system_requirements/).

Chef 360 Platform has four core areas.
Node Management enrolls nodes and tracks their identity, status, and installed skills.
Chef Courier runs ad hoc or scheduled jobs across nodes.
Declarative State Management (DSM) stores the cookbooks, policies, and node metadata that nodes retrieve, and it's the part that replaces Chef Infra Server.
Platform services provide identity, access, and the web interface.

Installing and running Chef 360 Platform requires the right product entitlement.
Check the [system requirements](/360/latest/reference/system_requirements/) before you install.

## Bring nodes under management

A node is any server Chef manages.
How you enroll it depends on where it runs and what access you have.

| Where the node runs | How to enroll it |
| --- | --- |
| A server you can reach over SSH or WinRM | `knife bootstrap` installs Chef Infra Client and registers the node. Chef 360 can also connect over SSH or WinRM with the credentials you give it, one node at a time or from a CSV file. |
| A cloud virtual machine built from an image | Build Chef Infra Client into the image, or install it with user data or another first-boot script. The documentation includes unattended-bootstrap examples. |
| Any node where you don't want to hand over credentials | Generate a Chef 360 self-enrollment key and run the generated script on the node, or use the generated cookbook. |
| A Linux server you can't or don't want an agent on | Use Agentless Mode over SSH. |

For the first row, `knife bootstrap` looks like this:

```shell
knife bootstrap <ADDRESS> -U <USERNAME> --sudo --chef-license-key <LICENSE>
```

Replace the placeholders with the node's address, an account that can use sudo, and your Chef license ID.
By default the first run has an empty run list, so add `--run-list`, or a policy with `--policy-name` and `--policy-group`, to apply code on the first run.
Bootstrapping is validatorless by default, which means each node gets its own key instead of sharing one.
See [bootstrap a node](/client/latest/install/bootstrap/) and [`knife bootstrap`](/workstation/latest/tools/knife/knife_bootstrap/).

For cloud virtual machines, the documentation states that you can build Chef Infra Client into an image that starts on boot, or install it using user data or another post-deployment script.
The way you pass user data to a server differs between platforms, so use your cloud provider's method.
Settings such as `chef_server_url` and proxy settings must be built in, so build them into the image or the script.
See [unattended bootstrapping](/client/latest/install/bootstrap/#unattended-bootstrapping).

### Enroll nodes into Chef 360 Platform

With Chef 360 Platform, you enroll nodes into Node Management in one of four ways:

| Method | How it works |
| --- | --- |
| Self-enrollment script | You generate a self-enrollment key and run the generated script on the node. The node enrolls itself, so Chef 360 doesn't need credentials for it. |
| Cookbook-based enrollment | You use the same key to generate a Chef Infra cookbook that enrolls nodes through their existing Chef Infra Client runs. |
| Single-node enrollment | Chef 360 connects to one node over SSH or WinRM with the credentials you give it. |
| Bulk enrollment | You upload a CSV file of nodes and credentials, and Chef 360 enrolls each one. |

A node cohort sets the skills and settings that every node in it receives.
It can restrict membership by IP range, CIDR block, or MAC address, and it can require an administrator to approve each node before enrollment completes.
Self-enrollment keys can be limited the same way, and they can expire.
That gives you a way to treat locations differently: you can create a cohort for the address range of your data center and another for each cloud network, so a node can only join the cohort that matches where it runs.
See [node enrollment](/360/latest/use/operators/node_enrollment/) and [node management](/360/latest/use/operators/node_management/).

The documentation doesn't describe a separate enrollment procedure for each cloud.
You run the self-enrollment script on the node, so it fits into whichever first-boot step you already use.

### Nodes without an agent

Agentless Mode runs Chef Infra Client against a node over SSH, without installing Chef Infra Client on the node.
That suits remote systems and edge devices, and servers where you can't or don't want an agent.
You describe each target in a `target_credentials` file and run:

```shell
chef-client -t <TARGET_NAME>
```

Every resource in the cookbook must support Agentless Mode, and the documentation lists Linux as the supported platform.
See [Agentless Mode](/client/latest/features/agentless/) for the supported resources and how to schedule runs.

## Use one policy across every location

Changing one cookbook can change every node that runs it.
A Policyfile limits that by locking a run list and cookbook versions together into one versioned revision.
A policy group, such as `dev`, `stage`, or `prod`, binds a set of nodes to a specific revision.

This is the part that keeps a hybrid fleet manageable.
Policy groups describe your environments, such as development and production.
They don't describe your locations.
A node in your data center and a node in a cloud can be in the same policy group and run the same revision, and the recipe handles the difference in location, as the example does.

The workflow from your workstation is the same for every location:

```shell
chef install
chef push dev Policyfile.lock.json
```

`chef install` finds a compatible set of cookbooks, builds the run list, and writes `Policyfile.lock.json`.
`chef push` uploads that file and its cookbooks to the central service and applies it to the policy group you name, here `dev`.
You test the revision in `dev`, then push the same lock file to the next group.
Nodes in `prod` don't see the change until you promote it.
Nodes join a policy group when you bootstrap them, with `--policy-name` and `--policy-group`.
See [Policyfiles](/policyfile/) and [policy groups in DSM](/360/latest/use/operators/dsm_operations/#policy-groups).

## Roll changes out in controlled waves

Policy groups control which revision a node runs.
Chef Courier controls when and in what order a coordinated change reaches nodes.

A job defines the target nodes, the steps, a schedule, and execution rules, including how many nodes run at once (`batchSize`), what must succeed before it continues (`successCriteria`), and whether it retries a failed step.
You can validate a job with a dry run before you submit it, and each run's results are recorded and can be reviewed.

```json
"batchSize": { "type": "number", "value": 1 },
"successCriteria": [
  { "numRuns": { "type": "percent", "value": 100 }, "status": "success" }
]
```

This fragment comes from the job template reference.
It sets a batch size of one node and requires 100 percent of runs to succeed.
See [job execution](/360/latest/use/operators/job_execution/) for the full format.

To aim a job at the right nodes in a hybrid estate, use what Node Management gives you:

- **Tags.** A tag is a key and a value on a node.
  The documentation's example is tagging nodes as development, staging, and production, so you roll a change out to development and staging before production.
  You can use the same idea for your own groupings, such as a location tag.
  You tag one node at a time in the interface.
- **Node filters.** A filter is a search expression over node attributes, skills, enrollment level, and health.
- **Node lists.** You select nodes, save them as a list, and create a Courier job from it.
  A job can target nodes, a filter, or a saved list.

Use both together: promote cookbook changes through policy groups, and use Courier for coordinated, one-time work such as a package update that needs ordering.

## See the whole fleet in one place

After each run, Chef Infra Client reports a summary of what it did.
That's true whether the node is in a data center or a cloud, so the reports from every location can land in the same place.

- **Chef Automate** collects the reports and shows the status of your nodes and their Chef Infra Client runs.
  Nodes send them with the data collector, which you set up with `data_collector.server_url` and a token in `client.rb`.
  See [data collection](/automate/data_collection/) and [client runs](/automate/client_runs/).
- **Chef Automate node integrations** discover nodes in your cloud accounts.
  The documentation describes node managers for AWS EC2, Azure, and Google Cloud Platform.
  For EC2, the node manager creates a node reference for each instance and collects the tags on it, and it polls the account for instance state every two hours.
  You can filter the nodes it finds by those tags, and scan them with Chef InSpec.
  See [node integrations](/automate/node_integrations/) and [nodes](/automate/nodes/).
- **Chef 360 Platform** shows every enrolled node on the Nodes page, with its operating system, installed skills, enrollment level, and health.
  You can filter and save node lists, and export the table to a CSV file.
  Courier keeps the history of every job run.
  See [node management](/360/latest/use/operators/node_management/).

Which of these you use depends on the central service you chose, and the documentation doesn't describe a single combined dashboard that merges Chef Automate and Chef 360 Platform.
If you use both, plan which one answers which question.

## Keep compliance consistent

A fleet that's configured the same way should be checked the same way.
The Compliance Phase adds a security check to every Chef Infra Client run.
When you turn it on, Chef Infra Client runs Chef InSpec profiles after it converges and reports the results with the run:

```ruby
default['audit']['compliance_phase'] = true
```

You also set which profiles to run, with the `node['audit']['profiles']` attribute.
Because that's a recipe or attribute change, it rolls out through the same policy groups as any other change, and it applies in every location.
See [the Compliance Phase](/client/latest/features/chef_compliance_phase/).

Compliance for the cloud accounts themselves is a separate check from compliance for the servers in them.
Chef InSpec has resource packs for [AWS](/inspec/resource_packs/aws/), [Azure](/inspec/resource_packs/azure/), and [Google Cloud](/inspec/resource_packs/gcp/), and Chef Automate can run scan jobs against those accounts.
For the checks and the steps, see these Chef Compliance pages:

- [The Chef way of AWS compliance](/solutions/compliance/aws_foundations/)
- [The Chef way of Azure compliance](/solutions/compliance/azure_foundations/)
- [The Chef way of GCP compliance](/solutions/compliance/gcp_foundations/)
- [The Chef way of virtual machine compliance](/solutions/compliance/vm_compliance/)
- [The Chef way of Linux server compliance](/solutions/compliance/linux_server/)

Convergence and compliance answer different questions: convergence makes the node match your code, and compliance checks that the result meets a requirement.

## Migrate from Chef Infra Server

If you run Chef Infra Server, you can import its data into DSM in Chef 360 Platform instead of rebuilding.
The import brings across organizations, users, cookbooks, policies, roles, environments, data bags, and nodes.

The process in outline:

1. Clean up stale data on Chef Infra Server first, such as unused Policyfile revisions and nodes that haven't checked in for 60 days or more.
1. Export the data with the `knife-ec-backup` tool, and compress the backup.
1. Upload the backup with the `chef-import-cli` tool, and monitor the import until it finishes.
1. Verify the organizations, nodes, cookbooks, roles, environments, and data bags in DSM.
1. Redirect nodes by updating DNS, or by setting `chef_server_url` in `/etc/chef/client.rb` on each node, so they check in with Chef 360 Platform at their next run.

In a hybrid estate, step 5 reaches nodes in every location, so check that the nodes in each cloud and data center can reach the new address before you redirect them.

Before you start, check these prerequisites: Chef Infra Server 12 or later, Chef 360 Platform 1.6.1 or later, and administrative access to the server you're exporting from.
The documentation also says unsupported objects can leave an import only partly successful, so verify the result.
See [Migrate from Chef Infra Server](/360/latest/migrate/on_prem_to_self_managed/) for the commands and the troubleshooting steps.
If you use Chef 360 SaaS instead, the process is different, so follow [Migrate to DSM in Chef SaaS](/360/latest/migrate/on_prem_to_saas/).

## Run in restricted networks

Some locations have limited internet access, and that's common in the data center side of a hybrid estate.
Plan for these points:

- Nodes that can't reach the internet need a different source for the Chef Infra Client package.
  See [install Chef Infra Client in an air-gapped environment](/client/latest/install/install_chef_air_gap/).
- For Chef Automate, see [air-gapped installation](/automate/airgapped_installation/).
- For nodes behind a proxy, set the proxy settings that Chef Infra Client reads.
  See [proxies](/client/latest/install/proxies/).
- Check the system requirements for the ports that nodes must open.

## What the documentation doesn't say

This page sticks to what the Chef documentation states.
A few things that people often assume aren't covered, so confirm them before you rely on them:

- **Cloud certification.** The supported platforms list names operating systems, not cloud providers. Chef Infra Client runs on a supported operating system wherever it runs, and the cloud helper methods detect where that is. See [supported platforms](/platforms/).
- **A cloud-by-cloud reference architecture.** The documentation doesn't include one, so the network and account design is yours.
- **Cloud-specific enrollment for Chef 360.** The enrollment methods above are generic, and the documentation doesn't describe a separate guide for each cloud.
- **Provisioning.** Chef configures servers. Use your cloud's own tools, or other tools, to create them.

## Where to start

If you're new to Chef, start with the example on this page, and then read [the Chef Infra Client run](/client/latest/#the-chef-infra-client-run).
If you run an estate today, look at [Connect every node to one central service](#connect-every-node-to-one-central-service) and, if you use Chef Infra Server, [Migrate from Chef Infra Server](#migrate-from-chef-infra-server).
Before you scale up:

- Pick the central service first, and check that every location can reach it.
- Keep values that differ between locations in attributes, not in recipes, and test every branch.
- Use policy groups for environments, such as development and production, and let the recipe handle location.
- Give every node an owner and an environment, and put nodes into policy groups or cohorts that match.
- Turn on the Compliance Phase for the nodes that have a security baseline to meet.

## Related documentation

- [Chef Infra Client](/client/latest/)
- [Chef Workstation](/workstation/latest/)
- [Chef 360 Platform](/360/latest/)
- [Declarative State Management](/360/latest/understand/dsm/)
- [Chef Automate](/automate/)
- [Supported versions](/versions/)
- [Chef platform overview](/platform_overview/)
- [Chef Compliance overview](/solutions/compliance/)
- [The Chef way of managing an on-prem fleet](/solutions/compliance/on_prem_fleet/)
