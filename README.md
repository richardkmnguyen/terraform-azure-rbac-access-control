Watch me explain this lab: [LoomLink](PASTE-VIDEO-LINK-HERE)

# terraform-azure-rbac-access-control

Used Terraform and PowerShell to put Azure role-based access control on a single file server VM: an Owner for the senior sysadmin, Virtual Machine Contributor for the help desk, and Reader for the auditor, all scoped to that one VM. Then I signed in as each person and proved what they could and could not do. Getting there meant finding that the guide's assumptions about one built-in role were wrong, catching that my lab was in the wrong Entra ID tenant before creating any identities, moving the whole series to a dedicated lab tenant, and working through MFA, sign-in and execution-policy hurdles until every live test came out the way the role definitions say it should.

# Cloud Systems Administration, Lab 2: Azure RBAC

**Terraform · Azure RBAC · Microsoft Entra ID · Least Privilege · Azure CLI · PowerShell**

Lab 1 controlled who can open which files *inside* a server. This lab answers a different question: who can manage the server itself from Azure. Start it, restart it, resize it, delete it, or decide who else gets access. A file server with perfect NTFS permissions is still at risk if anyone with a portal login can delete the VM. Azure RBAC is how that gets locked down, and this lab builds the model end to end with code: three personas, three roles, one VM, every assignment tested live and validated with a script that exports a report.

This is the second lab of a three-lab Cloud Systems Administration series. It builds directly on the [Lab 1 NTFS File Server](https://github.com/richardkmnguyen/terraform-azure-ntfs-file-server/blob/main/README.md) environment and adds no new infrastructure: Terraform reads Lab 1's resources and only creates three role assignments. The third lab (Azure Update Manager) is standalone.

| | |
|---|---|
| **Platform** | Terraform, Azure CLI, and PowerShell on a local Windows PC, against Lab 1's VMs in Central US on my personal Azure subscription |
| **Time** | 4 hours |

---

## What this lab is for

The business problem: a help desk ticket says the Finance file server is unresponsive. The on-call technician needs to restart it, but should never be able to change who has access to it. A compliance auditor needs to see the server's configuration and who has access, but should not be able to touch anything. A senior sysadmin needs full control, including granting and removing access. Each person gets exactly what their job requires.

Three ideas make that work:

- **Roles, not individual permissions.** Azure groups permissions into roles. I used three built-in ones: **Owner** (everything, including managing access), **Virtual Machine Contributor** (manage VMs but not access), and **Reader** (view only).
- **Scope is the blast radius.** A role assignment applies at a scope: a subscription, a resource group, or one resource. I scoped all three to FS01's resource ID, so DC01 and CLIENT01 in the same resource group are untouched. Pointing the same code at the resource group ID would quietly grant access to every VM in it.
- **Azure RBAC and NTFS are separate layers.** RBAC works at the Azure Resource Manager control plane (who can manage the VM). NTFS works inside Windows (who can open files). Neither one controls the other, and a real environment needs both.

The three test personas are cloud identities in Microsoft Entra ID, not the `lab.local` users from Lab 1. Azure RBAC cannot see accounts that only exist inside a domain controller.

| Persona | Entra ID account | Role on FS01 | What they should be able to do |
|---|---|---|---|
| SysAdmin | `rbac-sysadmin` | Owner | Everything, including granting and removing access |
| SupportTech | `rbac-supporttech` | Virtual Machine Contributor | Manage the VM (start, restart, resize), but not manage access |
| Auditor | `rbac-auditor` | Reader | View the VM and who has access, take no actions |

---

## Architecture

```
My local Windows PC (PowerShell, Azure CLI, Terraform)
        │
        │  az login as my admin account; Terraform calls the Azure management API
        │  Terraform state stored remotely under its own key (rbac-lab.terraform.tfstate)
        ▼
Azure subscription (personal)
        │
        ├── RG-TerraformState: one storage account, two state files
        │       ntfs-lab.terraform.tfstate   (Lab 1)
        │       rbac-lab.terraform.tfstate   (Lab 2)
        │
        └── RG-FileServerLab (built by Lab 1, read-only to this lab via data sources)
                │
                ├── DC01       no role assignments, out of scope
                ├── CLIENT01   no role assignments, out of scope
                └── FS01       ← scope for all three role assignments
                        ├── Owner                        → rbac-sysadmin
                        ├── Virtual Machine Contributor  → rbac-supporttech
                        └── Reader                       → rbac-auditor

Microsoft Entra ID (my tenant)
        └── rbac-sysadmin, rbac-supporttech, rbac-auditor (cloud accounts, MFA enrolled)
```

<!-- SCREENSHOT: Lab 2 RBAC architecture diagram from the SOP -->

| Component | Role |
|---|---|
| **`data.tf`** | Looks up Lab 1's resource group and FS01 by name. Data sources read existing resources and never create, change, or delete them |
| **`rbac.tf`** | The three role assignments, each scoped to FS01's resource ID |
| **`variables.tf`** | Inputs for the resource group, VM name, and three Entra ID Object IDs, with no defaults and a GUID format check |
| **Remote state** | Same storage account as Lab 1, different key, so destroying Lab 2 can never touch Lab 1's resources |
| **`00-create-test-users.ps1`** | Creates the three Entra ID test users after confirming which tenant I'm signed in to |
| **`01-get-object-ids.ps1`** | Looks up each user's Object ID and writes `terraform.tfvars` |
| **`validate-lab.ps1`** | Checks the live assignments against the expected principals, checks blast radius, and exports a report |
| **`02-test-persona.ps1`** | Signed in as one persona, runs the allow and deny tests for that role and records PASS/FAIL |

Key concepts I needed:

- **Object ID, not email.** Role assignments point at an identity's Object ID (a GUID). A UPN can change, the Object ID never does.
- **Tenant vs. subscription.** A tenant is the identity directory (users, groups). A subscription is where resources and billing live, and it belongs to exactly one tenant. Role assignments can only grant access to identities from the subscription's own tenant.
- **Control plane vs. data plane.** Azure RBAC on a VM governs management actions through Azure. It does not decide who can RDP in: that is the NSG plus Windows/AD credentials.
- **RBAC propagation.** New assignments can take a few minutes to take effect, so validation runs after a short wait instead of right after `apply`.

---

## What I built

### 1. Created the test identities in Entra ID
The guide assumed the three test users already existed. I wrote a script that creates them in whatever tenant the CLI is signed into, skips any that already exist, and prints each user's UPN and Object ID. After what happened with my tenant (below), I added a safety stop: the script shows the tenant name and domains and will not create anything until I type the tenant's name back.

```powershell
$id = az ad user create `
    --display-name $u.Display `
    --user-principal-name $upn `
    --password $plain `
    --force-change-password-next-sign-in false `
    --query id -o tsv
```

<img width="578" height="241" alt="RBACCreateTestUsers" src="https://github.com/user-attachments/assets/d352b45e-2008-4e73-bfe1-486f54e0d858" />
<img width="946" height="343" alt="RBACEntraIDUsers" src="https://github.com/user-attachments/assets/e125e09e-2149-4286-9aa6-e1a2359b38f4" />

### 2. Looked up Object IDs and wrote the variables
A second script asks for each UPN, looks up the Object ID with `az ad user show`, refuses to continue if the same account is entered twice, and writes `terraform.tfvars` for me. The variables have no defaults and a GUID check, so a leftover placeholder fails at `terraform plan` instead of landing a role on nobody:

```hcl
variable "auditor_object_id" {
  description = "Entra ID Object ID for Auditor - receives Reader on FS01."
  type        = string

  validation {
    condition     = can(regex("^[0-9a-fA-F]{8}-([0-9a-fA-F]{4}-){3}[0-9a-fA-F]{12}$", var.auditor_object_id))
    error_message = "auditor_object_id must be an Entra ID Object ID (GUID). Run scripts/01-get-object-ids.ps1."
  }
}
```

### 3. Wrote the Terraform configuration
Lab 2 creates no infrastructure. `data.tf` reads FS01 from Lab 1, and `rbac.tf` uses its ID as the scope. The `scope` line is the most important line in each block:

```hcl
data "azurerm_virtual_machine" "fs01" {
  name                = var.vm_name
  resource_group_name = data.azurerm_resource_group.lab.name
}

resource "azurerm_role_assignment" "supporttech_vm_contributor" {
  scope                = data.azurerm_virtual_machine.fs01.id
  role_definition_name = "Virtual Machine Contributor"
  principal_id         = var.support_user_object_id
}
```

The state file uses its own key in the same storage account as Lab 1:

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "RG-TerraformState"
    storage_account_name = "<your-unique-storage-account-name>"
    container_name       = "tfstate"
    key                  = "rbac-lab.terraform.tfstate"
  }
}
```

<img width="443" height="188" alt="RBACRBACTF" src="https://github.com/user-attachments/assets/7010c924-a81b-4da3-bb7b-4e039b9bfbf4" />

### 4. Deployed and validated
`terraform plan` showed exactly three resources to add and nothing to change or destroy, and every scope pointed at FS01. `terraform apply` created all three in about 40 seconds. After a short wait for propagation, `validate-lab.ps1` checked the live state in Azure, not just Terraform's record of it:

1. **Each role is on FS01 and held by the right person**, compared against the Object IDs in `terraform.tfvars`.
2. **Blast radius:** none of the three test users holds any role anywhere else in the subscription.

Both sections passed, and the script exported `RBAC_Lab_Report.txt`.

<img width="979" height="571" alt="RBACTerraformPlan" src="https://github.com/user-attachments/assets/016802a7-ebd4-4965-9b8a-f7f00a2b9ab7" />
<img width="1526" height="923" alt="RBACTerraformApply" src="https://github.com/user-attachments/assets/80d27e9e-ba61-459d-8cce-e30c04be12ec" />
<img width="738" height="450" alt="RBACValidateScript" src="https://github.com/user-attachments/assets/1b3234e8-dc0e-44d5-a736-b65f89fc81a2" />
<img width="1312" height="543" alt="RBACRoleAssignmentFS01" src="https://github.com/user-attachments/assets/1871a2fd-5e30-4da5-be17-76884a0a95ce" />

### 5. Proved each role live, signed in as each persona
Validation proves the assignments exist. Only signing in as each user proves what the roles actually allow. I signed in as each persona in turn and ran `02-test-persona.ps1`, which runs the allow and deny checks for that role and counts an expected `AuthorizationFailed` as a pass:

| Signed in as | Test | Expected | Result |
|---|---|---|---|
| All three | View FS01 details | Allowed | As expected |
| All three | View role assignments on FS01 | Allowed | As expected |
| All three | View DC01 (outside the scope) | Denied | As expected |
| Auditor | Restart FS01 | Denied | As expected |
| SupportTech | Restart FS01 | Allowed | As expected |
| SupportTech | Create a role assignment on FS01 | Denied | As expected |
| SysAdmin | Create a temporary role assignment on FS01 | Allowed | As expected |
| SysAdmin | Remove that role assignment | Allowed | As expected |

<img width="426" height="436" alt="RBACAuditorChecks" src="https://github.com/user-attachments/assets/a5567271-6e12-425b-84ce-76821b498021" />
<img width="450" height="491" alt="RBACSupportTechChecks" src="https://github.com/user-attachments/assets/3bbf2c6d-a8ae-4640-8431-60872d7299c9" />
<img width="434" height="479" alt="RBACSysAdminChecks" src="https://github.com/user-attachments/assets/41a426ee-584f-4130-adff-7ab1236bb7a3" />

> The DC01 check is the one I like most: every persona can see FS01 and gets denied on DC01, sitting in the same resource group. That is the scope doing its job.

---

## The errors, one at a time

### The guide was wrong about Virtual Machine Contributor
The guide said the help desk role "cannot delete or resize the VM" and "cannot view RBAC assignments", and its Step 7 expected `az role assignment list` to fail for SupportTech. Before running anything, I checked the built-in role definition:

```powershell
az role definition list --name "Virtual Machine Contributor" --query "[].permissions[0].actions" -o json
# Microsoft.Authorization/*/read
# Microsoft.Compute/virtualMachines/*
# ...
```

`Microsoft.Compute/virtualMachines/*` covers delete and resize, and `Microsoft.Authorization/*/read` covers reading role assignments. So the guide's test would have "failed" by succeeding, and its documentation overstated the protection. What VM Contributor really cannot do is *write* role assignments. I rewrote the SupportTech test to try creating one (denied), corrected the comments in `rbac.tf` and the permission matrix in `validate-lab.ps1`, and recorded the real finding: for a help desk that only needs to restart servers, VM Contributor is broader than least privilege. A custom "VM Operator" role (read, start, restart, deallocate) would be the right fit.

### The guide's tests could have stopped the server or proved nothing
- The Auditor test used `az vm stop`. If propagation had not finished, it could actually shut FS01 down. I used `az vm restart --no-wait` instead, so the worst case is a quick reboot and the command returns as soon as Azure accepts or rejects it.
- `az vm start` on a VM that is already running "succeeds" as a no-op, so it proves nothing. Restart is a real action.
- The SysAdmin test only listed role assignments, which all three roles can do. Creating and removing a temporary assignment is the thing only Owner can do.
- The guide said to "open a new terminal for each persona", but the Azure CLI shares one token cache across every terminal. `az login` switches identity everywhere. The test script prints who is signed in and asks me to confirm before running.

### I was in the wrong tenant
When I went to create the test users, I checked which directory the CLI was pointing at:

```powershell
az account show --query "{Sub:name, Tenant:tenantId, User:user.name}" -o table
az rest --method get --url "https://graph.microsoft.com/v1.0/organization" --query "value[0].{Name:displayName, Domains:verifiedDomains[].name}"
```

The subscription I had built Lab 1 in belonged to an organizational Entra ID tenant, not the dedicated lab tenant I wanted this series in. My account could sign in to more than one directory, and the CLI had defaulted to the wrong one. Running the user script would have created three test accounts in a directory that was never meant to hold lab identities. I stopped before creating anything and moved the series to a subscription in a tenant set up only for these labs.

Nothing was harmed, but this is exactly how test accounts end up in a production directory. Anyone who works in IT and also runs labs is signed in to their employer's tenant all day, and `az login` picks a default without asking. A help desk technician practising RBAC on a Saturday could create users next to real staff, assign them roles, and never notice. That is why the user script now refuses to run until I type back the name of the tenant it is about to change.

The teardown had to remove Lab 1 and nothing else. I inventoried first (read-only), then let Terraform's own state decide what to delete:

1. Listed every resource group and confirmed only two belonged to this series.
2. Listed the resources in each, checked for locks, and confirmed the state container held only Lab 1's state file.
3. Read `terraform plan -destroy` line by line: 23 resources, all inside the lab's resource group.
4. Ran `terraform destroy`, confirmed the resource group was gone and the state was empty, and only then deleted the state storage.
5. Signed out of that tenant with `az logout` and `az account clear`, so later commands could not reach it by accident.

Then I redeployed Lab 1 on the personal subscription with the same Terraform and scripts, which went cleanly because every earlier fix was already in the code.

### `--auth-mode login` could not list the state blobs
```
You do not have the required permissions needed to perform this operation.
Depending on your operation, you may need to be assigned one of the following roles: "Storage Blob Data Owner" ...
```
During the teardown inventory, my account had management rights on the storage account (control plane) but no data-plane role to read blobs. Using the account key with `--auth-mode key` listed them. It was a small, real example of the same control plane vs. data plane split this lab is about.

### `AADSTS50076`: MFA required, so "No subscriptions found"
```
Authentication failed against tenant ... 'Default Directory': ... AADSTS50076: ... you must use multi-factor authentication
No subscriptions found for <my personal account>.
```
A plain `az login` completes MFA for the account's home sign-in, but not for each tenant it finds afterwards, so it skipped my tenant and reported no subscriptions. Naming the tenant made the CLI prompt for MFA:

```powershell
az login --tenant <tenant-id>
```

### Signing in as the test users
- The account picker offered "Microsoft account" or "Work or school account". Entra ID users are work or school accounts, even in a personal tenant.
- Each test user had to register MFA (Microsoft Authenticator) on first sign-in, because Security Defaults are on in the tenant.
- If the browser silently reuses my own session, `az login --use-device-code` in a private window avoids it.

### Scripts were disabled
```
configure-lab.ps1 cannot be loaded because running scripts is disabled on this system.
```
Windows PowerShell's default execution policy blocks scripts. `Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass` allows them only in the current window and resets when it closes, and `Unblock-File` clears the "downloaded from the internet" mark on the project files.

### HCL one-line blocks, again
The guide's `versions.tf` had `azurerm = { source = "hashicorp/azurerm" version = "~> 3.0" }`: two arguments in a one-line block with no separator, which Terraform rejects. Same fix as Lab 1, one argument per line.

---

## Problems and fixes

| What broke | Cause | Fix |
|---|---|---|
| SupportTech's "cannot view RBAC" test would pass the wrong way | VM Contributor includes `Microsoft.Authorization/*/read` (and VM delete) | Read the role definition; tested creating an assignment instead; corrected docs and matrix |
| Auditor test could stop FS01; start test proved nothing | `az vm stop` during propagation; `start` on a running VM is a no-op | `az vm restart --no-wait` |
| "New terminal per persona" gave no isolation | Azure CLI token cache is shared across terminals | Identity check and confirmation before each test |
| Test users would have landed in the wrong directory | CLI defaulted to an organizational tenant, not the lab tenant | Inventoried, `terraform destroy` of Lab 1 only, redeployed in a dedicated lab tenant, added a tenant confirmation to the user script |
| Blob list denied with `--auth-mode login` | Control-plane rights only, no Storage Blob Data role | `--auth-mode key` |
| `AADSTS50076`, "No subscriptions found" | Tenant requires MFA, plain `az login` skipped it | `az login --tenant <tenant-id>` |
| Test user sign-in | Picker defaults to my personal account | "Work or school account", MFA registration, device code as fallback |
| `running scripts is disabled` | Default PowerShell execution policy | `Set-ExecutionPolicy -Scope Process Bypass`, `Unblock-File` |
| One-line provider block rejected | Two arguments on one line in the guide's HCL | One argument per line |

---

## Key takeaways

1. **Read the role definition, not the role name.** "Contributor" sounds limited and "Reader" sounds harmless. The actions list is the truth: VM Contributor can delete a VM, and all three roles can read who has access.
2. **Least privilege often means a custom role.** Built-in roles are a starting point. When the job is "restart the server", a role that can also delete it is too much.
3. **Scope is the most important line.** The same assignment at the resource group would have quietly covered DC01 and CLIENT01. The blast-radius check makes that mistake visible.
4. **Prove deny, not just allow.** Every test that expects `AuthorizationFailed` is as important as the ones that succeed, and a test only counts if it could actually fail.
5. **Know which tenant you are in before you create anything.** `az account show` and the Graph `/organization` call take five seconds. In a lab the wrong tenant is an inconvenience; in a directory with real users, a stray account with a role assigned is a security incident.
6. **Tear down with the tool that built it.** `terraform destroy` removed exactly what Terraform created and nothing else, which is what made a careful cleanup inside a tenant with other resources in it possible.
7. **Control plane and data plane are separate everywhere.** Azure RBAC vs. NTFS on the VM, and storage account management vs. blob data access. Having rights to manage a thing does not mean having rights to its contents.

If I did this again, I would check the tenant and subscription before Lab 1 instead of before Lab 2, read each built-in role's actions before writing the test plan, and build the custom VM Operator role from the start.

---

## Security notes

This is a lab configuration. Some of it is solid and some of it would not be acceptable in production:

**What the lab does well**
- **Every assignment is scoped to one VM**, and a script proves the test users have no access anywhere else.
- **Object IDs are validated and kept out of source control.** `terraform.tfvars` is in `.gitignore`, and a placeholder can never reach Azure.
- **Separate state files.** Destroying Lab 2 cannot touch Lab 1.
- **MFA on every test identity.**
- **The lab runs in a dedicated tenant**, separate from any directory with real users.

**What I would change for production**
- **Assign roles to Entra ID groups, not individual users**, the same lesson as NTFS groups in Lab 1.
- **Use a custom VM Operator role for the help desk** instead of VM Contributor.
- **Split Owner** into Contributor plus Role Based Access Control Administrator (or User Access Administrator), and use Privileged Identity Management for just-in-time elevation instead of standing Owner rights.
- **Use Conditional Access** instead of Security Defaults to require MFA and limit where admins can sign in from (needs Entra ID P1).
- **Test users share one password.** Fine for a lab, wrong for real accounts.

---

## Verification

| Check | How I verified it |
|---|---|
| Right tenant | `az account show` and the Graph `/organization` call showed my personal tenant |
| Test identities exist | `00-create-test-users.ps1` created three users; `01-get-object-ids.ps1` returned three distinct Object IDs |
| Configuration valid | `terraform validate` succeeded; GUID validation on every Object ID |
| Plan scoped correctly | `terraform plan` showed 3 to add, 0 to change, 0 to destroy, every scope FS01 |
| Assignments live in Azure | `validate-lab.ps1` section 1: all three roles on FS01 held by the expected principals |
| No access outside FS01 | `validate-lab.ps1` section 2: each test user holds roles only at FS01's scope |
| Roles enforced as designed | `02-test-persona.ps1` passed every check for Auditor, SupportTech, and SysAdmin |
| Report exported | `RBAC_Lab_Report.txt` and `Persona_Test_<persona>.txt` for each persona |

<img width="738" height="450" alt="RBACValidateScript" src="https://github.com/user-attachments/assets/c7ad564e-1177-4029-ad2b-84f6b4c56951" />
<img width="426" height="436" alt="RBACAuditorChecks" src="https://github.com/user-attachments/assets/6b763836-c3c0-428d-bd2f-407e831148ed" />
<img width="450" height="491" alt="RBACSupportTechChecks" src="https://github.com/user-attachments/assets/7d06aa93-d349-4727-9452-41355eab6f29" />
<img width="434" height="479" alt="RBACSysAdminChecks" src="https://github.com/user-attachments/assets/03838b1f-0ab5-4e19-8c41-42771ba94984" />

---

## Teardown

Role assignments and Entra ID users cost nothing. The cost is Lab 1's VMs, so between sessions I deallocate them:

```powershell
# Pause: stops compute charges (deallocate, not just stop)
az vm deallocate --ids $(az vm list -g RG-FileServerLab --query "[].id" -o tsv)

# Remove only Lab 2's role assignments (Lab 1 untouched)
terraform destroy

# Remove the test users when the series is done
az ad user delete --id rbac-sysadmin@<tenant-domain>
az ad user delete --id rbac-supporttech@<tenant-domain>
az ad user delete --id rbac-auditor@<tenant-domain>
```

Run `terraform destroy` from the Lab 2 folder to remove only the three role assignments. Deleting the resource group instead would wipe Lab 1 too.

<img width="626" height="164" alt="RBACDeallocate" src="https://github.com/user-attachments/assets/f58f5ebe-980a-4e67-95c2-8cc2ff444362" />

---

## Next steps

- **Lab 3:** Azure Update Manager, standalone
- Build and test a custom **VM Operator** role and swap it in for SupportTech
- Move the assignments to **Entra ID groups** (`GRP_Azure_HelpDesk`, `GRP_Azure_Auditors`)
- Use **Privileged Identity Management** for time-limited Owner access
- Add an **Azure Policy** that denies role assignments at resource group scope or above for this lab
- Send Azure Activity Log role-assignment events to **Splunk** from my SIEM lab

---

**Status: complete.** Found and corrected the guide's role assumptions → moved the series to a dedicated lab tenant and rebuilt Lab 1 → three Entra ID personas created and enrolled in MFA → three role assignments deployed by Terraform and scoped to FS01 → validation and blast-radius checks ALL PASS → every allow and deny verified live as each persona.
