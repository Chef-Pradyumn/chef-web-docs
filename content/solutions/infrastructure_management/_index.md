+++
title = "Chef Infrastructure Management"
draft = false
gh_repo = "chef-web-docs"

[cascade]
  [cascade.params]
    swiftype_search_products = ["client", "workstation", "server", "automate"]
    section_root = "/solutions/infrastructure_management"
    breadcrumbs = true

[menu.overview]
  identifier = "overview/solutions/infrastructure_management/overview"
  parent = "overview/solutions/infrastructure_management"
  title = "Overview"
  weight = 1
+++

Chef Infrastructure Management keeps your systems in the state you declare ---
packages, files, services, users, and settings --- and brings them back when
they drift, across cloud, on-premises, and hybrid environments, wherever your fleet runs.

## Why infrastructure drifts between changes

A fleet drifts away from its intended configuration between planned changes:
a manual fix applied during an incident, a package updated on one system but not
the next, a new system built from an old image.
Configuring systems by hand doesn't scale past a handful of hosts, and a runbook
describes what someone meant to do, not what's actually on the system.
Repeating the same change across hundreds of systems, by hand, produces inconsistent
results even when every individual step was correct.

## What this solution implements

A Chef Infra cookbook describes the state each system should be in, as resources
such as files, packages, and services.
Chef Infra Client compares each system with that description and changes only what
doesn't match, so the same policy that built a system also corrects it when it drifts.
Run on a schedule, drift is corrected on the next run instead of being discovered during
the next incident.

Start here:

- **New to Chef Infra?**
  The [Chef Infra quickstart guide](/client/19/quickstart/)
  generates a repository, writes a recipe, and runs it in local mode, with the commands at each step.
- **Developing a cookbook on one system?**
  [Develop and test a cookbook locally](#develop-and-test-a-cookbook-locally)
  covers local mode without a central server.
- **Have a different environment or problem in mind?**
  See [common infrastructure scenarios](#common-infrastructure-scenarios)
  for cloud, on-premises, hybrid fleets, and more.

## Getting Chef Infra

Configuration policy reaches your systems through the same three steps,
whether you're writing your first cookbook or bringing a new environment
under management.

**Authoring.**
Policy is written on a workstation with
[Chef Workstation](/workstation/latest/), which provides
the tools to generate a Chef Infra repository and cookbooks and to test them before
they reach a managed system.

**Distributing.**
Cookbooks reach a system either from a local repository, using local mode,
or from a central Chef Infra Server that each node synchronizes with in the
traditional client-server workflow.

**After you install it.**
Chef Infra Client runs on each managed system and converges it toward the policy
assigned to it:
it builds the resource collection, checks the current state of each resource,
and changes only what differs.
Run it once to bring a system into line,
then run it again to confirm nothing changes,
and schedule it so the configuration stays that way.

## Infrastructure configuration

Infrastructure configuration describes what each system should look like,
and gives Chef Infra Client what it needs to make the system match.

### What gets managed

Chef Infra manages the configuration your systems actually depend on:

- Packages and software installation
- Files, directories, and templates
- Services and their running state
- Users, groups, and permissions
- Operating system settings and scheduled tasks
- Application hosts in cloud, on-premises, and hybrid environments

Each item is expressed as a [resource](/client/19/resources/), which describes the
desired state of a configuration item and the steps to reach it.
Resources are declared in order in recipes, and recipes are grouped into a
[cookbook](/client/19/cookbooks/) ---
see [Develop and test a cookbook locally](#develop-and-test-a-cookbook-locally)
to write and run one.

### How you run Chef Infra Client

How you run Chef Infra Client depends on how policy reaches the system,
and you can use more than one approach in the same fleet.

| Approach | Fits | How it runs |
| --- | --- | --- |
| [Local mode](/client/19/reference/ctl_chef_client/#run-in-local-mode) | Cookbook development, testing, and single systems | Runs against a local repository with `--local-mode`, without a central server |
| [Client-server](/server/) | Fleets that take policy from Chef Infra Server | Each node authenticates, synchronizes its cookbooks, and converges locally |
| [Recurring runs](/client/19/reference/ctl_chef_client/#run-as-a-service) | Holding configuration in place over time | Runs as a daemon or on a schedule, correcting drift on each run |

### Results and evidence

Each Chef Infra Client run reports which resources were updated and which were
already up to date, so a run that changes nothing is evidence that the system
matches policy.
In the traditional workflow, nodes can report node data back to the server, and
[Chef Automate](/automate/) gives you a view of
run results across the fleet.

To see which environments and building blocks this solution covers, see
[supported environments and building blocks](#supported-environments-and-building-blocks).

## Convergence and drift

A recipe tells Chef Infra Client what a system should look like.
Convergence is the process of making it look that way, on every system that needs it.
Because each resource checks the current state before acting, re-running an unchanged
recipe on a system that's already correct changes nothing --- which is what
makes it safe to run on a schedule instead of as a one-off fix.

### How resources reconcile state

Every resource states what it manages and the action to take.
This example from the [quickstart](/client/19/quickstart/) uses the
`file` resource to keep a file's content in the declared state:

```ruby
file "#{ENV['HOME']}/test.txt" do
  content 'This file was created by Chef Infra!'
end
```

**The first run and every run after it aren't the same.**
On the first run, Chef Infra Client creates the file.
On later runs, it reports the resource as up to date and leaves the file alone ---
unless someone has changed the content, in which case it restores it.

### Before you apply policy to production

- **Develop locally first.**
  Local mode lets you run a cookbook on one system without a central server.
  See [Develop and test a cookbook locally](#develop-and-test-a-cookbook-locally).
- **Test before production.**
  Cookbooks change system configuration.
  Test them the way you'd test any code, with
  [Chef Workstation](/workstation/latest/).
- **Manage cookbooks as code.**
  Keep cookbooks in version control, review every change before it's merged,
  and assign an owner to each cookbook so changes to production systems are traceable.
- **Roll out in stages.**
  Apply a new or changed cookbook to a small set of systems, review what changed,
  and then expand to the rest of the fleet.
- **Check product lifecycle status.**
  Chef Infra Server is deprecated and reaches end of life in November 2026.
  [Chef 360 Platform](/360/latest/) is the current Chef platform for
  infrastructure operations, including Declarative State Management.
  Read the [Chef Infra Server overview](/server/) and contact your
  Chef account representative before you plan a client-server deployment.
- **Review licensing.**
  You must accept the Chef End User License Agreement before you use Chef products,
  and a Chef license key is required for some workflows, such as downloading binaries
  from the Chef Download Portal.
  See [About Chef licenses](/licensing/).
- **Expect the first run to change things.**
  The first run on an existing system corrects everything that differs from policy.
  Review what a cookbook manages before you assign it to a system that's already in service.

## Common infrastructure scenarios

Each scenario below names the problem, the approach, and where to start.

If you haven't run Chef Infra Client before,
the [Chef Infra quickstart guide](/client/19/quickstart/)
is the best end-to-end starting point.
It covers generating a repository, writing a recipe, and running it in local mode.
Those mechanics are the same whichever scenario you're working on,
so it's worth doing once even if your actual problem is elsewhere.

### Standardize configuration across cloud environments

**The problem.**
Your application hosts run in the cloud, and each one is built and patched a little
differently, so no two hosts match by the time they're in service.

**The Chef Way.**
Describe the baseline configuration of an application host once, as a cookbook,
and run Chef Infra Client on every host so each one converges to that baseline.
Run it on a schedule so service health and configuration stay consistent as hosts
are replaced.

**Start here.**

1. List what every application host needs: packages, configuration files, and services.
1. Express each item as a [resource](/client/19/resources/) in a recipe.
1. Run the recipe on one host in local mode and read the output before scaling up.
1. Run Chef Infra Client on every host that should share the baseline.
1. Schedule recurring runs so new and replaced hosts converge without anyone configuring them by hand.

### Manage on-premises systems consistently

**The problem.**
Systems in your own facilities have been configured over years by different teams,
and nobody can say with confidence what's actually on each one.

**The Chef Way.**
Capture the intended configuration of each class of managed host in cookbooks,
with clear ownership for each one.
Chef Infra Client converges every host to its policy, and a run that reports no
changes tells you the host already matches.

**Start here.**

1. Group your managed hosts by role, and agree who owns the policy for each role.
1. Write a [cookbook](/client/19/cookbooks/) for each role,
   starting with the configuration that matters most.
1. Test each cookbook with [Chef Workstation](/workstation/latest/) before it reaches a production host.
1. Run Chef Infra Client on a small set of hosts and review what changed.
1. Expand to the rest of the role once the results are what you expect.

### Apply one configuration approach across a hybrid fleet

**The problem.**
Your fleet spans cloud and on-premises environments.
Each has its own tools and, today, its own process for configuration.

**The Chef Way.**
Use the same cookbooks for the same roles wherever they run, so shared configuration
intent is written once.
Keep environment-specific differences explicit in policy instead of in someone's memory.

**Start here.**

1. Identify which configuration is shared across environments and which is genuinely environment-specific.
1. Put the shared configuration in cookbooks that every environment uses.
1. Confirm network, identity, and workload compatibility for each environment separately; a cookbook doesn't validate them for you.
1. Run Chef Infra Client in each environment and compare results side by side.
1. Schedule recurring runs in every environment so both stay consistent with the same policy.

### Recover systems that drift from policy

**The problem.**
The same handful of settings keep changing on systems after they're configured:
an emergency change left in place, a file edited by hand, a service stopped and never restarted.

**The Chef Way.**
Declare those settings as resources, then let Chef Infra Client correct them.
Re-running on a schedule holds the configuration, because Chef Infra Client acts only
where a system has drifted.

**Start here.**

1. Identify which settings drift most often across your fleet.
1. Make sure each one is managed by a resource in the cookbook assigned to those systems.
1. Run Chef Infra Client and confirm the drifted resources are reported as updated.
1. Run it again and confirm every resource is reported as up to date.
1. Schedule recurring runs so drift is corrected on the next run, not the next incident.

### Develop and test a cookbook locally

**The problem.**
You want to write and try a cookbook without access to a central server,
and without affecting systems anyone else depends on.

**The Chef Way.**
Generate a Chef Infra repository with Chef Workstation, then run Chef Infra Client in
local mode, which uses local repository data and a local Chef Zero service instead of
a central Chef Infra Server.

**Start here.**

1. Install [Chef Workstation](/workstation/latest/), then
   generate a repository with an example cookbook:

   ```shell
   chef generate repo my_first_repo
   ```

1. Edit the example recipe to declare the resources you want to manage.
1. Run the example recipe in local mode:

   ```shell
   chef-client --local-mode --override-runlist example
   ```

   This changes the system it runs on.
   The quickstart example creates `$HOME/test.txt`.

1. Run the same command again and confirm the resource is reported as up to date.
1. See the [Chef Infra Client executable reference](/client/19/reference/ctl_chef_client/#run-in-local-mode)
   for the full set of local-mode options.

### Verify configuration against a benchmark

**The problem.**
Your systems match your cookbooks, but you've been asked to show they also
meet a security benchmark.

**The Chef Way.**
Use [Chef Compliance](/solutions/compliance/) to check
systems against CIS Benchmarks and DISA STIGs, and Chef Infra cookbooks to correct the
configuration behind any failure.

**Start here.**

1. Read the [Chef Compliance](/solutions/compliance/) solution overview.
1. Turn on the [Compliance Phase](/client/latest/features/chef_compliance_phase/) so assessment runs as part of each Chef Infra Client run.
1. Correct failing controls in your cookbooks, then reassess to confirm the result.

## Supported environments and building blocks

The same building blocks apply in every environment.
What changes is what you plan for in each one, and how policy reaches the system.

This solution covers **3 environment types**:

| Environment | What you plan for |
| --- | --- |
| [Cloud](#standardize-configuration-across-cloud-environments) | Application hosts, baseline configuration, and service health |
| [On-premises](#manage-on-premises-systems-consistently) | Managed hosts, configuration consistency, and operational ownership |
| [Hybrid](#apply-one-configuration-approach-across-a-hybrid-fleet) | Cloud workloads, on-premises workloads, and shared configuration intent |

Building blocks:

| Building block | Role |
| --- | --- |
| [Resource](/client/19/resources/) | Describes the desired state of one configuration item and the steps to reach it |
| [Recipe](/client/19/cookbooks/) | Declares resources in order |
| [Cookbook](/client/19/cookbooks/) | Contains recipes and supporting configuration |
| Run-list | Selects the recipes a system runs |
| [Chef Infra Client](/client/19/) | Converges each managed system toward its policy |
| [Chef Workstation](/workstation/latest/) | Authors and tests cookbooks before they reach a managed system |
| [Chef Infra Server](/server/) | Central hub for cookbooks, policy, and node data in the traditional workflow (deprecated; end of life November 2026) |
| [Chef 360 Platform](/360/latest/) | Current Chef platform for infrastructure operations, including Declarative State Management |

### How a Chef Infra Client run works

| Step | What happens |
| --- | --- |
| Load policy | The run-list selects the recipes to run |
| Compile resources | Chef Infra Client builds the ordered resource collection |
| Check current state | Each resource is compared with the current state of the system |
| Converge | Only the resources that differ are changed |

## Get started

**Understand the workflow.**
Read the [Chef Infra Client overview](/client/19/)
to see how a run expands the run-list, compiles resources, and converges the node.

**Plan your environments.**
Use the environment table above to decide what you're managing in cloud,
on-premises, and hybrid environments.

**Set up your workstation.**
Install [Chef Workstation](/workstation/latest/)
and generate a Chef Infra repository.

**Run your first cookbook.**
Work through the
[Chef Infra quickstart guide](/client/19/quickstart/)
to write a recipe and run it in local mode.
To manage systems at scale, run Chef Infra Client on every managed system and
schedule recurring runs.
If you're planning a client-server deployment, check the
[Chef Infra Server](/server/) lifecycle status and evaluate
[Chef 360 Platform](/360/latest/) first.

**Review your results.**
Read each run's output to see which resources were updated and which were
already up to date.

**Write your own resources.**
Browse the [resource reference](/client/19/resources/)
to find the resources that match the configuration you manage, and add them to your cookbooks.
