# Databricks System Tables View Creation Script

## Overview

This Jupyter notebook automates the creation of workspace-scoped views from Databricks system tables. It enables workspace administrators to access system table data filtered to their specific workspace, without requiring account-level permissions.

## Purpose

Databricks system tables (in the `system` catalog) contain valuable operational and usage data across all workspaces in an account. However, these tables typically require account-level permissions to access. This script:

1. **Validates permissions** - Confirms the requesting user is a workspace admin for the specified workspace
2. **Grants system table access** - Temporarily grants SELECT permissions on system tables to an admin service principal
3. **Creates filtered views** - Generates views of system tables filtered by `workspace_id`
4. **Manages access control** - Sets up appropriate permissions on the target catalog and schema

## Prerequisites

### Required Permissions
- **Account Admin** access to run this notebook
- An **account-level service principal** with admin privileges

### Required Resources
- Databricks workspace with:
  - Unity Catalog enabled
  - DBR (Databricks Runtime) - latest version or serverless compute recommended
  - Ability to create catalogs and schemas

### Dependencies
- `databricks.sdk` - Databricks SDK for Python
- `delta.tables` - Delta Lake operations
- PySpark libraries (included in DBR)

## Configuration

### Step 1: Create Account-Level Service Principal

If you don't already have an account-level service principal:

1. Follow [Step 1 of the OAuth M2M documentation](https://docs.databricks.com/en/dev-tools/auth/oauth-m2m.html#step-1-create-a-service-principal)
2. Generate a client secret following [Step 3](https://docs.databricks.com/en/dev-tools/auth/oauth-m2m.html#step-3-create-an-oauth-secret-for-a-service-principal)

### Step 2: Configure the Notebook

In **Cell 0**, update the following configuration values:

```python
# Account host - choose the appropriate one for your cloud:
# - AWS: "accounts.cloud.databricks.com"
# - Azure: "accounts.azuredatabricks.net"
# - GCP: "accounts.gcp.databricks.com"
HOST = "XXX"

# Your Databricks account ID (format: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx)
ACCOUNT_ID = "XXX"

# Service principal credentials
CLIENT_ID = "XXX"
CLIENT_SECRET = "XXX"  # Consider using Databricks Secrets instead
```

**Security Best Practice**: Instead of hardcoding `CLIENT_SECRET`, use [Databricks Secrets](https://docs.databricks.com/en/security/secrets/index.html):

```python
CLIENT_SECRET = dbutils.secrets.get(scope="your_scope", key="your_key")
```

In **Cell 3**, specify the admin service principal:

```python
principal = "XXX"  # Replace with your admin SPN or admin user
```

### Step 3: Set Widget Parameters

When running the notebook, provide these parameters via widgets:

| Parameter | Description | Example |
|-----------|-------------|---------|
| `workspace_id` | The workspace ID for which to create views | `1234567890123456` |
| `user_name` | Email of the workspace admin requesting access | `user@company.com` |
| `target_catalog` | Name of the catalog where views will be created | `workspace_system_tables` |
| `target_schema` | Name of the schema where views will be created | `ws_1234567890123456` |

## How It Works

### Workflow

1. **Authentication** (Cell 0)
   - Authenticates to Databricks Account using OAuth M2M
   - Initializes the AccountClient with service principal credentials

2. **Permission Validation** (Cells 1-2)
   - Retrieves workspace permissions for the specified workspace
   - Validates that `user_name` is a workspace admin
   - Grants temporary SELECT permissions on system tables to the admin principal

3. **View Creation** (Cell 3)
   - **If user is a workspace admin**:
     - Creates the target catalog and schema
     - Grants permissions to both the admin principal and requesting user
     - Iterates through all system tables (excluding specific schemas/tables)
     - Creates views filtered by `workspace_id`
     - Revokes temporary admin permissions
   - **If user is not a workspace admin**:
     - Displays an error message and exits

### Tables Included

The script creates views for all tables in the `system` catalog **except**:

**Excluded Schemas:**
- `information_schema`
- `marketplace`

**Excluded Tables:**
- `clean_room_events`
- `list_prices`
- `node_types`

Tables without a `workspace_id` column are also excluded.

### View Naming Convention

Views are named using the pattern: `{schema_name}_{table_name}_view`

**Examples:**
- `system.access.audit` → `access_audit_view`
- `system.compute.clusters` → `compute_clusters_view`
- `system.billing.usage` → `billing_usage_view`

## Usage

### Running the Notebook

1. Open the notebook in a Databricks workspace
2. Ensure you're using the latest DBR version or serverless compute
3. Update all configuration values in Cell 0 and Cell 3
4. Set the widget parameters (workspace_id, user_name, target_catalog, target_schema)
5. Run all cells sequentially

### Accessing the Views

After successful execution, users can query the views:

```sql
SELECT * FROM {target_catalog}.{target_schema}.access_audit_view
WHERE event_date >= current_date() - 7
```

## Permissions Model

### Granted Permissions

- **During Execution**: Admin principal receives temporary SELECT on system tables
- **Post-Execution**: 
  - Requesting user (`user_name`) receives ALL PRIVILEGES on the target catalog and schema
  - Admin principal permissions are revoked from the schema

### Customizing Permissions

Modify the GRANT statements in Cell 3 to adjust permissions:

```python
# Example: Grant only SELECT instead of ALL PRIVILEGES
spark.sql(f"GRANT SELECT ON SCHEMA {target_write_path} TO `{user_name}`")
```

## Troubleshooting

### Common Issues

| Issue | Possible Cause | Solution |
|-------|----------------|----------|
| "User is not a workspace admin" | User doesn't have admin role in workspace | Verify workspace permissions in Account Console |
| "Unable to create the target catalog or schema" | Invalid catalog/schema name or insufficient permissions | Check naming conventions and account admin access |
| Authentication errors | Invalid service principal credentials | Verify CLIENT_ID and CLIENT_SECRET are correct |
| "workspace_id column not found" | Table doesn't support workspace filtering | Table is intentionally excluded; this is expected |

### Debugging Tips

1. **Check workspace permissions**:
   ```python
   workspace_permissions_lookup.filter(col("user_name") == "your.email@company.com").show()
   ```

2. **Verify workspace ID**:
   - Find your workspace ID in the Databricks URL or Account Console

3. **Test service principal**:
   ```python
   # Test if AccountClient can authenticate
   workspaces = list(a.workspaces.list())
   print(f"Found {len(workspaces)} workspaces")
   ```

## Security Considerations

1. **Never commit secrets to version control**
   - Use Databricks Secrets for CLIENT_SECRET
   - Add `.ipynb` files with secrets to `.gitignore`

2. **Principle of least privilege**
   - Only grant necessary permissions to users
   - Revoke admin permissions after view creation

3. **Audit trail**
   - All actions are logged in system.access.audit
   - Monitor service principal usage

## Limitations

- Only works with tables that have a `workspace_id` column
- Requires account admin access to execute
- Views are static - they don't automatically update when new system tables are added
- Cannot filter tables in `information_schema` or `marketplace` schemas

## Maintenance

### Re-running the Script

To update views (e.g., when new system tables are added):
1. Re-run the notebook with the same parameters
2. The `CREATE OR REPLACE VIEW` statement will update existing views

### Cleanup

To remove created resources:

```sql
-- Drop all views in a schema
DROP SCHEMA {target_catalog}.{target_schema} CASCADE;

-- Drop the entire catalog
DROP CATALOG {target_catalog} CASCADE;
```

## References

- [Databricks System Tables Documentation](https://docs.databricks.com/en/admin/system-tables/index.html)
- [OAuth M2M Authentication](https://docs.databricks.com/en/dev-tools/auth/oauth-m2m.html)
- [Databricks Secrets](https://docs.databricks.com/en/security/secrets/index.html)
- [Unity Catalog Permissions](https://docs.databricks.com/en/data-governance/unity-catalog/manage-privileges/index.html)

## Support

For issues or questions:
1. Check the Databricks documentation
2. Review system.access.audit for execution logs
3. Contact your Databricks account administrator

---

**Version**: 2.0  
**Last Updated**: December 2025  
**Compatibility**: DBR 13.0+, Serverless
