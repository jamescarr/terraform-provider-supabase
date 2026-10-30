# Provider roadmap

Checklist plan for `/plan-loop`. Every `- [ ]` line is one independently shippable slice (one PR,
CI green, squash-merged). Nested lines under a bullet are its acceptance criteria. `(after: N)`
lists prerequisite bullet numbers; bullets are numbered by file order, checked ones included.

Research behind this plan (2026-10-28): the provider ships 7 resources + 4 data sources over 43
of the 219 Management API operations; it pins `github.com/supabase/cli/pkg v1.2.3` (April 2026),
which is the last tag the Go proxy can resolve because upstream moved the module to
`apps/cli-go/pkg` without changing its module path. The upstream generated client covers only
`/v1`; the 50 `/v2` operations (log drains, webhooks, org members, private link) have no Go client
anywhere. Upstream has open PRs for most known bugs; those are ported here with attribution.

## Conventions for every bullet

- Follow `internal/provider/third_party_auth_resource.go` as the exemplar: `RequiresReplace` on
  identity attributes, `Read` returns `(found, diags)` and the caller calls `RemoveResource` on
  404, idempotent 404-tolerant `Delete`, validated multi-part import IDs, readiness waits via
  `utils.go` where the API needs an ACTIVE project.
- Tests are hermetic gock tests named `TestAcc<Resource>_<Case>` gated by `TF_ACC=1` (CI sets it),
  plus plain unit tests for pure helpers. Mock paths go in `constants_test.go`.
- Every new resource/data source/action ships with `examples/<kind>/supabase_<name>/*.tf`,
  regenerated `docs/` (`go generate ./...` must leave no diff) and `docs/schema.json`.
- Singleton ("settings-style") resources import by `project_ref`, use `id = project_ref`, and
  document what `Delete` does (reset to the API default where one exists, otherwise a documented
  no-op). They never hold the whole API payload as one opaque JSON string.
- Sensitive values the API never returns (passwords, provider secrets, hook secrets) are kept from
  prior state on Read and documented as write-only on the API side.
- No bullet widens scope into neighbouring resources; file a follow-up bullet instead.
- The upstream Go client is generated from a staging spec (`api.supabase.green/api/v1-yaml`).
  Before building on a client method added after `pkg v1.2.3`, confirm the operation exists in
  production (`https://api.supabase.com/api/v1-json`) and say so in the PR.
- Upstream PR and issue numbers (`#NNN`) refer to `supabase/terraform-provider-supabase`. Read
  them with `gh pr view NNN --repo supabase/terraform-provider-supabase` and
  `gh pr diff NNN --repo supabase/terraform-provider-supabase`; never without `--repo`. A ported
  commit ends with one `Co-authored-by: <name or login> <ID+login@users.noreply.github.com>`
  trailer per upstream PR author (`gh api users/<login> --jq .id` gives the ID).
- This fork has no PR CI. Before merging, run all of the following in the worktree (each wrapped
  in the plan-loop lock) and list them in the PR body:
  `go mod tidy && git diff --exit-code go.mod go.sum`, `go build ./...`, `golangci-lint run`,
  `go generate ./... && git diff --exit-code`, `TF_ACC=1 go test -cover ./internal/provider/`,
  and `make generate-json`. Commit the resulting `docs/schema.json`; upstream's bot does this in
  CI, but it does not run here.
- Local tools come from the parent-directory `mise.toml` (Go 1.26.10, Terraform 1.17.1,
  golangci-lint 2.15.0, goreleaser 2.18.2). Do not add mise or tool-version files to the repo.

## Phase 0: client and dependency unblocker

- [x] Switch to the upstream subdirectory module via `replace` and migrate the `Output` renames
  - `go.mod` keeps `require github.com/supabase/cli/pkg v1.2.3` and adds
    `replace github.com/supabase/cli/pkg => github.com/supabase/cli/apps/cli-go/pkg v0.0.0-20261028134026-c72507449430`
    (or the current `@develop` pseudo-version at implementation time). No fork, no vendoring.
  - All 60 missing `api.*` identifiers are migrated: response types gain the `Output` suffix
    (`SecretResponseOutput`, `AuthConfigResponseOutput`, `ThirdPartyAuthOutput`,
    `V1ProjectWithDatabaseResponseOutputStatus*`, `ApiKeyResponseOutputType*`,
    `SupavisorConfigResponseOutput*`, `V1ServiceHealthResponseOutputName*`, …),
    `CreateApiKeyBodyTypeSecret/Publishable` become `api.Secret`/`api.Publishable`, and the
    addon variant accessor becomes `AsListProjectAddonsResponseOutputSelectedAddonsVariantId0()`.
  - `go build ./...`, `golangci-lint run`, `TF_ACC=1 go test ./internal/provider/` and
    `go generate ./...` (no diff) all pass.
  - `CONTRIBUTING.md` gains a "Bumping the Management API client" section: the exact
    `go mod edit -replace github.com/supabase/cli/pkg=github.com/supabase/cli/apps/cli-go/pkg@develop && go mod tidy`
    procedure, the note that Dependabot cannot bump a `replace`, and the expectation that spec
    syncs may rename generated types and track staging ahead of production.
- [ ] Generate a provider-local `/v2` client in `internal/apiv2` (after: 1)
  - `internal/apiv2/v2-openapi.json` is a committed snapshot of `https://api.supabase.com/api/v2-json`;
    `internal/apiv2/{types,client}.gen.go` are produced from it by `oapi-codegen` (same major as
    upstream, run via `go tool`, config files alongside, `nullable-type: true`), with an
    `overlay.yaml` only if codegen cannot consume a schema as-is.
  - `make generate-apiv2` refreshes the snapshot and regenerates; `main.go` `//go:generate`
    regenerates from the committed snapshot; the CI `generate` job fails on any diff.
  - `provider.Configure` builds an `*apiv2.ClientWithResponses` with the same base URL, bearer
    token, User-Agent and GET-only retry transport as the v1 client; `configure_client.go`
    exposes both clients to resources through one provider-data struct, and every existing
    resource/data source still compiles against it.
  - A unit test proves the v2 client sends `Authorization: Bearer` and retries a GET 502.
- [ ] Repo hygiene: CI Terraform version, release and contributor docs match the code
  - `.github/workflows/test.yml` runs the `test` job on Terraform `1.14.x` (write-only
    arguments need 1.11+, ephemeral resources 1.10+, actions 1.14+); `docs/index.md` states the
    minimum supported Terraform version.
  - `main.go` declares `commit` and `main.go` reports it alongside `version`, or
    `.goreleaser.yml` stops setting `main.commit`; `goreleaser build --snapshot --clean` passes.
  - `CONTRIBUTING.md` and `GNUmakefile` comments no longer claim acceptance tests create real,
    billable resources; they state that `TF_ACC=1` tests run against gock mocks and that
    `.go-version` is the toolchain CI uses.
  - `templates/resources/edge_function_secrets.md.tmpl` describes the real apply order (upsert
    first, then delete removed names) and `docs/` is regenerated.

## Phase 1: correctness fixes to existing resources

- [ ] Port upstream PR #380: `supabase_project` reads `instance_size = "nano"` when no compute addon is selected (after: 1)
  - Closes upstream issue #372. `Read` sets `instance_size` to `nano` when
    `V1ListProjectAddons` returns no `compute_instance` addon, and still derives other sizes from
    the addon variant id (new union accessor from bullet 1).
  - `TestAccProjectResource_NanoInstanceSize` creates with `nano`, refreshes, and shows no diff;
    the existing project tests still pass.
- [ ] Port upstream PR #363: `supabase_edge_function_secrets` rejects duplicate secret names (after: 1)
  - Closes upstream issue #362. Duplicate names known at plan time fail validation with a
    diagnostic that names the duplicate but never includes a value; names only known at apply
    time are checked before any `V1BulkCreateSecrets` call.
  - Tests cover plan-time rejection, apply-time rejection with an unknown-at-plan value, and
    that the happy path is unchanged.
- [ ] Port upstream PR #364: write-only `database_password_wo` on `supabase_project` (after: 3, 4)
  - Closes upstream issue #353. `database_password` becomes Optional; `database_password_wo`
    (Optional, Sensitive, WriteOnly) and `database_password_wo_version` (Optional) are added with
    `ExactlyOneOf(database_password, database_password_wo)`.
  - `Create` reads the write-only value from `req.Config`; `Update` rotates when
    `database_password_wo_version` changes; the write-only value is never written to state or plan.
  - `docs/resources/project.md` explains both forms and that `database_password` is stored in
    state; tests cover create, rotation via version bump, and the mutual-exclusion validator.
    (Terraform `1.14.x` from bullet 3).
- [ ] Port upstream PRs #369 and #381: `supabase_apikey` manages publishable keys, custom roles, and keys deleted out of band (after: 1)
  - Closes upstream issue #359. `type` is Optional+Computed (`secret` default, `publishable`
    allowed, `RequiresReplace`); `secret_jwt_template.role` is Optional+Computed and sent on
    create/update for secret keys; `description` round-trips.
  - `project_ref` has `RequiresReplace`; `Read` returns not-found on 404 and the caller removes
    the resource from state instead of erroring.
  - Tests cover publishable create, custom role create/update, 404 removal, and import of a
    publishable key by `<project_ref>/<name>/publishable`.
- [ ] Port upstream PR #376: `supabase_settings.auth` preserves `before`/`after user created` hook secrets (after: 1)
  - The hook secret fields added upstream are included in `copySensitiveFields`, so a plan after
    apply shows no diff; `docs/resources/settings.md` lists them among the write-only fields.
  - One gock test applies a hook config with secrets and refreshes without drift.
- [ ] `supabase_project` drift and replace semantics (after: 4, 6)
  - `Read` on 404 removes the resource from state (plan recreates) instead of leaving it; the
    terminal statuses `REMOVED`/`GOING_DOWN` after a successful GET are treated the same way.
  - `organization_id` and `region` carry `RequiresReplace`; the apply-time "Update is not
    supported" errors for them are deleted.
  - Docs state that `database_password` is never read back and that import leaves it unset.
  - Tests cover 404 removal and a plan that shows replacement on region change.
- [ ] `supabase_settings` drift, replace, and the dead `pooler` attribute (after: 8)
  - `project_ref` carries `RequiresReplace`.
  - A 404 on the project (`V1GetProject`) during `Read` removes the resource from state; a 404 or
    406 from an individual settings endpoint while the project exists keeps that attribute as-is
    and emits a warning naming the endpoint.
  - `pooler` is wired to `V1GetPoolerConfig`/`V1UpdatePoolerConfig` using the same
    `parseConfig` key-preserving pattern as the other categories, or removed with a deprecation
    if the API payload cannot round-trip; the choice is documented.
  - Tests cover project 404 removal, endpoint 404 warning, and pooler create/update/import.
- [ ] `supabase_branch` import and refresh (after: 1)
  - Import accepts `<parent_project_ref>/<branch_id>` and sets both attributes; bare
    `<branch_id>` still works and is documented as losing `persistent`.
  - `Read` refreshes `git_branch` and `persistent` from `V1GetABranchConfig`/`V1ListAllBranches`
    and removes the resource on 404; `region` is documented as create-only.
  - Tests cover both import forms, 404 removal, and an out-of-band `git_branch` rename producing a
    diff.
- [ ] `supabase_edge_function_secrets` validates reserved names client-side (after: 5)
  - A `name` validator rejects the `SUPABASE_` prefix at plan time; the two
    `*ReservedPrefixFails` tests assert the validator error instead of a mocked API error.
  - `Read` with an empty remote list and a non-empty state removes the resource.

## Phase 2: per-service configuration resources (replacing the `supabase_settings` blob)

- [ ] `supabase_network_restrictions` resource (after: 1)
  - Attributes: `project_ref`, `db_allowed_cidrs` (set), `db_allowed_cidrs_v6` (set), computed
    `status`/`entitlement`. Create/Update call `V1UpdateNetworkRestrictions` (apply) and poll
    `V1GetNetworkRestrictions` until the config is applied; private ranges are rejected at plan time.
  - `Delete` restores allow-all (`0.0.0.0/0`, `::/0`); import by `project_ref`.
- [ ] `supabase_ssl_enforcement` resource (after: 1)
  - Attributes: `project_ref`, `database` (bool). `Create`/`Update` use
    `V1UpdateSslEnforcementConfig` and poll until applied; the "not allowed to configure SSL
    enforcements" 400 is surfaced verbatim; `Delete` disables enforcement.
- [ ] `supabase_postgres_config` resource (after: 1)
  - Every field of `UpdatePostgresConfigBody` (43 at head) is a typed Optional+Computed attribute
    with enum validators where the API defines them, plus `restart_database` (write-only trigger).
  - Update sends only user-set keys (PUT with the merged current config), so unset attributes are
    never clobbered; `Delete` is a documented no-op.
- [ ] `supabase_storage_config` resource (after: 1)
  - Typed attributes for `file_size_limit`, `features` (image transformation, s3 protocol,
    icebergCatalog as present at head), `object_versioning`; computed `migration_version`.
  - `Delete` is a documented no-op.
- [ ] `supabase_pooler_config` resource (after: 1)
  - Attributes: `project_ref`, `default_pool_size`, `max_client_conn`, `pool_mode` (enum);
    computed connection map identical to the `supabase_pooler` data source so users can migrate.
  - `Delete` is a documented no-op.
- [ ] `supabase_postgrest_config` resource (after: 1)
  - Typed `db_schema`, `db_extra_search_path`, `max_rows`, `db_pool`,
    `db_pool_acquisition_timeout`; `jwt_secret` is excluded and documented as unmanageable.
  - `Delete` is a documented no-op.
- [ ] `supabase_auth_config` resource, core settings (after: 1)
  - Typed nested blocks for every non-`external_*` field of `UpdateAuthConfigBody`: `site`,
    `smtp`, `mailer` templates/subjects, `security`, `sessions`, `mfa`, `rate_limits`, `hooks`,
    `sms`, `password`, `jwt`. Write-only fields (`smtp_pass`, hook secrets, SMS provider secrets)
    are kept from prior state on Read.
  - Only user-set attributes are sent on PATCH; unknown server fields are ignored rather than
    erroring; import by `project_ref`; `Delete` is a documented no-op.
- [ ] `supabase_auth_config` external providers block (after: 19)
  - `external_providers` map keyed by provider name (`google`, `github`, `apple`, …) with
    the shared field set (`enabled`, `client_id`, `secret`, `additional_client_ids`, `url`,
    `skip_nonce_check`, `email_optional`); unsupported fields for a given provider fail
    validation. `apple.additional_client_ids` round-trips as today's `preserveAppleAdditionalClientIDs`.
- [ ] Deprecate `supabase_settings` (after: 10, 13, 14, 15, 16, 17, 18, 19)
  - The resource and each attribute carry a `DeprecationMessage` naming its replacement;
    `docs/guides/settings-migration.md` shows the `import`/`removed` flow for each category.
  - The resource keeps working; no existing test is deleted.

## Phase 2b: new project-scoped resources

- [ ] Port upstream PR #366: `supabase_custom_hostname` resource (after: 1)
  - Create initialises the hostname, re-verifies DNS, and activates once `status` allows;
    `activate` can be deferred via an attribute so users can set DNS records first; computed
    `ssl`/`ownership_verification` records are exposed for DNS automation.
  - Delete removes the config; import by `project_ref`; tests cover each state transition.
- [ ] `supabase_vanity_subdomain` resource (after: 1)
  - Create checks availability then activates; `Read` uses `V1GetVanitySubdomainConfig`; update
    of `subdomain` replaces; `Delete` deactivates; import by `project_ref`.
- [ ] `supabase_signing_key` resource (after: 1)
  - Create makes a standby key (`algorithm` with enum validator); `status` is Optional+Computed
    and drives `V1UpdateProjectSigningKey` transitions (`standby` → `in_use` →
    `previously_used` → `revoked`); `Delete` calls `V1RemoveProjectSigningKey` and surfaces the
    API error when the key is not yet removable; import by `<project_ref>/<id>`.
- [ ] `supabase_sso_provider` resource (after: 1)
  - SAML provider with `metadata_url` or `metadata_xml` (ExactlyOneOf), `domains`,
    `attribute_mapping` (normalized JSON), `name_id_format`; full CRUD via the
    `/config/auth/sso/providers` ops; import by `<project_ref>/<provider_id>`.
- [ ] Port upstream PR #365: `supabase_apikey` ephemeral resource (after: 3, 7)
  - Opens a short-lived secret key and closes it on `Close`; supports both key types; never
    touches state; docs note the Terraform version requirement.
- [ ] Port upstream PR #367: `supabase_vault_secret` resource via `V1RunAQuery` (after: 3)
  - Closes upstream issue #360. Create/Update/Delete run parameterised `vault.create_secret`,
    `vault.update_secret`, `DELETE FROM vault.secrets` statements; `Read` resolves by id, falling
    back to name on import (`<project_ref>/<name>`); `secret` is Sensitive and never logged in a
    diagnostic, with `secret_wo`/`secret_wo_version` offered as the write-only alternative
    (same shape as bullet 6); the docs state that secret values transit the Management API
    `database/query` endpoint.
- [ ] `supabase_storage_bucket` resource via the project Storage API (after: 1)
  - Uses `github.com/supabase/cli/pkg/storage` against `https://<project_ref>.supabase.co` (or
    the endpoint-derived host) authenticated with a `secret` (or legacy `service_role`) key
    fetched through `V1GetProjectApiKeys` with reveal; a clear error when neither exists. The
    resolved key and Storage client are cached per `project_ref` for the provider process
    (single-flight), so a refresh of N buckets costs one Management API call per project, and
    `Read` fetches the single bucket (`GET /storage/v1/bucket/{id}`) rather than listing all.
  - Attributes: `name` (RequiresReplace), `public`, `file_size_limit`, `allowed_mime_types`;
    import by `<project_ref>/<name>`; tests mock both the Management API and the Storage API.
- [ ] `supabase_realtime_config`, `supabase_backup_schedule` and `supabase_pgsodium_config` resources (after: 1)
  - Three small singletons on `V1Get/UpdateRealtimeConfig`, `V1Get/UpdateBackupSchedule`,
    `V1Get/UpdatePgsodiumConfig`; `root_key` on pgsodium is Sensitive and its docs warn that
    rotation makes old data unreadable. Each ships its own tests and docs; one PR is acceptable.
- [ ] `supabase_disk` resource (after: 1)
  - `size_gb` and `type` (enum) drive `V1ModifyDatabaseDisk`; computed autoscale config and
    utilisation from the read endpoints; shrink attempts fail at plan time; polls project status
    after resize; `Delete` is a documented no-op.

## Phase 3: `/v2` resources (need bullet 2)

- [ ] `supabase_log_drain` resource (after: 2)
  - `type` (enum of drain types at head), `name`, `config` (nested per type), `description`;
    full CRUD on `/v2/projects/{ref}/analytics/log-drains`; import by `<project_ref>/<id>`.
- [ ] `supabase_project_webhook_endpoint` resource (after: 2)
  - `url`, `description`, `enabled`, `event_types`, computed `signing_secret` (Sensitive);
    CRUD on `/v2/projects/{ref}/webhooks/endpoints`; import by `<project_ref>/<id>`.
- [ ] `supabase_organization_webhook_endpoint` resource (after: 32)
  - Same shape as the project endpoint resource on `/v2/organizations/{slug}/webhooks/endpoints`;
    shared helpers are factored out of bullet 32 rather than copied.
- [ ] `supabase_private_link_association` resource (after: 2)
  - `project_ref`, `aws_account_id`, optional `database_identifier`; create/list/delete on the
    private-link ops; `Read` uses the list op; import by `<project_ref>/<aws_account_id>`.
- [ ] `supabase_organization_member` resource (after: 2)
  - Create sends an invitation (`email`, `role_id`); `Read` resolves the member by email across
    members and pending invitations; `Update` changes the role for accepted members; `Delete`
    deletes a pending invitation, and for accepted members emits a warning that the API has no
    removal op and drops the resource from state. Docs lead with these limits.
  - `supabase_organization_roles` data source ships in the same PR so `role_id` is discoverable.

## Phase 4: data sources and actions

- [ ] Organization and project data sources (after: 1)
  - `supabase_organization` (by slug), `supabase_organizations`, `supabase_project` (by ref),
    `supabase_projects` (filtered by organization); each exposes the non-sensitive fields of the
    corresponding `V1*` responses.
- [ ] `supabase_available_regions` and `supabase_typescript_types` data sources (after: 1)
  - Regions list for validating `supabase_project.region`; generated TypeScript types with
    `included_schemas` input for CI codegen.
- [ ] `supabase_advisors` data source (after: 1)
  - `kind` (`security`|`performance`) returns lint results from the two `/advisors` ops for use
    in CI gates.
- [ ] Project lifecycle actions: `supabase_project_pause`, `supabase_project_restore`, `supabase_project_restart` (after: 3)
  - Implemented with the plugin-framework `action` package; each invokes the corresponding
    `V1*AProject` op and waits for a terminal status with a configurable timeout; docs state the
    minimum Terraform version and that actions are not tracked in state.
- [ ] `supabase_project_upgrade` action (after: 39)
  - Checks `V1GetPostgresUpgradeEligibility`, invokes `V1UpgradePostgresVersion` with
    `target_version`, polls `V1GetPostgresUpgradeStatus` until completion; ineligibility is a
    clear error before any mutation.

## Not planned

- Read replicas (`/read-replicas/setup|remove`) have no read or list op; nothing to reconcile.
- Branch push/merge/reset/diff, PITR/physical restores, migrations, transfers, compute instances
  and the OAuth flow are imperative or CLI-owned.
- Storage objects, database webhooks (SQL triggers) and anything else that is a database object
  rather than Management API state stay out until a SQL-backed pattern has proven itself with
  `supabase_vault_secret`.
- Upstream: once bullet 1 lands, offer the `replace` line and the `Output` sweep to
  `supabase/terraform-provider-supabase` as a PR; it is the smallest fix for the same breakage.
