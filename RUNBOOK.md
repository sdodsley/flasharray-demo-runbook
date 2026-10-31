# Runbook: from an empty array to protected, fleet-placed storage

Presenter's runbook for the eight-step live demo of the `everpure.flasharray` collection (1.46.0 or later). What to have ready, what each step does and shows, what to say at each pause, how to reset, and what to do when something goes wrong.

- Playbook: `empty_array_to_protected_storage.yml`
- Run time: about 6 minutes without pauses
- Reset: `--tags teardown`, safe to repeat
- Validated: 31 October 2026, Purity//FA 6.10.6

## 1. The story in one screen

One playbook takes a FlashArray from nothing to a tenant with protected storage and a Fusion-placed workload, in eight steps that each map to one idea: discover, give the app team an identity, carve out their tenant, define hosts, create and connect volumes, protect them, let Fusion place a workload from a preset, then prove it all from the array's own records. After every step the playbook pauses so you can talk.

| Step | Modules | What exists afterwards |
|---|---|---|
| 1 Discover | `purefa_info` | An inventory and a pass on the REST 2.40 floor |
| 2 Identity | `purefa_apiclient` | A local key pair and an API client on the array for the app team |
| 3 Tenant | `purefa_realm`, `purefa_pod`, `purefa_policy_management_access` | Realm with quota and QoS, a pod inside it, a storage-role policy scoped to the realm |
| 4 Hosts | `purefa_host`, `purefa_hg` | Two iSCSI hosts and a host group in the realm |
| 5 Volume | `purefa_volume`, `purefa_hg` | 100G and 50G volumes in the pod, connected to the group |
| 6 Protect | `purefa_pg`, `purefa_pgsched`, `purefa_pgsnap` | Protection group, hourly schedule with 14-day retention, first snapshot |
| 7 Fleet | `purefa_fleet`, `purefa_preset`, `purefa_workload` | A workload preset and a Fusion workload placed from it |
| 8 Verify | `purefa_info`, `purefa_audits`, `purefa_metrics` | Read-back assertions, the audit trail of this run, live metrics |

Names follow the array's own rules: hosts and host groups live directly in the realm as `realm::name`; volumes and protection groups cannot sit in a realm, so they live in the pod as `realm::pod::name`.

## 2. Before the day

### What the target must offer

| Requirement | Why |
|---|---|
| Purity//FA newer than 6.8.2, REST API 2.40 or later | `purefa_workload` needs 2.40; realms and access policies 2.36; fleets 2.38. The metrics exporter needs Purity 6.7. |
| A Fusion fleet with the array as a member | Presets and workloads are fleet objects. The playbook can create a one-member fleet on a standalone array (`demo_create_fleet`). |
| A Fusion storage class a member offers | A preset must name one. `flasharray-x` for a //X, `flasharray-c` for a //C. Set `demo_storage_class`. |
| A token for a fleet-wide identity | Preset writes, workload lookup and placement are refused (403) for a local array user. Use a directory-service user known to every member. A local array_admin token is enough for steps 1 to 6 and 8. |
| Clocks within a few minutes | Step 8 lists audit events since five minutes before the run started. |

### Controller setup

```sh
ansible-galaxy collection install -r requirements.yml      # everpure.flasharray >= 1.46.0
pip install -r requirements.txt                           # py-pure-client >= 1.88.0, pytz, ...
export PUREFA_URL=<array management address>
export PUREFA_API=<API token of the fleet-wide user>
```

`purefa_preset` and `purefa_metrics` arrived in 1.46.0. Ansible resolves every module when it loads the play, so an older collection refuses to parse the file at all, whatever tags you pass. `openssl` must be on the controller: step 2 generates a throwaway key pair under `~/.everpure-demo`.

### Variables to look at

Everything is in the `vars:` block at the top of the playbook and can be overridden with `-e`. The ones that change per target: `demo_storage_class`, the realm and pod names if the array already has them, and `demo_preset` if you want to use an existing preset instead of creating one.

## 3. Rehearsal, the day before

Run the whole thing without pauses, run it again, tear it down twice, and look at the array. The second run must report zero changes and the second teardown must report zero changes and zero failures; that is what makes it safe to re-run on stage.

```sh
ansible-playbook empty_array_to_protected_storage.yml -e demo_pause=false
ansible-playbook empty_array_to_protected_storage.yml -e demo_pause=false   # expect changed=0
ansible-playbook empty_array_to_protected_storage.yml --tags teardown
ansible-playbook empty_array_to_protected_storage.yml --tags teardown       # expect changed=0, failed=0
```

- [ ] Run 1 ends with about 16 changed tasks and no failures; step 8 prints the audit list and `none` under "Hardware components not reporting ok".
- [ ] Step 7 prints where Fusion placed the workload and status `ready`.
- [ ] After the teardowns, no `demo-app-*` objects remain on any fleet member; the fleet itself is deliberately left in place.
- [ ] Run the real thing once more *with* pauses in the terminal you will present from. `pause` needs a real terminal; under a job runner it hangs or aborts.

## 4. On the day

```sh
ansible-playbook empty_array_to_protected_storage.yml
```

Each step ends with a pause that recaps what happened and names the next step. Press Enter when you are done talking. If you need to restart from a step, use its tag: `--tags step4`. Steps are self-contained once the earlier ones have run.

### Step 1 Discover (`purefa_info`)

- Shows: negotiated REST version and counts of hosts, groups, volumes, protection groups, pods, realms and API clients.
- Say: one module reads the whole array; the same call gates the demo on the version floor, so nothing later fails for a reason we could have known now.
- Pause: *Step 1 · Discover done: the array is inventoried and meets the version floor. Press Enter for step 2 · Identity*

### Step 2 Identity (`openssl`, `purefa_apiclient`)

- Shows: key pair generated locally; API client `demo-app-team-client` registered with the public key, role storage_admin, 1-hour tokens.
- Say: the app team never gets a password. They sign JWTs with their private key and the array exchanges them for short-lived tokens. Step 8 prints the client_id and key_id they need.
- Pause: *Step 2 · Identity done: a key pair was generated locally and the API client registered on the array with its public key. Press Enter for step 3 · Tenant*

### Step 3 Tenant (`purefa_realm`, `purefa_pod`, `purefa_policy_management_access`)

- Shows: realm `demo-app-team` with a 5T quota and 1 GB/s, 100K IOPS limits; pod `app-data` inside it; policy `demo-app-team-storage` granting the storage role on that realm only.
- Say: a realm is a hard boundary with its own quota and QoS. The pod is where volumes and protection groups live. The policy is what you bind to an AD group or a local user to hand the team their slice and nothing else.
- Pause: *Step 3 · Tenant done: the realm, a pod inside it and the realm-scoped access policy exist. Press Enter for step 4 · Hosts*

### Step 4 Hosts (`purefa_host`, `purefa_hg`)

- Shows: two iSCSI hosts `demo-app-team::app-host-01/02` and host group `demo-app-team::app-cluster`.
- Say: hosts belong to the realm directly. Connecting through a group means every volume reaches every node of the cluster with the same LUN.
- Pause: *Step 4 · Hosts done: two hosts and their host group live in the realm. Press Enter for step 5 · Volume*

### Step 5 Volume (`purefa_volume`, `purefa_hg`)

- Shows: `demo-app-team::app-data::db-data-01` (100G) and `db-log-01` (50G) created and connected to the group.
- Say: thin, instant, and already counted against the realm's quota. Three-part names read as realm, pod, volume.
- Pause: *Step 5 · Volume done: the volumes exist in the realm's pod and are connected to the host group. Press Enter for step 6 · Protect*

### Step 6 Protect (`purefa_pg`, `purefa_pgsched`, `purefa_pgsnap`)

- Shows: protection group `db-protect` with both volumes; schedule every hour, keep all for 2 days then 6 a day for 14 days; snapshot `db-protect.demo-baseline`.
- Say: both volumes are snapped together, so data and log stay consistent. The schedule values are deliberately not the factory defaults, so this step visibly changes something.
- Pause: *Step 6 · Protect done: protection group with both volumes, a snapshot schedule with retention, and a first snapshot. Press Enter for step 7 · Fleet*

### Step 7 Fleet (`purefa_fleet`, `purefa_preset`, `purefa_workload`)

- Shows: preset `demo-app-preset` created (two 10 GiB data volumes on the storage class); Fusion's placement recommendation; workload `demo-app-workload` with its member, status `ready` and volume names.
- Say: a preset is the template; the workload is Fusion deciding where it fits across the fleet. Nobody picked an array. This step takes about a minute while Fusion provisions.
- Pause: *Step 7 · Fleet done: a workload preset was created and Fusion placed a workload from it. Press Enter for step 8 · Verify*

### Step 8 Verify (`purefa_info`, `purefa_audits`, `purefa_metrics`)

- Shows: twelve assertions over a fresh read-back; the app team's client_id, key_id, issuer and TTL; the audit trail of this run in UTC; volume performance for the two volumes; hardware components not reporting ok (expect `none`).
- Say: nothing here is taken from what we sent. It is what the array says exists, who did it, and how the hardware feels. The same modules are what you would wire into monitoring.
- Pause: *Step 8 · Verify done: read-back, audit trail and metrics shown. That is the whole demo; press Enter to finish*

## 5. Reset for the next run

```sh
ansible-playbook empty_array_to_protected_storage.yml --tags teardown
```

Teardown works in reverse order: workload and demo preset, every snapshot of the protection group, the group, the host group, the volumes, the hosts, the pod, the access policy, the realm, the API client, the local key pair. It reads what is left first, so running it twice is a no-op, and a failure in the fleet part is reported and skipped rather than stopping the rest. Two things are left on purpose: a fleet created by step 7, and a preset you named yourself with `-e demo_preset`.

## 6. When it goes wrong

| You see | Cause | Do |
|---|---|---|
| `couldn't resolve module/action everpure.flasharray.purefa_preset` before anything runs | Collection older than 1.46.0 | Reinstall from `requirements.yml`; `ansible-galaxy collection list everpure.flasharray` must show 1.46.0 or later. |
| Step 1 assertion: "negotiated REST x.y ... need 2.40" | Array too old, or py-pure-client older than the array | Upgrade Purity past 6.8.2, or upgrade py-pure-client. |
| Step 7: `Failed to read workload ... status_code 403`, or the preset create is refused with a directory-identity hint | `PUREFA_API` is a local array user's token | Use a token for a directory-service user known to every fleet member. |
| Step 7: `Recommendation calculation failure. Error: No viable targets` | No fleet member offers `demo_storage_class`, or the preset needs parameters | Set `-e demo_storage_class=flasharray-c` (or the right class); with `-e demo_preset` pick a simpler preset. |
| Step 7: "No workload preset named ..." | Typo, or a fleet-qualified name | Pass the bare name; the playbook strips a `fleet:` prefix but check the spelling against the list it prints. |
| Step 8: "Skipped metrics: ..." | Purity older than 6.7, an ID token instead of an API token, or a proxy in the shell | The message carries the real error. The task already bypasses `https_proxy`; check the token type and Purity version. |
| Step 8 audit list is empty or short | Controller and array clocks differ by more than five minutes | Fix the clock, or pass `-e demo_audit_since="YYYY-MM-DD HH:MM:SS"` in UTC. |
| Step 2 warns that public_key, issuer, role or token_ttl differ | An API client of that name already exists with other values; the array cannot change them in place | The playbook drops the client when it regenerates the key pair; otherwise run teardown or delete the client and re-run step 2. |
| Teardown: pod or realm cannot be eradicated, a protection group is retention-locked | The pod was created outside the playbook on an array with a container default protection | The playbook creates the pod with `with_default_protection: false` for this reason. A locked group can only be destroyed and left to its eradication timer. |
| The play stops at a pause and never returns, or aborts with an EOF | No terminal attached (job runner, redirected input) | Run from a foreground shell, or pass `-e demo_pause=false`. |
| Teardown prints "Fleet objects were not all removed" and continues | The fleet part failed (usually the token) | Delete `demo-app-workload` and `demo-app-preset` in Fusion, or re-run teardown with the fleet-wide token. |

## 7. Variables and tags

| Variable | Default | Meaning |
|---|---|---|
| `demo_pause` | `true` | Wait for Enter after every step. `false` runs straight through. |
| `demo_realm` / `demo_pod` | `demo-app-team` / `app-data` | Tenant names. Volumes and protection groups are `realm::pod::name`. |
| `demo_realm_quota`, `demo_realm_bw_qos`, `demo_realm_iops_qos` | `5T`, `1G`, `100K` | Realm limits. Quota needs a unit suffix. |
| `demo_api_client`, `demo_issuer`, `demo_api_client_role`, `demo_api_client_ttl` | `demo-app-team-client`, `demo-app-team-issuer`, `storage_admin`, `3600` | Step 2. TTL is seconds, 1 to 86400. |
| `demo_hosts`, `demo_hostgroup`, `demo_volumes` | two hosts, `app-cluster`, 100G + 50G | Steps 4 and 5. |
| `demo_snap_frequency`, `demo_snap_keep_all_for`, `demo_snap_per_day`, `demo_snap_days` | `3600`, `172800`, `6`, `14` | Step 6 schedule, in seconds and counts. Not the factory defaults on purpose. |
| `demo_min_rest_version` | `"2.40"` | Step 1 gate. |
| `demo_create_fleet`, `demo_fleet_name` | `true`, `demo-fleet` | Create a one-member fleet when the array is standalone. |
| `demo_preset` | `""` | Bare name of an existing preset to use instead of creating one. Teardown never deletes a preset named here. |
| `demo_preset_name`, `demo_storage_class`, `demo_preset_volume_count`, `demo_preset_volume_size` | `demo-app-preset`, `flasharray-x`, `"2"`, `"10737418240"` | The preset step 7 creates. Count and size are strings because the preset API stores numbers as text. |
| `demo_workload`, `demo_workload_host`, `demo_workload_parameters` | `demo-app-workload`, `""`, `[]` | Step 7 workload; host and parameters are optional. |
| `demo_audit_since` | `""` | UTC start of the step 8 audit window; empty means five minutes before the run began. |

Tags: `step1` to `step8` (also `discover`, `identity`, `tenant`, `hosts`, `volume`, `protect`, `fleet`, `verify`), and `teardown`, which never runs unless asked for.

## 8. Validation record

- Date: 31 October 2026
- Collection: everpure.flasharray 1.46.0
- Target: fleet of two FlashArrays, Purity//FA 6.10.6, REST 2.54 negotiated, py-pure-client 1.94.1, ansible-core 2.17.9
- Identity: directory-service user known to both members
- Full run: run 1 37 tasks ok, 16 changed, 0 failed; run 2 0 changed; teardown 1 12 changed, 0 failed; teardown 2 0 changed, 0 failed; no objects left on either member
- Static: `ansible-playbook --syntax-check` clean; ansible-lint production profile clean; every module option checked against the collection's argument specs
