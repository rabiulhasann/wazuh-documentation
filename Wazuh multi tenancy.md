# Wazuh Multi-Tenancy Setup Guide

Step-by-step configuration of multi-tenancy in Wazuh using OpenSearch Dashboards, with checkpoints after every stage.

*Based on the guide by Syed Jawad Ali Shah.*

---

## Table of Contents

1. [Overview](#1-overview)
2. [Prerequisites](#2-prerequisites)
3. [Step 0: Enable Multi-Tenancy in the Dashboard](#3-step-0-enable-multi-tenancy-in-the-dashboard)
4. [Task 1: Create Agent Groups and Labels](#4-task-1-create-agent-groups-and-labels)
5. [Task 2: Create Internal User, Indexer Role, and Mapping (Data Isolation)](#5-task-2-create-internal-user-indexer-role-and-mapping-data-isolation)
6. [Task 3: Create Wazuh API Policy, Role, and Role Mapping](#6-task-3-create-wazuh-api-policy-role-and-role-mapping)
7. [Task 4: Enable `run_as` and Verify](#7-task-4-enable-run_as-and-verify)
8. [Repeat for the Second Group](#8-repeat-for-the-second-group-industry)
9. [Troubleshooting](#9-troubleshooting)
10. [Security Notes](#10-security-notes)
11. [Master Checklist](#11-master-checklist)

---

## 1. Overview

### What is multi-tenancy?

Multi-tenancy is an architecture where a single application instance serves multiple customers (tenants). Each tenant's data is isolated and invisible to others.

| Model | Description | Isolation |
|---|---|---|
| Single application, single database | Tenants share one database, separated by tenant IDs or schemas | Lower |
| Single application, multiple databases | One app instance, one database per tenant | Stronger |

**Advantages:** lower cost per user, faster onboarding, centralized maintenance, easy scaling, per-tenant customization, logical data isolation, single control panel.

**Limitations:** more access points (larger attack surface), added code/database complexity, harder backup and restore, fewer customizations, provider-side problems affect all tenants.

### Multi-tenancy in Wazuh

In the Wazuh dashboard, tenants are containers for index patterns, visualizations, dashboards, and other saved objects. Access to tenants is controlled by roles (read or write). By default, dashboard users have access to two independent tenants (Global and Private).

**Typical use case (MSSP):** a security provider monitors a retail company, a healthcare organization, and a logistics firm from one Wazuh server. Each client logs into the same dashboard but sees only its own agents, logs, alerts, and dashboards.

### How isolation works in this guide

Isolation is built from four layers:

| Layer | Mechanism | Where configured |
|---|---|---|
| 1. Identification | A `group` label in each agent group's `agent.conf` | Agents management > Groups |
| 2. Data isolation | Document-level security (DLS) filtering on `agent.labels.group` | Indexer management > Security > Roles |
| 3. API isolation | Wazuh RBAC policy scoped to `agent:group` | Server management > Security |
| 4. Identity | One internal user mapped to both roles | Indexer + Server security |

### Example used throughout

| Item | Value |
|---|---|
| Group 1 | `Rabiul` |
| Group 2 | `Industry` |
| Internal user | `rabiul` |
| Indexer role | `Read_Rabiul` |
| API policy | `Read_Rabiul` |
| API role | `Rabiul` |

---

## 2. Prerequisites

- Wazuh server, indexer, and dashboard installed (guide tested on Wazuh 4.11.2, dashboard on Ubuntu)
- Administrator access to the dashboard
- `sudo` access on the dashboard host
- At least two enrolled agents (example: `Agent1` and `Agent12`)

**Checkpoint 0**
- [ ] You can log into the dashboard as an administrator
- [ ] At least two agents appear under **Agents management > Summary**
- [ ] You have shell access with `sudo` on the dashboard host

---

## 3. Step 0: Enable Multi-Tenancy in the Dashboard

### 3.1 Edit the configuration file

```bash
sudo nano /etc/wazuh-dashboard/opensearch_dashboards.yml
```

### 3.2 Add or update these settings

```yaml
opensearch_security.multitenancy.enabled: true
opensearch_security.multitenancy.tenants.preferred: ["Global", "Private"]
uiSettings.overrides.defaultRoute: /app/wz-home?security_tenant=global
```

> **Note:** The original PDF text writes the route as `wz_home` (underscore), but its screenshot uses `wz-home` (hyphen). Use the hyphenated form `/app/wz-home?security_tenant=global`.

The `defaultRoute` line is optional. It sets which tenant is selected at login.

Related settings already present in the file (leave as they are):

```yaml
opensearch.requestHeadersAllowlist: ["securitytenant","Authorization"]
opensearch_security.readonly_mode.roles: ["kibana_read_only"]
opensearch_security.cookie.secure: true
```

### 3.3 Save and exit

`CTRL + X`, then `Y`, then `Enter`.

### 3.4 Restart the dashboard

```bash
sudo systemctl restart wazuh-dashboard
```

**Checkpoint 1**
- [ ] `sudo systemctl status wazuh-dashboard` shows `active (running)`
- [ ] The dashboard loads and you can log in again
- [ ] The file contains `opensearch_security.multitenancy.enabled: true` (check with `grep multitenancy /etc/wazuh-dashboard/opensearch_dashboards.yml`)

---

## 4. Task 1: Create Agent Groups and Labels

### 4.1 Open the Groups page

1. Log into the dashboard as administrator.
2. Open the menu (☰) and go to **Agents management > Groups**.

### 4.2 Create the two groups

1. Click **Add new group**.
2. Enter the name `Rabiul` and click **Save new group**.
3. Repeat with the name `Industry`.

**Checkpoint 2a**
- [ ] The Groups list shows `Rabiul`, `Industry`, and `default`

### 4.3 Add an identifying label to each group

1. Select the group and click **Edit group configuration**.
2. Replace the contents of `agent.conf` with the following, then click **Save**.

**Rabiul group:**

```xml
<agent_config>
  <labels>
    <label key="group">Rabiul</label>
  </labels>
</agent_config>
```

**Industry group:**

```xml
<agent_config>
  <labels>
    <label key="group">Industry</label>
  </labels>
</agent_config>
```

This label is what the document-level security filter matches on later (`agent.labels.group`).

**Checkpoint 2b**
- [ ] Each group's `agent.conf` shows its own label after saving
- [ ] The label value exactly matches the group name (case-sensitive)

### 4.4 Assign agents to groups

1. Go to **Agents management > Summary**.
2. Open the actions menu on an agent and choose **Edit groups**.
3. Select the target group (for example `Agent1` to `Rabiul`) and save.
4. Assign the second agent (for example `Agent12`) to `Industry`.

**Checkpoint 2c**
- [ ] The agent list shows `Agent1` in `Rabiul` and `Agent12` in `Industry`
- [ ] Agents have received the new configuration (allow a few minutes; agents must be connected to pull it)
- [ ] New alerts from each agent contain the `agent.labels.group` field (check in **Discover** with the `wazuh-alerts-*` index pattern)

---

## 5. Task 2: Create Internal User, Indexer Role, and Mapping (Data Isolation)

This layer restricts which alert and monitoring documents a user can read. The configuration below follows the official Wazuh procedure for creating and mapping an internal user.

### 5.1 Create the internal user

1. Click the upper-left menu icon (☰) and go to **Indexer management > Security > Internal users**.
2. Click **Create internal user**.
3. Enter a username (example: `rabiul`) and a strong password (at least 8 characters, with an uppercase letter, a lowercase letter, a digit, and a special character).
4. Click **Create**.

**Checkpoint 3a**
- [ ] The user appears in the **Internal users** list

### 5.2 Create the custom role

Go to **Indexer management > Security > Roles** and click **Create role**.

**Role name and cluster permissions**

| Field | Value |
|---|---|
| Name | `Read_Rabiul` |
| Cluster permissions | `cluster_composite_ops_ro` |

**Index permission 1: general read access (no DLS)**

| Field | Value |
|---|---|
| Index | `*` |
| Index permissions | `read` |
| Document-level security | leave empty |

> **Important:** Do **not** put a DLS filter on the `*` entry. Saved objects (index patterns, dashboards) live in the `.kibana*` indices and have no group field, so a filter there hides them and causes health-check errors such as `no permissions for [indices:data/write/index]`.

**Index permission 2: alerts (click "Add another index permission")**

| Field | Value |
|---|---|
| Index | `wazuh-alerts*` |
| Index permissions | `read` |

Document-level security (replace the group name accordingly):

```json
{
  "bool": {
    "must": {
      "match": {
        "agent.labels.group": "Rabiul"
      }
    }
  }
}
```

**Index permission 3: monitoring (click "Add another index permission")**

| Field | Value |
|---|---|
| Index | `wazuh-monitoring*` |
| Index permissions | `read` |

Document-level security (note the different field name, `group`):

```json
{
  "bool": {
    "must": {
      "match": {
        "group": "Rabiul"
      }
    }
  }
}
```

**Tenant permissions**

| Field | Value |
|---|---|
| Tenant | `global_tenant` |
| Access | **Read only** |

Click **Create**.

> **Tip:** The role name cannot be changed after creation. Valid characters are A-Z, a-z, 0-9, `_` and `-`.

**Checkpoint 3b**
- [ ] `Read_Rabiul` appears in the Roles list
- [ ] Its Permissions tab shows three index entries: `*` (no DLS), `wazuh-alerts*` (DLS on `agent.labels.group`), `wazuh-monitoring*` (DLS on `group`)
- [ ] Tenant `global_tenant` is set to read only
- [ ] The DLS values exactly match the group name (case-sensitive)

### 5.3 Map the user to the role

1. Open the role and select the **Mapped users** tab.
2. Click **Manage mapping**.
3. Add the user you created (`rabiul`) in the **Users** field.
4. Click **Map**.

The user now has read access to the Wazuh alerts and monitoring documents of the authorized agent group.

**Checkpoint 3c**
- [ ] The **Mapped users** tab of `Read_Rabiul` lists `rabiul`

---

## 6. Task 3: Create Wazuh API Policy, Role, and Role Mapping

This layer restricts which agents the user can see and manage through the Wazuh API.

### 6.1 Create the policy

1. Menu (☰) > **Server management > Security > Policies**.
2. Click **Create policy** and fill in:

| Field | Value |
|---|---|
| Policy name | `Read_Rabiul` |
| Action | `agent:read` (click **Add**). Add more if needed, for example `agent:restart`, `agent:upgrade` |
| Resource | `agent:group` |
| Resource identifier | `Rabiul` (click **Add**) |
| Effect | `Allow` |

3. Click **Create policy**.

> **Note:** The PDF's screenshot shows `agent:restart` selected in the dropdown but not yet added; only `agent:read` is in the Actions list. Click **Add** for each action you want the policy to include.

**Checkpoint 4a**
- [ ] `Read_Rabiul` appears in the Policies list with the expected actions and the resource `agent:group:Rabiul`

### 6.2 Create the role

1. Open the **Roles** tab and click **Create role**.
2. Fill in:

| Field | Value |
|---|---|
| Role name | `Rabiul` |
| Policies | `Read_Rabiul` |

3. Click **Create role**.

**Checkpoint 4b**
- [ ] Role `Rabiul` appears in the list with policy `Read_Rabiul` attached

### 6.3 Create the role mapping

1. Open the **Roles mapping** tab and click **Create Role mapping**.
2. Fill in:

| Field | Value |
|---|---|
| Roles | `Rabiul` and `cluster_readonly` (gives basic configuration read permissions) |
| Internal users | `rabiul` |

3. Click **Save role mapping**.

> **Note:** The PDF's screenshot shows only `Rabiul` selected, while the text also asks for `cluster_readonly`. Add both, as the text describes. Also, some Wazuh versions create a separate mapping rule per role, so you may need one mapping for each.

**Checkpoint 4c**
- [ ] Roles mapping lists `Rabiul` (and `cluster_readonly`) mapped to user `rabiul`

---

## 7. Task 4: Enable `run_as` and Verify

### 7.1 Enable `run_as` in `wazuh.yml`

```bash
sudo nano /usr/share/wazuh-dashboard/data/wazuh/config/wazuh.yml
```

Under the `hosts:` section, set `run_as: true`:

```yaml
hosts:
  - default:
      url: https://127.0.0.1
      port: 55000
      username: wazuh-wui
      password: <your-wazuh-wui-password>
      run_as: true
```

> The `wazuh-wui` username is required when `run_as` is enabled. Never publish the real password.

### 7.2 Restart the dashboard

```bash
sudo systemctl restart wazuh-dashboard
```

**Checkpoint 5a**
- [ ] `run_as: true` is set under `hosts > default`
- [ ] The dashboard restarted cleanly and the administrator can still log in

### 7.3 Verify tenant isolation

1. Log out of the administrator account.
2. Log in as `rabiul`.
3. Open **Endpoints** (Agents).

**Expected result:** only the `Rabiul` group's agents appear (for example `Agent1`, group `Rabiul`). Agents from `Industry` and `default` are not visible.

**Checkpoint 5b**
- [ ] `rabiul` sees only agents in the `Rabiul` group
- [ ] Dashboard charts (Top 5 groups, Top 5 OS, Agents by status) reflect only that group
- [ ] In **Discover**, alerts show only `agent.labels.group: Rabiul`

### 7.4 Verify the user cannot change security settings

1. While logged in as `rabiul`, open **Server management > Security**.

**Expected result:** a "You have no permissions" message (requires `security:read` on `user:id:*` and `role:id:*`).

**Checkpoint 5c**
- [ ] The Security section is blocked for `rabiul`
- [ ] `rabiul` cannot edit policies, roles, or role mappings

---

## 8. Repeat for the Second Group (Industry)

Repeat these items using `Industry` in place of `Rabiul`:

| Stage | Item to create |
|---|---|
| Task 2 | New internal user, and indexer role `Read_Industry` with the same three index entries; DLS `agent.labels.group: "Industry"` on `wazuh-alerts*` and `group: "Industry"` on `wazuh-monitoring*` |
| Task 2 | Map the new user to `Read_Industry` |
| Task 3 | Policy `Read_Industry` with resource `agent:group` = `Industry` |
| Task 3 | Role `Industry` with policy `Read_Industry` |
| Task 3 | Role mapping: `Industry` + `cluster_readonly` to the new user |
| Task 4 | Log in as the new user and verify |

**Checkpoint 6**
- [ ] The Industry user sees only `Industry` agents and alerts
- [ ] The `rabiul` user still sees only `Rabiul` data
- [ ] Neither user can open **Server management > Security**

---

## 9. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Tenant selector not shown after restart | Multi-tenancy setting missing or YAML indentation error | Re-check `opensearch_dashboards.yml`; run `sudo journalctl -u wazuh-dashboard -n 50` |
| Dashboard fails to start | YAML syntax error | Fix the file; each setting must be on one line, keys not indented |
| User sees no alerts | Label missing on agents, or label value differs from DLS query | Confirm `agent.labels.group` exists in Discover; match case exactly |
| User sees all alerts | DLS not applied on `wazuh-alerts*`, or the user is also mapped to another broad role | Review the role's index entries and the user's mappings; remove broad roles |
| Health check: `no permissions for [indices:data/write/index]` | DLS placed on the `*` entry hides `.kibana*` saved objects, so the app tries to recreate index patterns | Keep `*` with `read` and no DLS; put DLS only on `wazuh-alerts*` and `wazuh-monitoring*` |
| `no permissions for [indices:data/read/search]` | The role only covers some indices and a page queries another one | Keep the `*` read entry from Task 2; check the Network tab (F12) for the denied index |
| User sees no agents | API role not mapped, `run_as` not enabled, or policy resource wrong | Recheck Task 3 and Task 4 |
| "Wazuh not ready yet" or API errors after `run_as` | Wrong `wazuh-wui` credentials | Verify username and password in `wazuh.yml` |
| Label not appearing in alerts | Agent hasn't received updated group config | Wait for sync; check the agent is connected and belongs to the group |

---

## 10. Security Notes

- **Use strong passwords** for internal users and store them in a password manager.
- **Never share `wazuh.yml`** or screenshots showing the `wazuh-wui` password.
- **Least privilege:** grant only the actions each user needs (`agent:read` first; add `agent:restart` or `agent:upgrade` only when required).
- **Two layers must both be configured.** The indexer role (DLS) protects alert data; the Wazuh API role protects agent management. Missing either leaves a gap.
- **Multi-tenant environments have a larger attack surface.** Review roles and mappings regularly and keep Wazuh updated.
- **Backups:** restore procedures are more complex in multi-tenant setups; test them.

---

## 11. Master Checklist

- [ ] **0.** Prerequisites confirmed
- [ ] **1.** Multi-tenancy enabled in `opensearch_dashboards.yml`; dashboard restarted
- [ ] **2a.** Groups `Rabiul` and `Industry` created
- [ ] **2b.** Group labels added to each `agent.conf`
- [ ] **2c.** Agents assigned to groups; `agent.labels.group` visible in alerts
- [ ] **3a.** Internal user created
- [ ] **3b.** Indexer role `Read_Rabiul` created (`*` read, DLS on `wazuh-alerts*` and `wazuh-monitoring*`, `global_tenant` read only)
- [ ] **3c.** User mapped to the role
- [ ] **4a.** API policy `Read_Rabiul` created
- [ ] **4b.** API role `Rabiul` created
- [ ] **4c.** Role mapping created (`Rabiul` + `cluster_readonly`)
- [ ] **5a.** `run_as: true` enabled; dashboard restarted
- [ ] **5b.** Login as tenant user shows only own group's agents and alerts
- [ ] **5c.** Tenant user blocked from Security settings
- [ ] **6.** Steps repeated and verified for the `Industry` group

---

## Conclusion

Following this guide, one Wazuh instance can serve several clients or departments while keeping each one's agents, alerts, and dashboards isolated through labels, document-level security, and Wazuh RBAC. This is well suited to MSSPs, enterprises with multiple departments, and any setup that needs to reduce infrastructure cost while keeping tight data control.
