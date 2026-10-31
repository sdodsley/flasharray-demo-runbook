# FlashArray demo: from an empty array to protected, fleet-placed storage

A live-demo Ansible playbook for the [`everpure.flasharray`](https://galaxy.ansible.com/ui/repo/published/everpure/flasharray/) collection, and the runbook for presenting it.

In eight steps, one playbook takes a FlashArray from nothing to a tenant with protected storage and a Fusion-placed workload, pausing after every step so the presenter can talk:

| Step | Modules | Result |
|---|---|---|
| 1 Discover | `purefa_info` | inventory and a pass on the REST 2.40 floor |
| 2 Identity | `purefa_apiclient` | a key pair and an API client for the app team |
| 3 Tenant | `purefa_realm`, `purefa_pod`, `purefa_policy_management_access` | realm with quota and QoS, a pod in it, a realm-scoped access policy |
| 4 Hosts | `purefa_host`, `purefa_hg` | two iSCSI hosts and a host group in the realm |
| 5 Volume | `purefa_volume`, `purefa_hg` | two volumes in the pod, connected to the group |
| 6 Protect | `purefa_pg`, `purefa_pgsched`, `purefa_pgsnap` | protection group, schedule with retention, first snapshot |
| 7 Fleet | `purefa_fleet`, `purefa_preset`, `purefa_workload` | a workload preset and a Fusion workload placed from it |
| 8 Verify | `purefa_info`, `purefa_audits`, `purefa_metrics` | read-back assertions, the run's audit trail, live metrics |

## Contents

- [`empty_array_to_protected_storage.yml`](empty_array_to_protected_storage.yml) - the playbook. Every setting is a variable at the top; `--tags teardown` removes everything it created and is safe to run repeatedly.
- [`RUNBOOK.md`](RUNBOOK.md) - the presenter's runbook: what to have ready, a rehearsal checklist, what each step shows and what to say at each pause, reset, troubleshooting, variables, and the validation record.
- [`requirements.yml`](requirements.yml) / [`requirements.txt`](requirements.txt) - the collection and Python packages.

## Quick start

```sh
ansible-galaxy collection install -r requirements.yml   # everpure.flasharray >= 1.46.0
pip install -r requirements.txt                        # py-pure-client >= 1.88.0 and friends
export PUREFA_URL=<array management address>
export PUREFA_API=<API token>
ansible-playbook empty_array_to_protected_storage.yml                     # the demo, pausing after each step
ansible-playbook empty_array_to_protected_storage.yml -e demo_pause=false # straight through (rehearsal, CI)
ansible-playbook empty_array_to_protected_storage.yml --tags teardown     # reset
```

## What the target needs

- Purity//FA newer than 6.8.2 (REST API 2.40 or later); the metrics step needs Purity 6.7 or later.
- A Fusion fleet with the array as a member (the playbook can create a one-member fleet on a standalone array), and a Fusion storage class a member offers: set `demo_storage_class` (`flasharray-x` for a //X, `flasharray-c` for a //C).
- For step 7, `PUREFA_API` must be a token for an identity every fleet member knows (a directory-service user). A local array admin's token is enough for steps 1 to 6 and 8; Fusion refuses it (403) for anything that spans the fleet.
- `openssl` on the controller; the collection at 1.46.0 or later, which is the first release with `purefa_preset` and `purefa_metrics`.

The runbook has the full list, a rehearsal checklist, and a troubleshooting table keyed on the exact messages you might see.

## Status

Validated on 31 October 2026 against a two-array fleet on Purity//FA 6.10.6 (REST 2.54) with collection master for 1.46.0 plus the `purefa_preset` module: full run, idempotent re-run (0 changes), teardown twice (second one 0 changes, 0 failures), nothing left on either array. Details in the runbook's validation record.

## Notes

- Nothing in this repository holds credentials; the playbook reads `PUREFA_URL` and `PUREFA_API` from the environment and generates its throwaway key pair under `~/.everpure-demo`.
- `pause` needs a real terminal. Run the demo from a foreground shell, or pass `-e demo_pause=false`.
