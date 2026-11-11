# The Armada Framework

Armada is an opinionated set of conventions for organizing server fleets and infrastructure stacks in a single Ansible repository. Start with the inventory to find a target, the layer playbook to see what runs and in what order, and the layer variables to see how each role is configured.

The [Kerbmark Guidelines](https://github.com/d1ys3nk0/kerbmark) defer to Armada for organizing and operating Ansible repositories.

## Repository layout

A **layer** is a project-defined operational grouping of roles. It selects and orders capabilities for a target and owns their execution context. An inventory identifies the hosts and describes their machine-specific inputs. A stack can span multiple hosts, and a host can participate in multiple layers.

| Path | Ownership |
| --- | --- |
| `inventories/<inventory>/hosts.yml` | Inventory hosts, groups, and machine-specific inputs. Projects can document a multi-file inventory where needed. |
| `playbooks/<layer>/` | Permanent operational entrypoints for one layer. |
| `variables/` | Global declarations, composition settings, and a directory for each layer that declares role inputs or scoped values. |
| `roles/<role>/` | One reusable capability and its implementation, defaults, assets, and tests. |
| `assets/` | Controller-side payloads genuinely shared across roles. |
| `lib/` | Shared controller-side helpers grouped by purpose. |
| `tests/` | Repository-wide behavior and test infrastructure shared across roles. |
| `migrations/<layer>/YYMMDDHHMMSS-slug/` | One historical transition and its operator notes, playbooks, and dedicated assets. |

Keep each path only where it has content. Use populated YAML files with names that describe their purpose. Keep shared controller helpers in `lib/` and installed programs in their owning roles.

## Layer entrypoints and target selection

| Entrypoint under `playbooks/<layer>/` | Purpose |
| --- | --- |
| `setup.yml` | Converge the target to its desired configuration. |
| `update.yml` | Explicitly refresh managed dependencies. |
| `assess.yml` | Add when operators need a standalone read-only assessment. |
| `reconcile.yml` | Add only where the project manages master/replica reconciliation for infrastructure Cells. |

Use ordered `roles:` lists with literal role names, tags, and selection conditions where needed. Use separate plays for distinct host, account, privilege, or readiness boundaries. Put task logic and guards in the owning capability and declare configuration in variable files. Playbooks can bind a declared value or a runtime output to a receiving role's namespaced parameter. Keep those bindings explicit.

Use scoped tags for focused operations. A focused run must retain its prerequisites. Keep opt-in capabilities out of untagged setup. Validate the target, identity, and incompatible selections before migration preflight, credential loading, or other side effects.

For Task-based repositories, expose `play:setup`, `play:update`, `play:setup:check`, and `play:update:check`. Pass the target explicitly as positional arguments and select Ansible tags with `T`. Use the same target validation and dispatch for live runs and previews, and accept the same public inputs for syntax checks and tag listings. Document the target grammar, supported layer and inventory combinations, tags, and prerequisite behavior in the project. Keep the root `Taskfile.yml` focused on wiring and command discovery, with executable procedures grouped by responsibility under `.taskfiles/`.

### Mapping Meridian identities to Ansible targets

Projects using [Meridian](https://github.com/d1ys3nk0/meridian#qualified-identities) document how each operation’s arguments resolve to a qualified identity, inventory, layer, and host selection. Resolve and validate the effective selection before accessing credentials or performing side effects. A short selector is sufficient when the project fixes its enclosing scope and validates that it resolves unambiguously.

A Cell operation selects a deployment target. A Space operation selects an application environment and may involve multiple Cells. An operation limited to one placement of a Space selects a logical deployment. Document the scope of the resources each operation can affect, including Realm-shared resources. Define lock and migration completion keys for the affected scope and document how operations with overlapping scopes coordinate.

## Variable ownership and precedence

| Path | Declares |
| --- | --- |
| `inventories/<inventory>/hosts.yml` | Machine inputs named `iv_*` and Ansible-native inventory keys where needed. |
| `variables/_<group>.yml` | Global constants named `gc_*`. |
| `variables/_<group>.vault.yml` | Global secrets named `gs_*`. |
| `variables/<group>.yml` | Recognized Ansible-native settings and file-local composition. |
| `variables/<layer>/_<group>.yml` | Layer constants named `lc_*`. |
| `variables/<layer>/_<group>.vault.yml` | Layer secrets named `ls_*`. |
| `variables/<layer>/<group>.yml` | Namespaced role inputs. |

Use lowercase, hyphen-separated group names, with a leading underscore only where shown. The immediate directory names under `playbooks/` determine the allowed directories under `variables/`. A layer may omit its variable directory. Reject nested variable directories and symlinks. Declare role inputs separately in each consuming layer.

Start each YAML document with `---` and a blank line. In a role input file, give each declared role one nonempty `# Role: <role_name>` section with a blank line before and after the heading. Place phase inputs under their parent role. When role prefixes overlap, the most specific role namespace owns the declaration.

In other variable files, including decrypted Vault YAML, group declarations into nonempty purpose sections headed by a descriptive column-zero comment such as `# Target identity`. Leave a blank line before and after each heading and keep explanatory comments next to their declarations.

Public variable files may declare file-local temporaries named `_[a-z][a-z0-9_]*` in an unheaded preamble before the first section. Reference a temporary at least twice in its defining file. Inline single-use values and use a scoped declaration for configuration shared across files. Root composition files may declare only these temporaries and recognized Ansible-native settings.

Each setting has one authoritative declaration. Bind inventory inputs directly to role inputs where possible. Put a default at the boundary that handles omission and use a shared declaration for intentional sharing. Detect duplicate declarations and unresolved references. Document the load order, whether lists and maps replace or merge, permitted operator overrides, and their precedence. Protect target identity and connection settings from conflicting overrides.

Bind scoped names in namespaced role input values or root composition values. Global constants may reference other global constants. Layer constants may reference global or same-layer constants. Plays may reference global or layer constants from their imported files. Resolve layer references in each layer that loads the consumer, and load layer-specific composition files only from the plays that use them. Each `lc_*` declaration requires at least two distinct direct consuming assignments in its layer, including play-level assignments and role invocation bindings. Each `ls_*` declaration requires at least one. Root composition assignments count when loaded in that layer. Repeated mentions, repeated loads, additional hosts, and transitive references add no consumers. Global declarations have no usage minimum.

### Vault

Encrypt each `*.vault.yml` file in full with Ansible Vault. Apply YAML document formatting to the plaintext and retain the ciphertext's Vault header in Git. Load Vaults only for selected capabilities, after the applicable target, identity, and readiness checks. A global Vault may serve any layer. Fail safely when required Vault access is unavailable. If using a Vault password file, use an ignored, owner-only `.vpass` in the repository root. Keep secrets out of logs, diagnostics, and diffs.

### Example

The `production` inventory contains `web-01`, and the `web` layer runs the reverse proxy role on that target. The inventory owns the host's HTTP port, and the layer maps it to the role's public input.

**`inventories/production/hosts.yml`**

```yaml
---

web_hosts:
  hosts:
    web-01:
      iv_http_port: 8080
```

**`variables/web/proxy.yml`**

```yaml
---

# Role: reverse_proxy

reverse_proxy_listen_port: "{{ iv_http_port }}"
```

**`playbooks/web/setup.yml`**

```yaml
---

- name: Configure the web stack
  hosts: web_hosts
  vars_files:
    - ../../variables/web/proxy.yml
  roles:
    - role: reverse_proxy
      tags: [proxy]
```

The project selects the production inventory and the web setup playbook through its documented target interface. The role consumes `reverse_proxy_listen_port` and can be reused with another inventory or with a caller-supplied port.

## Roles, assets, and tests

- Give each reusable role one capability, namespaced inputs, generic defaults, and validation for required or unsafe inputs. Use its own `<role>_*` variables and Ansible-provided variables. Apply the same namespace to facts, registered results, and custom loop bindings. Standard Ansible loop variables and Jinja-local bindings keep their normal names.
- Keep cross-capability ordering and exchange of outputs in the layer. Pass endpoints, paths, and runtime outputs through the receiving role's inputs. Compose sibling capabilities in the layer. Keep project inventory assumptions and sibling source-tree lookups out of reusable roles. Put shared implementation and activation behavior behind named helpers with explicit namespaced inputs and outputs.
- Where callers must interleave parts of a capability, expose focused entrypoints under `roles/<role>/phases/<phase>/`. Keep phase implementation and shared resources under the capability owner. Phase-specific inputs use the owning capability and phase namespace.
- Keep tasks, handlers, templates, installed programs, and single-owner assets with the role. Use fixed role-owned sources for repository-selected templates and runtime files. Distinguish controller source paths from managed-host destinations. When a template source must be configurable, validate the selected template against the role's variable contract before rendering and fail closed if its references cannot be analyzed safely.
- Keep single-role tests and their fixtures, inventories, scenarios, and helpers with the role. Keep cross-role tests and shared fixtures at their common owning scope. Install runtime files from an explicit selection that excludes test material.

## Migrations

Put each historical transition, one-time repair, or cutover in its own `migrations/<layer>/YYMMDDHHMMSS-slug/` directory. Use the owning layer, the migration's UTC creation timestamp, and a lowercase, hyphen-separated slug. Keep the directory name stable as the migration ID.

```text
migrations/<layer>/YYMMDDHHMMSS-slug/
  apply.yml       # Required forward playbook
  README.md       # Required operator notes
  rollback.yml    # Optional rollback playbook
  aux_script.py   # Example of a migration-specific helper
```

Keep migration-specific scripts, tasks, templates, fixtures, and tests in that directory. Keep shared helpers in their owning locations. A migration may call a reusable role to converge replacement state. Permanent entrypoints and roles stay free of historical detection and cleanup.

In the migration's README, record its purpose, affected targets, prerequisites, verification steps, and rollback or recovery procedure. The layer groups migrations by operational ownership. The project's dispatcher resolves and validates the inventory, host selection, and affected operation scope before loading credentials or performing side effects.

Select `apply.yml` for forward execution and invoke `rollback.yml` only through an explicit rollback operation. When a migration has no rollback playbook, follow its documented recovery procedure. When relocating an existing migration, update its callers and preserve its identity, execution order, and completion records.

Follow [Kerbmark’s infrastructure operations guidance](https://github.com/d1ys3nk0/kerbmark/blob/main/infrastructure/migrations.md#idempotent-migrations) for idempotency, migration outcomes, completion, and read-only previews. Document whether migrations run explicitly or before convergence, which inputs they require, and how the Ansible dispatcher represents each outcome and invokes rollback. Validate required inputs and inspect the historical state before the playbook makes changes.

For a replacement that transfers state, use two ordered forward migrations. Prepare and verify the replacement first, then remove obsolete resources after confirming cutover and readiness on the target. Identify superseded managed resources when designing the replacement and include their removal in the change. Preserve resources required for the agreed rollback. When migrations run before convergence and cleanup must wait, have the dispatcher report Kerbmark’s deferred outcome distinctly from success and failure.

## Verification

- Run structural and variable contract checks through normal local lint, hooks, and CI. Cover layout, section formatting, temporary placement, ownership, duplicate declarations, references, role invocations, and statically known templates. Fail on indirect access that cannot be analyzed safely. Report paths, locations, and identifiers without values.
- Run syntax and tag checks for supported targets through the normal dispatch inputs. Exercise focused prerequisites, opt-in behavior, overrides, credential isolation, and side-effect-free previews. For layout changes, compare effective non-secret inputs, role order, and tag selection, including active migration and rollback callers.
- Verify public structure and staged ciphertext without the owner password. Reject tracked password files. Use synthetic Vault fixtures for public tests of decryption, key rules, and failure paths. Before committing, use owner-private access to verify decrypted section formatting, declarations, and layer secret usage.
- Test migrations against fresh state, recognized historical state, interruption, repeated runs, refusal paths, and preview. Verify actual side effects as well as reported changes.
