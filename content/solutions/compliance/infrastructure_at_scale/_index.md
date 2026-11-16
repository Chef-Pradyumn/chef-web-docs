+++
title = "The Chef way of managing infrastructure at scale across many application teams"
description = "Coordinate platform standards and application-owned configuration across many teams with Chef: define ownership boundaries, reuse versioned cookbooks, separate access, promote Policyfiles, and track compliance across the fleet."
draft = false
gh_repo = "chef-web-docs"

[menu.overview]
  identifier = "overview/solutions/compliance/infrastructure_at_scale"
  parent = "overview/solutions/compliance"
  title = "Managing infrastructure at scale"
  weight = 13
+++

When many application teams manage servers independently, configuration becomes difficult to govern.
Teams may use different cookbook versions, make competing changes to shared settings, or lack a clear owner for production access.
A central team that reviews every server change can become a bottleneck, while unrestricted team access can create security and reliability risks.

Chef lets platform teams define shared standards and application teams manage their own configuration within those boundaries.
Cookbooks and Policyfiles describe the desired state, Chef Infra Client applies it on nodes, and Chef 360 Platform's Declarative State Management (DSM) provides centralized policy and cookbook management.
Use Chef Courier to coordinate runs and rollouts, and the Compliance Phase to report whether the controls remain in effect.

This page focuses on the operating model for teams working at scale.
For a hands-on single-team example, see [The Chef way of managing a small fleet](/solutions/compliance/small_fleet/).
For the basic node convergence model and platform deployment choices, see
[The Chef way of managing an on-prem fleet](/solutions/compliance/on_prem_fleet/).

## What changes as teams scale

At scale, the challenge extends beyond writing a recipe.
Teams need clear ownership for each part of the desired state, the nodes it applies to, and the path a change takes to production.

| Concern | Platform team | Application team |
| --- | --- | --- |
| **Baseline** | Defines organization-wide requirements, such as approved accounts, logging, and security settings | Consumes the approved baseline and identifies application-specific requirements |
| **Configuration code** | Maintains shared cookbooks and reusable policy patterns | Owns application cookbooks and the application's Policyfile |
| **Access** | Sets up organizational boundaries, roles, and least-privilege policies | Gets only the access needed to develop and promote its configuration |
| **Release** | Defines testing, review, and promotion standards | Tests changes and promotes the application policy through its rollout groups |
| **Operations** | Monitors compliance and shared services | Resolves application-specific convergence and compliance failures |

These roles describe a recommended operating model, not required Chef 360 Platform roles.
Map them to the people and permissions your organization already uses.

## Choose team boundaries before onboarding nodes

Chef 360 Platform and DSM organize access within different boundaries.
Similar names in the two services don't grant cross-service access.

### Chef 360 Platform organizational units

Chef 360 Platform tenants can contain one or more organizational units.
An organizational unit provides a logical boundary for users and the entities they manage.
Users can hold different roles in different units, and roles combine policies to grant specific platform actions.

Use a separate organizational unit when teams need distinct access to Chef 360 Platform resources or administration.
Keep shared platform operators in the units they need to operate, and grant application teams only the roles and actions required for their work.
Read the [organizational unit guidance](/360/latest/use/administrators/organizations/manage_organizational_units/),
[user administration documentation](/360/latest/use/administrators/users_and_roles/user_management/), and
[role administration documentation](/360/latest/use/administrators/users_and_roles/role_management/).

### Configuration boundaries

DSM organizations scope Chef Infra objects, including cookbooks, nodes, roles, environments, and Policyfiles.
DSM administrators configure group permissions at the organization, object-type, and individual-object levels.

Use separate DSM organizations when application teams need stronger isolation between configuration objects, permissions, or release schedules.
Use one shared DSM organization when teams share a trust boundary and administrators can manage object permissions.
DSM doesn't share cookbooks across organizations automatically.
Publish and version shared cookbooks in each organization where policies need them.
Read [DSM access control](/360/latest/use/administrators/organizations/dsm_organizations/) and
[DSM operations documentation](/360/latest/dsm/).

{{< note >}}

Chef 360 Platform organizational units and DSM organizations remain independent.
Chef 360 Platform Node Management and DSM maintain separate node records.
Plan enrollment, user access, and Chef Infra Client registration for both services as needed.

{{< /note >}}

## What you'll build

A team operating model with three layers:

- A **platform baseline** cookbook owned by the platform team, with shared settings and controls.
- An **application cookbook and Policyfile** owned by each application team, with application-specific resources and the version of the baseline it uses.
- **Environment policy groups** such as `payments-dev`, `payments-stage`, and `payments-prod`, which let each application promote its locked policy through controlled stages.

You won't put every team's configuration in one enormous cookbook or give every team unrestricted write access to shared production policies.
Instead, each policy has a clear owner, a review path, and a known set of nodes.

## Before you begin

You need:

- A Chef 360 Platform tenant and the organizational units and DSM organization or organizations that fit your access model.
- **Chef Workstation** for contributors who build and test cookbooks and Policyfiles.
- **Chef Infra Client registered with DSM** on managed nodes. Chef 360 Platform Node Management enrollment for Courier doesn't register Chef Infra Client with DSM.
See [The Chef way of managing a small fleet](/solutions/compliance/small_fleet/) for the client configuration prerequisite.
- **A Git repository** with branch protection, required reviews, and CI checks enabled for cookbook and Policyfile changes.
- **A test environment** with representative operating systems and application dependencies.
- **A change owner** for shared baseline releases and a technical owner for each application policy.

## Step 1: Define the ownership model

Before teams publish policies, write down who can change shared configuration and who can promote application releases.

| Area | Recommended owner | Boundary to define |
| --- | --- | --- |
| Shared baseline cookbook | Platform team | Accepted versions, supported platforms, release cadence, and deprecation notice |
| Application cookbook | Application team | Repository, maintainers, service dependencies, and test coverage |
| Policyfile | Application team, reviewed by platform team | Cookbook versions, run list, and required baseline version |
| DSM organization | DSM administrators | Which teams can read, upload, update, and promote objects |
| Chef 360 Platform organizational unit | Tenant administrators | Which users can perform platform actions for each unit |
| Production rollout | Application owner with platform guardrails | Canary node, batch size, approval requirements, and rollback owner |

Keep the roles separate where it reduces risk, but avoid a process that requires the platform team to approve routine application changes manually.
Automate required tests and policy checks so review focuses on intent and exceptions.

## Step 2: Publish a shared baseline cookbook

The platform team owns a versioned cookbook for settings every application node must share.
Keep it focused on organization-wide controls, such as approved service accounts, logging configuration, endpoint protection, and security agents.
Don't put application deployment details in the baseline.

Release the cookbook through the same review and test path as application code:

1. Review the change in version control.
2. Run Cookstyle and Test Kitchen on the supported platforms.
3. Publish a tagged cookbook release to the DSM organization that will consume it.
4. Announce breaking changes and the last supported version before removing or changing a shared resource.

Application teams should consume a released cookbook version, not an unreviewed branch or a mutable path.
See [Policyfile](/client/latest/policy/policyfile/) for how a lock file records the resolved cookbook versions.

## Step 3: Give each application team an owned policy

Each application team maintains its own cookbook and Policyfile.
The Policyfile includes the approved shared baseline cookbook and the team's application cookbook, and defines their ordered run list.
Keep application-specific values in attributes or Policyfile overrides rather than forking the baseline cookbook.

For example, the run list for a payment service includes:

```ruby
name 'payments_service'

default_source :chef_server, 'https://<DSM_HOST>/organizations/<DSM_ORG>'

run_list 'recipe[org_baseline::default]',
         'recipe[payments_service::default]'

cookbook 'org_baseline', '= 3.4.1'
cookbook 'payments_service', path: '.'
```

This example assumes the baseline cookbook exists in the target DSM organization.
Replace `<DSM_HOST>` and `<DSM_ORG>` with your DSM endpoint and organization, and configure Chef Workstation to authenticate to that organization.
The `Policyfile.lock.json` produced by `chef install` records the exact revisions used by the policy.
Commit the Policyfile and lock file together so reviewers can see both the intended run list and its resolved dependencies.

If teams share one DSM organization, configure cookbook and policy permissions for each team through DSM groups and object ACLs.
If teams use separate DSM organizations, publish the shared baseline into each organization and pin its version there.
See [DSM role-based access control](/360/latest/use/administrators/organizations/dsm_organizations/).

## Step 4: Test changes before promotion

Every application team should test its own policy with the shared baseline version it intends to deploy.
Use Test Kitchen or a comparable CI test environment with representative supported platforms.
Test:

- The application cookbook converges successfully on a clean node.
- A second converge is idempotent and reports no unexpected changes.
- The controls in the team's Chef InSpec profile pass after convergence.
- The policy lock file records the baseline cookbook version the team intends to use.

Require review for changes to shared cookbook versions, production policy groups, and compliance controls.
Store test output with the commit or CI run so a release has an audit trail.
For a complete single-team walkthrough, see [The Chef way of managing a small fleet](/solutions/compliance/small_fleet/).

## Step 5: Promote each policy through environments

Use distinct policy groups for each application's deployment stages.
For example, a payment application can use `payments-dev`, `payments-stage`, and `payments-prod`, while an inventory application uses its own groups.
A policy group binds nodes to a specific Policyfile revision, so promotion moves a tested revision rather than rebuilding a different policy for production.

A typical promotion path is:

1. Publish the tested revision to the application's development policy group.
2. Converge representative nodes and review Chef Infra Client and compliance results.
3. Promote the same revision to staging and check integrations and operational requirements.
4. Promote the same revision to production after the required approval.
5. Roll back by assigning the last known-good revision if the release causes an incident.

Use the Chef DSM CLI or API to manage policy groups and Policyfiles.
The Chef 360 Platform UI displays policy and group details but doesn't create or update them.
See [Policy groups and Policyfiles in DSM](/360/latest/dsm/).

## Step 6: Coordinate large rollouts with Chef Courier

Chef Courier can target specific node IDs or filters, and can sequence or batch work across target groups.
Use it for coordinated runs, maintenance tasks, and changes that need controlled rollout or run evidence.
Keep policy promotion in DSM: Courier schedules Chef Infra Client runs, while the node's assigned policy name and group determine which configuration it retrieves.

For a production rollout, use separate target groups for a canary, a small validation batch, and the remaining production nodes.
Set success criteria so the next group starts only after the previous group meets its criteria.
Choose a batch size and timeout that match the service's failure tolerance and expected converge duration; test those values in a non-production group first.
See [Chef Courier jobs](/360/latest/use/operators/job_execution/) and the
[Courier job template reference](/360/latest/use/operators/job_execution/job_template/).

Don't use a single tenant-wide job targeting every node for application-specific changes.
Filter or list only the nodes that belong to that application and environment.
Review the target set before enabling a recurring schedule.

## Step 7: Review shared controls and team-owned outcomes

Enable the Compliance Phase for controls that should run with each Chef Infra Client converge.
Use Chef InSpec profiles for application requirements and shared profiles for platform baselines.
Where teams share a Chef Automate instance, use profile, control, node, and environment filters to distinguish results and route findings to the owning team.
See [Chef Infra Client Compliance Phase](/client/latest/features/chef_compliance_phase/) and
[Compliance reporting in Chef Automate](/automate/reports/).

Define ownership for each class of failure:

- **Shared baseline failure**: platform team owns the cookbook or standard.
- **Application policy failure**: application team owns its recipe, dependency, and rollout.
- **Node-specific drift**: the node owner investigates the local change, while Chef Infra Client restores managed resources on the next run.
- **Accepted exception**: the control owner records a justified, time-bounded waiver and tracks its expiry.

## Operating principles

- **One owner for each policy.** Every Policyfile and production group has a named team responsible for releases and rollback.
- **Share code; pin releases.** Reuse baseline cookbooks, but lock each application policy to a reviewed version.
- **Separate identities from nodes.** Assign human users least-privilege roles. Use Chef Infra Client credentials for DSM requests, and Chef 360 Platform Node Management enrollment for Courier.
- **Promote one revision.** Move the same tested Policyfile revision through stages; don't rebuild a different release at each stage.
- **Keep production targets explicit.** Use application and environment node groups or filters, not broad tenant-wide targets.
- **Measure compliance by ownership.** Route shared baseline findings to the platform team and app-specific findings to the application team.

## Where to go next

**Build the shared baseline.**
Use [The Chef way of Linux server compliance](/solutions/compliance/linux_server/) or
[The Chef way of configuring Windows servers](/solutions/compliance/windows_server/) for platform-specific controls.

**Plan rollout mechanics.**
See [The Chef way of managing an on-prem fleet](/solutions/compliance/on_prem_fleet/) for convergence, rollout waves, and deployment options.

**Review the single-team implementation.**
See [The Chef way of managing a small fleet](/solutions/compliance/small_fleet/) for an end-to-end cookbook and Policyfile example.

## Related

- [Compliance solution overview](/solutions/compliance/)
- [User administration in Chef 360 Platform](/360/latest/use/administrators/users_and_roles/user_management/)
- [Role and policy administration in Chef 360 Platform](/360/latest/use/administrators/users_and_roles/role_management/)
- [Chef 360 Platform organizational units](/360/latest/use/administrators/organizations/manage_organizational_units/)
- [DSM role-based access control](/360/latest/use/administrators/organizations/dsm_organizations/)
- [DSM operations documentation](/360/latest/dsm/)
- [Chef Courier job template reference](/360/latest/use/operators/job_execution/job_template/)
- [Policyfile](/client/latest/policy/policyfile/)
- [Chef Infra Client Compliance Phase](/client/latest/features/chef_compliance_phase/)
