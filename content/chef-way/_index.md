+++
title = "The Chef Way"
draft = false
st_robots = "noindex, follow"
toc = false
+++

Chef has 15+ years of history—which means multiple ways to do almost everything. Every task page in these docs can label its guidance with one of six tags below, so you always know whether what you're reading is current best practice or something older we still support.

- ✓ **Chef Way**—the current, recommended approach. If you do nothing else, follow this.
- ? **Why**—the reasoning behind a recommendation, so you can judge when it doesn't apply to you.
- ⇄ **Alternative**—a different supported approach with its own tradeoffs—not wrong, just situational.
- ◷ **Legacy**—still works, still supported, but superseded. New builds shouldn't start here.
- ✕ **Avoid**—a known anti-pattern. Technically possible, reliably causes pain later.
- → **Migration**—the concrete path from a Legacy or Avoid pattern to the current Chef Way.

## Worked example—distributing cookbooks

{{% chef_way label="way" %}}
Use Chef 360 Platform's Compass service to distribute Policyfiles to nodes.
{{% /chef_way %}}

{{% chef_way label="why" %}}
Compass gives you drift detection and rollback that a bare Infra Server can't.
{{% /chef_way %}}

{{% chef_way label="alternative" %}}
Self-hosted Chef Infra Server, if you require an air-gapped environment.
{{% /chef_way %}}

{{% chef_way label="legacy" %}}
Chef Infra Server 14 with `knife upload`—supported through end-of-life.
{{% /chef_way %}}

{{% chef_way label="avoid" %}}
Manually `scp`-ing cookbooks to nodes outside of any policy system.
{{% /chef_way %}}

{{% chef_way label="migration" %}}
See the Infra Server to Chef 360 migration guide in each product's Upgrade section.
{{% /chef_way %}}

Looking for a specific product instead? See [how these docs are organized](/start-here/).
