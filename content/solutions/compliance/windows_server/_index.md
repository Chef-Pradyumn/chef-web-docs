+++
title = "The Chef way of configuring Windows servers"
description = "Configure and secure Windows Server systems with Chef: manage registry keys, Windows features, firewall profiles, and services with Chef Infra Client, then validate the active state with Chef InSpec."
draft = false
gh_repo = "chef-web-docs"

[menu.overview]
  identifier = "overview/solutions/compliance/windows_server"
  parent = "overview/solutions/compliance"
  title = "Configuring Windows servers"
  weight = 12
+++

<!-- cspell:ignore LanmanServer WinStations RemoteRegistry SMB1Protocol winrm -->

Windows Server systems across data centers and cloud accounts require strict, repeatable baseline configurations.
In an unmanaged environment, administrators often configure settings by hand using Server Manager,
local Group Policy editors, or one-off PowerShell scripts.
Over time, those manual changes produce snowflake servers:
each system drifts from the intended standard, patch cycles introduce inconsistent defaults,
and security teams lack auditable evidence showing which controls are active.

Chef replaces manual changes with code that you test, version, and enforce automatically.
Use Chef Infra Client to describe your desired system state---from registry values and Windows features
to firewall profiles and services---and converge each server to match it.
Use Chef InSpec to audit those same settings and report failing controls.
When paired with Chef 360 Platform, you can publish locked configuration baselines to
Declarative State Management (DSM) and schedule automated converges using Chef Courier.

If you haven't written a Chef InSpec profile before, start with
[The Chef way of SSH compliance](/solutions/compliance/ssh_baseline/).

## What Windows server configuration covers

Windows Server configuration spans four layers.
Each layer addresses a different aspect of the operating system,
and a server can satisfy one layer while remaining vulnerable in another.

| Layer | What you're asking | How Chef handles it |
| --- | --- | --- |
| **System and security settings** | Are protocol settings, Remote Desktop encryption, and system policies configured? | The `registry_key` resource for registry values and `security_policy` for local user rights |
| **Roles and features** | Are required roles installed and legacy or vulnerable features removed? | The `windows_feature` resource to install or remove roles and sub-features |
| **Network and firewall** | Are Windows Firewall profiles enabled and only authorized ports accessible? | The `windows_firewall_profile` and `windows_firewall_rule` resources |
| **Services and remote access** | Are required services running, unnecessary daemons disabled, and WinRM active? | The `windows_service` resource for daemon state and startup modes |

### Published benchmarks for Windows Server

Chef publishes prebuilt audit and remediation content based on the CIS Benchmarks and DISA STIGs for Windows Server:

| Platform | Available benchmarks | Coverage type |
| --- | --- | --- |
| Windows Server 2025 | CIS Windows Server 2025 | Audit and remediation |
| Windows Server 2022 | CIS Windows Server 2022, STIG Windows Server 2022 | Audit and remediation |
| Windows Server 2019 | CIS Windows Server 2019, STIG Windows Server 2019 | Audit and remediation |

Consult the [Operating Systems](/premium_content/operating_systems/) coverage table for exact profile versions.
This walkthrough writes a custom baseline profile to teach the core mechanics of Windows configuration.

## What you'll build

You'll create a cookbook named `windows_baseline` and a companion InSpec profile that assert:

- The legacy SMBv1 protocol is disabled in the registry and the `SMB1Protocol` Windows feature is removed.
- Windows Firewall is enabled across Domain, Private, and Public profiles.
- Remote Desktop requires Network Level Authentication (NLA) before connection.
- Remote Desktop enforces TLS security rather than native RDP encryption.
- The `RemoteRegistry` service is disabled and stopped.

By the end, you'll converge a Windows Server instance, run the compliance checks over WinRM,
and schedule recurring converges across your fleet.

{{< note >}}

**Where the details come from.**
Chef Infra Client's [`registry_key`](/client/19/resources/bundled/registry_key/),
[`windows_feature`](/client/19/resources/bundled/windows_feature/),
[`windows_firewall_profile`](/client/19/resources/bundled/windows_firewall_profile/),
and [`windows_service`](/client/19/resources/bundled/windows_service/) resources define the desired state.
The [`registry_key`](/inspec/latest/resources/core/registry_key/),
[`windows_feature`](/inspec/latest/resources/core/windows_feature/),
[`windows_firewall`](/inspec/latest/resources/core/windows_firewall/),
and [`service`](/inspec/latest/resources/core/service/) InSpec resources validate the active configuration.

{{< /note >}}

## Before you begin

You need:

- **Chef Workstation** installed on your workstation. Run `chef --version` and `inspec version` to confirm.
- **A target running Windows Server** (2019, 2022, or 2025).
- **Administrative access** over WinRM (port 5985/5986) or OpenSSH.
- **WinRM configured for remote management**. On the Windows server, run:

  ```ps1
  winrm quickconfig -quiet
  ```

Test on a non-production server first before applying baseline policies across your organization.

## Step 1: Create a cookbook

Generate a cookbook for your Windows baseline:

```shell
chef generate cookbook windows_baseline
cd windows_baseline
```

Open `metadata.rb` and update `maintainer` and `maintainer_email` with your organization's details.

## Step 2: Write the recipe

Open `recipes/default.rb` and add the following resources:

```ruby
# Disable the insecure SMBv1 protocol in the LanmanServer registry
registry_key 'HKLM\SYSTEM\CurrentControlSet\Services\LanmanServer\Parameters' do
  values [
    { name: 'SMB1', type: :dword, data: 0 },
  ]
  action :create
end

# Remove the SMBv1 optional feature from the system
windows_feature 'SMB1Protocol' do
  action :remove
end

# Enable Windows Firewall across Domain, Private, and Public profiles
windows_firewall_profile %w(Domain Private Public) do
  action :enable
end

# Require Network Level Authentication (NLA) and TLS for Remote Desktop
registry_key 'HKLM\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' do
  values [
    { name: 'UserAuthentication', type: :dword, data: 1 },
    { name: 'SecurityLayer', type: :dword, data: 2 },
  ]
  action :create
end

# Disable and stop the Remote Registry service
windows_service 'RemoteRegistry' do
  action [:disable, :stop]
end
```

What each resource accomplishes:

`registry_key`
: Creates or updates values under the specified registry hive.
  Setting `SMB1` to `0` prevents the server from negotiating SMBv1 connections.
  Under `RDP-Tcp`, setting `UserAuthentication` to `1` enforces NLA, and setting `SecurityLayer` to `2` requires TLS encryption.

`windows_feature`
: Calls DISM or PowerShell to uninstall Windows optional features and roles.
  Removing `SMB1Protocol` uninstalls the binaries so the service can't be started.

`windows_firewall_profile`
: Enforces the enabled state for the specified firewall profiles (`Domain`, `Private`, and `Public`).

`windows_service`
: Manages Windows services using the Windows Service Control Manager.
  The `[:disable, :stop]` actions stop the service if running and set its startup type to `Disabled`.

Lint your cookbook with [Cookstyle](/workstation/cookstyle/):

```shell
cookstyle .
```

## Step 3: Write the compliance checks

Create an InSpec profile in your repository:

```shell
inspec init profile windows-server-baseline
```

In `windows-server-baseline/inspec.yml`, set the supported platform family:

```yaml
name: windows-server-baseline
title: Windows server baseline
maintainer: Example Organization, Inc.
copyright: Example Organization, Inc.
copyright_email: compliance@example.com
license: Apache-2.0
summary: Verify that Windows Server systems enforce baseline security controls
version: 0.1.0
supports:
  - platform-family: windows
```

Replace `windows-server-baseline/controls/example.rb` with `controls/windows.rb`:

```ruby
control 'win-01' do
  impact 'high'
  title 'Disable SMBv1 protocol'
  desc 'The insecure SMBv1 protocol must be disabled to prevent remote code execution and legacy protocol vulnerabilities.'
  desc 'rationale', 'SMBv1 lacks modern security features like pre-authentication integrity and encryption, making it vulnerable to exploit frameworks like EternalBlue.'
  tag 'windows', 'smb'

  describe registry_key('HKLM\SYSTEM\CurrentControlSet\Services\LanmanServer\Parameters') do
    its('SMB1') { should cmp 0 }
  end

  describe windows_feature('SMB1Protocol') do
    it { should_not be_installed }
  end
end

control 'win-02' do
  impact 'high'
  title 'Enable Windows Firewall on all network profiles'
  desc 'Windows Firewall must be active on Domain, Private, and Public profiles to control incoming and outgoing network traffic.'
  tag 'windows', 'firewall'

  %w(Domain Private Public).each do |profile|
    describe windows_firewall(profile) do
      it { should be_enabled }
    end
  end
end

control 'win-03' do
  impact 'high'
  title 'Require Network Level Authentication for Remote Desktop'
  desc 'Remote Desktop connections must require Network Level Authentication (NLA) before an RDP session is established.'
  desc 'rationale', 'NLA authenticates connecting users before the full RDP session and login screen load, mitigating credential exposure and denial of service attacks.'
  tag 'windows', 'rdp'

  describe registry_key('HKLM\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp') do
    its('UserAuthentication') { should cmp 1 }
  end
end

control 'win-04' do
  impact 'medium'
  title 'Enforce TLS encryption for Remote Desktop'
  desc 'Remote Desktop sessions must negotiate SSL/TLS encryption rather than native RDP security.'
  tag 'windows', 'rdp', 'encryption'

  describe registry_key('HKLM\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp') do
    its('SecurityLayer') { should cmp 2 }
  end
end

control 'win-05' do
  impact 'medium'
  title 'Disable the Remote Registry service'
  desc 'The Remote Registry service allows remote users to modify registry settings and must be disabled unless explicitly required.'
  tag 'windows', 'services'

  describe service('RemoteRegistry') do
    it { should_not be_enabled }
    it { should_not be_running }
  end
end
```

Validate the profile with `inspec check`:

```shell
inspec check windows-server-baseline
```

```text
Location :   windows-server-baseline
Profile :    windows-server-baseline
Controls :   5
Timestamp :  2026-11-11T14:41:22+05:30
Valid :      true

No errors, warnings, or offenses
```

## Step 4: Converge the server

On the target Windows server, run Chef Infra Client in local mode to apply the cookbook:

```shell
chef-client --local-mode --override-runlist windows_baseline
```

On the first run, Chef Infra Client updates the registry keys, removes the `SMB1Protocol` feature,
enables the firewall profiles, and disables the `RemoteRegistry` service.
Run the command a second time: Chef Infra Client reports `0` updated resources,
verifying that your configuration is idempotent.

## Step 5: Run the assessment

From your administrative workstation, execute the profile against the target server over WinRM:

```shell
inspec exec windows-server-baseline \
  --target winrm://<USER>@<SERVER_IP>:5985 \
  --password '<PASSWORD>'
```

If testing a node with unhardened defaults, InSpec displays actionable failures:

```text
Profile: Windows server baseline (windows-server-baseline)
Version: 0.1.0
Target:  winrm://Administrator@192.0.2.10:5985

  ×  win-01: Disable SMBv1 protocol (1 failed)
     ✔  Registry Key HKLM\SYSTEM\CurrentControlSet\Services\LanmanServer\Parameters SMB1 is expected to cmp == 0
     ×  Windows Feature SMB1Protocol is expected not to be installed
     expected that `Windows Feature SMB1Protocol` is not installed
  ✔  win-02: Enable Windows Firewall on all network profiles
     ✔  Windows Firewall Domain is expected to be enabled
     ✔  Windows Firewall Private is expected to be enabled
     ✔  Windows Firewall Public is expected to be enabled
  ×  win-03: Require Network Level Authentication for Remote Desktop (1 failed)
     ×  Registry Key HKLM\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp UserAuthentication is expected to cmp == 1
     expected: 1
          got: 0
  ✔  win-04: Enforce TLS encryption for Remote Desktop
     ✔  Registry Key HKLM\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp SecurityLayer is expected to cmp == 2
  ✔  win-05: Disable the Remote Registry service
     ✔  Service RemoteRegistry is expected not to be enabled
     ✔  Service RemoteRegistry is expected not to be running

Profile Summary: 3 successful controls, 2 control failures, 0 controls skipped
Test Summary: 6 successful, 2 failures, 0 skipped
```

After re-converging the server with `windows_baseline`, re-execute InSpec:
all 5 controls pass.

## Step 6: Test before you release

Use [Test Kitchen](/workstation/latest/tools/kitchen/) to test your Windows baseline on clean VMs
prior to production deployment.
Configure `kitchen.yml` to use Vagrant, Azure, or AWS drivers with the `winrm` transport:

```yaml
driver:
  name: vagrant

provisioner:
  name: chef_infra

verifier:
  name: inspec

transport:
  name: winrm

platforms:
  - name: windows-2022
    driver:
      box: mwrock/Windows2022

suites:
  - name: default
    verifier:
      inspec_tests:
        - path: ../windows-server-baseline
```

Execute the test lifecycle:

```shell
kitchen test
```

Test Kitchen provisions the VM, runs Chef Infra Client, runs the InSpec verifier, and destroys the instance.

## Scale configuration across your fleet

Individual manual runs prove the pattern.
To enforce your baseline across dozens or hundreds of Windows nodes, automate distribution:

- **Chef 360 Platform**:
  Enroll Windows nodes through [single-node or bulk enrollment](/360/latest/use/operators/node_enrollment/).
  Publish your Policyfile to Declarative State Management (DSM) using `chef push fleet`,
  then schedule recurring converges with [Chef Courier](/360/latest/use/operators/job_execution/).
- **Scheduled tasks**:
  For self-managed systems, use the [`chef_client_scheduled_task`](/client/19/resources/bundled/chef_client_scheduled_task/)
  resource to schedule `chef-client` execution through Windows Task Scheduler.
- **Compliance Phase**:
  Embed the InSpec profile in your cookbook or Policyfile with `default['audit']['compliance_phase'] = true`
  to send audit reports directly to [Chef Automate](/automate/reports/) on every run.

## Fix what fails

When an audit reports failures on Windows nodes:

- **Drift**: If an engineer manually alters a registry setting or stops the firewall,
  Chef Infra Client automatically reverts the setting to the baseline on the next run.
- **Feature installation reboots**: Some Windows roles require a reboot to finalize state.
  Use the [`reboot`](/client/19/resources/bundled/reboot/) resource with `action :request_reboot`
  to coordinate maintenance schedules cleanly.
- **Approved business exceptions**: If a specific server requires legacy settings,
  record a formal waiver in YAML instead of disabling the control.
  See [Waive an exception you've accepted](/solutions/compliance/ssh_baseline/#step-7-waive-an-exception-youve-accepted).

## Where to go next

**Add CIS benchmarks.**
Review the [Operating Systems](/premium_content/operating_systems/) coverage table
and add the published CIS benchmark for Windows to your Compliance Phase configuration.

**Configure web servers.**
If your Windows servers host web applications, follow
[The Chef way of configuring web servers and middleware](/solutions/compliance/web_servers_middleware/)
to audit and secure IIS with the `iis_site` resource.

**Manage fleet rollouts.**
See [The Chef way of managing a small fleet](/solutions/compliance/small_fleet/)
to learn how to use policy groups for phased canary releases.

## Related

- [Compliance solution overview](/solutions/compliance/)
- [The Chef way of SSH compliance](/solutions/compliance/ssh_baseline/)
- [The Chef way of Linux server compliance](/solutions/compliance/linux_server/)
- [The Chef way of managing a small fleet](/solutions/compliance/small_fleet/)
- [The Chef way of virtual machine compliance](/solutions/compliance/vm_compliance/)
- [Chef Infra Client `registry_key` resource](/client/19/resources/bundled/registry_key/)
- [Chef Infra Client `windows_feature` resource](/client/19/resources/bundled/windows_feature/)
- [Chef InSpec `registry_key` resource](/inspec/latest/resources/core/registry_key/)
- [Chef InSpec `windows_feature` resource](/inspec/latest/resources/core/windows_feature/)
- [Chef InSpec `windows_firewall` resource](/inspec/latest/resources/core/windows_firewall/)
