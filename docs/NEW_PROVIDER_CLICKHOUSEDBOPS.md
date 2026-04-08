# Proposal: New `pulumi-clickhousedbops` Provider

## Context

GitHub Issue: https://github.com/pulumiverse/pulumi-clickhouse/issues/7

Users have requested management of ClickHouse **users, roles, and permissions** via Pulumi.
This functionality exists in Terraform through a separate provider:
[`ClickHouse/terraform-provider-clickhousedbops`](https://github.com/ClickHouse/terraform-provider-clickhousedbops).

Following Pulumi's standard pattern (one TF provider = one Pulumi provider), this requires
a new `pulumi-clickhousedbops` provider, similar to how Azure has separate `pulumi-azure`
and `pulumi-azuread` packages.

## Resources to Bridge

The `clickhousedbops` TF provider (v1.9.0) exposes 5 resources:

| Terraform Resource | Pulumi Resource | Description |
|---|---|---|
| `clickhousedbops_database` | `Database` | Create/manage databases |
| `clickhousedbops_user` | `User` | Create/manage users |
| `clickhousedbops_role` | `Role` | Create/manage roles |
| `clickhousedbops_grant_role` | `GrantRole` | Assign roles to users/roles |
| `clickhousedbops_grant_privilege` | `GrantPrivilege` | Grant specific privileges |

### Provider Configuration

The `clickhousedbops` provider requires:

```hcl
provider "clickhousedbops" {
  protocol = var.protocol    # Connection protocol
  host     = var.host        # ClickHouse server hostname
  port     = var.port        # ClickHouse server port
  auth_config {
    strategy = var.strategy  # Authentication strategy
    username = var.username
    password = var.password
  }
}
```

## Steps to Create the Provider

### 1. Request a New Repo in Pulumiverse

Open a PR against [`pulumiverse/infra`](https://github.com/pulumiverse/infra) to add the
new repository configuration.

**Create file `02-repositories/clickhousedbops.yaml`:**

```yaml
name: pulumi-clickhousedbops
description: Pulumi provider for ClickHouse database operations (users, roles, grants)
type: provider
```

**Update team/member configs** (if needed) in `01-teams/` and `03-members/` to grant
the appropriate team push access to the new repo.

Once merged, the Pulumiverse infra Pulumi program will provision the GitHub repo
with standard settings (branch protection, team access, etc.).

### 2. Scaffold the Provider

Use the official boilerplate:

1. Create the repo from the
   [`pulumi/pulumi-tf-provider-boilerplate`](https://github.com/pulumi/pulumi-tf-provider-boilerplate)
   template (or run `setup.sh` after cloning)
2. Replace all `xyz` references with `clickhousedbops`
3. Update `provider/go.mod` to depend on the upstream TF provider:
   ```
   require github.com/ClickHouse/terraform-provider-clickhousedbops v1.9.0
   ```
4. Update `provider/resources.go`:
   - Import the upstream provider package
   - Shim it with `pf.ShimProvider()`
   - Set provider metadata:
     ```go
     Name:              "clickhousedbops",
     DisplayName:       "ClickHouse Database Operations",
     Publisher:         "pulumiverse",
     GitHubOrg:         "ClickHouse",
     TFProviderVersion: "1.9.0",
     PluginDownloadURL: "github://api.github.com/pulumiverse/pulumi-clickhousedbops",
     ```
   - Map resources (autodiscovery via `MustComputeTokens` should handle most, but
     add explicit mappings for any resources needing custom `ComputeID`)

### 3. Build and Generate SDKs

```bash
cd provider && go mod tidy && cd -
make tfgen       # Generate schema from TF provider
make provider    # Build the provider binary
make build_sdks  # Generate Go, Node.js, Python, .NET, Java SDKs
cd sdk && go mod tidy && cd -
```

### 4. Configure CI/CD

The repo will need these GitHub secrets for publishing:
- `NPM_TOKEN` — for `@pulumiverse/clickhousedbops` on npmjs.com
- `NUGET_PUBLISH_KEY` — for `Pulumiverse.Clickhousedbops` on nuget.org
- `PYPI_PASSWORD` — for `pulumiverse_clickhousedbops` on PyPI

### 5. Register in Pulumi Registry

Add an entry to
[`pulumi/registry`](https://github.com/pulumi/registry) in
`community-packages/package-list.json` so the provider appears on
https://www.pulumi.com/registry/.

### 6. Add Examples

Create example programs demonstrating usage:

```typescript
import * as clickhouse from "@pulumiverse/clickhouse";
import * as dbops from "@pulumiverse/clickhousedbops";

// Cloud infrastructure
const service = new clickhouse.Service("my-service", {
    // ...service config...
});

// Database operations (connects to the service)
const db = new dbops.Database("app-db", {
    clusterName: "my-cluster",
    name: "app",
    comment: "Application database",
});

const role = new dbops.Role("writer-role", {
    clusterName: "my-cluster",
    name: "writer",
});

const user = new dbops.User("app-user", {
    clusterName: "my-cluster",
    name: "app_user",
    passwordSha256Hash: "...",
});

const grant = new dbops.GrantRole("writer-to-user", {
    clusterName: "my-cluster",
    roleName: role.name,
    granteeUserName: user.name,
});

const privilege = new dbops.GrantPrivilege("insert-on-app", {
    clusterName: "my-cluster",
    privilegeName: "INSERT",
    databaseName: db.name,
    granteeRoleName: role.name,
    grantOption: false,
});
```

## Checklist

- [ ] PR to `pulumiverse/infra` adding `02-repositories/clickhousedbops.yaml`
- [ ] Scaffold provider from `pulumi-tf-provider-boilerplate`
- [ ] Configure `resources.go` with upstream TF provider bridge
- [ ] `make tfgen && make build_sdks` succeeds
- [ ] CI/CD secrets configured
- [ ] Example programs for each resource
- [ ] PR to `pulumi/registry` for registry listing
- [ ] Update pulumiverse/pulumi-clickhouse#7 with link to new provider
