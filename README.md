# Infisical secrets operator

## Purpose

This wrapper installs the upstream `secrets-operator` chart at `v0.11.11`.
It also owns the shared `InfisicalConnection` and `InfisicalAuth` used by
other charts. The chart version is `1.0.0`. CI publishes it to GHCR so
that the secrets operator does not depend on Harbor.

## Before first install

1. Confirm that `infisical/universal-auth-credentials` exists. It must have
   the keys `clientId` and `clientSecret`. Do not put their values in Git.
2. Use the existing release name `infisical-secrets-operator` and namespace
   `default`. Other charts use `default/infisical-prod` and
   `default/infisical-prod-universal-auth`.
3. Use `cluster-config/apps/dev/infisical-secrets-operator.yaml` for the
   API address and to enable the shared objects.
4. After CI publishes the first chart, replace `PENDING_CI_VERSION` in
   `cluster-config/argo/dev/infisical-secrets-operator.yaml` with its exact
   version. Review the Argo resource diff before a manual sync.
5. Keep all seven CRDs tracked. Do not disable `secrets-operator.installCRDs`.

No cluster changes are part of this local build.

## Values reference

| Value | Default | Use |
| --- | --- | --- |
| `connection.enabled` | `false` | Create the shared connection. |
| `connection.name` | `infisical-prod` | Name used by the shared auth. |
| `connection.address` | empty | Infisical API URL, including `/api`. Required when enabled. |
| `auth.enabled` | `false` | Create the shared universal auth. |
| `auth.name` | `infisical-prod-universal-auth` | Name used by other charts. |
| `auth.credentialsRef.name` | `universal-auth-credentials` | Existing credentials Secret. |
| `auth.credentialsRef.namespace` | `infisical` | Credentials Secret namespace. |
| `auth.credentialsRef.clientIdKey` | `clientId` | Client ID key name only. |
| `auth.credentialsRef.clientSecretKey` | `clientSecret` | Client secret key name only. |
| `secrets-operator` | `{}` | Values passed to upstream without template changes. |

The connection and auth use the release namespace. Auth can also reference
an existing connection when `connection.enabled` is false. Keep
`secrets-operator.hostAPI` and `connection.address` equal in the environment
file. The first supports the legacy secret resources; the second supports
the shared v1beta1 connection.

## Upgrades

Change the dependency version and `appVersion` together. Use
`/opt/homebrew/bin/helm dependency update charts` with an isolated Helm
cache. Keep the generated `Chart.lock`. Run the local wrapper checks,
render with the environment values, and compare with live resources.
Review CRD schema changes before a manual sync. Do not upgrade the operator
as part of adoption.

## Adoption notes

The existing Application is `infisical-secrets-operator`. It tracks
19 objects, including seven CRDs. The upstream release, image, replica
count, arguments, resources and RBAC stay unchanged. The CRDs are upstream
templates controlled by `installCRDs`, not files in a `crds/` directory.
No `skipCrds` option is added.

The two wrapper objects keep their current names, namespace, specs and
credential references. The wrapper adds `app.kubernetes.io/name`,
`app.kubernetes.io/instance`, `app.kubernetes.io/managed-by` and
`helm.sh/chart` labels. These ownership labels are the only intended
resource differences. No secret resource is migrated or pruned.

The new Application uses project `platform`, two sources and GHCR, as
required by the wrapper layout. It has no automated sync. Live has no
sync options or ignore rules to preserve. `Delete=false` prevents the
root Application from deleting it.

Local results and the exact parity output are in
`.omo/evidence/wrappers/infisical-secrets-operator/` in the HomeLab workspace.

The local check passed with exit code 0: four suites and 12 tests passed.
Kubeconform validated 12 resources in the all-on render and 10 in the
default render; each skipped only the seven CRDs, with no errors.
Live parity was `SAME=17 CHANGED=2 NEW=0 PRUNE=0 HOOK=0 UNKNOWN-KIND=0`.
The changed objects are `default/infisical-prod-universal-auth`
(`InfisicalAuth`) and `default/infisical-prod` (`InfisicalConnection`).
Each differs only by the three `app.kubernetes.io` labels listed above.
The parity tool ignores `helm.sh/chart`; that label is also added.

The datree catalog has v1beta1 schemas for both kinds, and the all-on
kubeconform run validated them. No shared-library skip-list change is
needed.
