To implement a **default set of RBAC roles** that will be shared across all teams/applications, with the flexibility for teams to **add additional roles** if needed, we can modify the approach to use a **map-based approach** with a **default set** for the roles. 

### **Overview of the Plan:**
1. **Default RBAC roles**: These roles will be applied to all applications by default.
2. **Custom Roles**: Any additional roles that need to be assigned to specific applications/teams can be added to the default set.
3. **Dynamic handling**: The roles and scopes are passed dynamically without hardcoding them in the `main.tf` files.

---

### **Step 1: Component Repo Updates**

We will keep the component repo as is with minimal changes, mainly focusing on how we handle dynamic RBAC role assignments.

#### **1.1 Update `variables.tf` in the Component Repo**
First, define the variables for the RBAC roles as a map, where each application/team can append to a default set.

```hcl
variable "rbac_roles_map" {
  description = "Map of RBAC roles for different applications/teams"
  type = map(list(object({
    role_definition      = string
    role_definition_type = string
    scope                = string
  })))
  default = {
    "default" = [
      {
        role_definition      = "Website Contributor"
        role_definition_type = "name"
        scope                = "/subscriptions/${data.azurerm_client_config.current.subscription_id}/resourceGroups/${local.system_rg_name}"
      },
      {
        role_definition      = "Web Plan Contributor"
        role_definition_type = "name"
        scope                = "/subscriptions/${data.azurerm_client_config.current.subscription_id}/resourceGroups/${local.system_rg_name}"
      },
      {
        role_definition      = "Auto-Scale Settings Contributor"
        role_definition_type = "name"
        scope                = "/subscriptions/${data.azurerm_client_config.current.subscription_id}/resourceGroups/${local.system_rg_name}"
      },
      {
        role_definition      = "Key Vault Administrator"
        role_definition_type = "name"
        scope                = "/subscriptions/${data.azurerm_client_config.current.subscription_id}/resourceGroups/${local.system_rg_name}/providers/Microsoft.KeyVault/vaults/${var.keyvault_name}"
      },
      {
        role_definition      = "Role Based Access Control Administrator"
        role_definition_type = "name"
        scope                = "/subscriptions/${data.azurerm_client_config.current.subscription_id}/resourceGroups/${local.system_rg_name}"
      },
      {
        role_definition      = "Log Analytics Contributor"
        role_definition_type = "name"
        scope                = "/subscriptions/${data.azurerm_client_config.current.subscription_id}/resourceGroups/${local.system_rg_name}/providers/Microsoft.OperationalInsights/workspaces/${var.log_analytics_workspace}"
      },
      {
        role_definition      = "API Management Service Contributor"
        role_definition_type = "name"
        scope                = "/subscriptions/${data.azurerm_client_config.current.subscription_id}/resourceGroups/${local.system_rg_name}/providers/Microsoft.ApiManagement/service/${var.apim_name}"
      },
      {
        role_definition      = "Storage Blob Data Contributor"
        role_definition_type = "name"
        scope                = "/subscriptions/${data.azurerm_client_config.current.subscription_id}/resourceGroups/${local.system_rg_name}/providers/Microsoft.Storage/storageAccounts/${var.storage_account_name}"
      },
      {
        role_definition      = "Storage Account Contributor"
        role_definition_type = "name"
        scope                = "/subscriptions/${data.azurerm_client_config.current.subscription_id}/resourceGroups/${local.system_rg_name}/providers/Microsoft.Storage/storageAccounts/${var.storage_account_name}"
      },
      {
        role_definition      = "Reader"
        role_definition_type = "name"
        scope                = "/subscriptions/${data.azurerm_client_config.current.subscription_id}/resourceGroups/${local.system_rg_name}"
      }
    ]
  }
}
```

This structure allows you to define **default roles** under the `"default"` key, and if any specific application team needs additional roles, they can add them to this map under their app name (e.g., `"app1"`, `"app2"`, etc.).

---

#### **1.2 Update `main.tf` in the Component Repo**
The `role_assignment` module can dynamically handle the map-based RBAC role assignments by flattening the map and passing it into the `role_assignment` module.

```hcl
module "role_assignment" {
  source = "github.com/cloud-era/terraform-azure-role_assignment?ref=init"

  res_rbac_info = flatten([
    for app, roles in var.rbac_roles_map : [
      for role in roles : {
        principal_id                 = var.customer_spn_object_id
        role_definition              = role.role_definition
        role_definition_type         = role.role_definition_type
        scope                        = role.scope
        skip_service_principal_aad_check = false
      }
    ]
  ])
}
```

This will iterate over the **`rbac_roles_map`**, which includes a default set of roles for the `"default"` application, and can include additional roles for other applications as required.

---

### **Step 2: Landing Zone Repo Updates**

The Landing Zone repo will pass the **`rbac_roles_map`** to the **Component Repo** using the updated variables.

#### **2.1 Update `main.tf` in the Landing Zone Repo**

Modify `main.tf` to include the `rbac_roles_map` and pass it to the **component repo**:

```hcl
module "landing_zone" {
  source = "github.com/cloud-era/terrafora-azure-component-landing-zone?ref=init"

  # Existing parameters
  eonid               = var.eonid
  location            = local.location
  lz_name             = var.lz_name
  short_name          = var.short_name
  short_env           = var.short_env
  env                 = var.env_name
  vnet_address_prefix = var.vnet_address_prefix

  # Resource tags
  tags = local.core_tagging

  # RBAC roles map
  rbac_roles_map = var.rbac_roles_map
}
```

Here, the **`rbac_roles_map`** is passed directly to the `landing_zone` module, which will be forwarded to the `role_assignment` module in the component repo.

#### **2.2 Update `variables.tf` in the Landing Zone Repo**

Define the `rbac_roles_map` variable in the Landing Zone `variables.tf` file:

```hcl
variable "rbac_roles_map" {
  description = "Map of RBAC roles for different applications/teams"
  type = map(list(object({
    role_definition      = string
    role_definition_type = string
    scope                = string
  })))
}
```

---

#### **2.3 Update `terraform.tfvars` in the Landing Zone Repo**

Provide values for the **default roles** and the **SPN object ID** in `terraform.tfvars`:

```hcl
customer_spn_object_id = "spn-object-id"

rbac_roles_map = {
  "default" = [
    {
      role_definition      = "Website Contributor"
      role_definition_type = "name"
      scope                = "/subscriptions/${data.azurerm_client_config.current.subscription_id}/resourceGroups/${local.system_rg_name}"
    },
    {
      role_definition      = "Web Plan Contributor"
      role_definition_type = "name"
      scope                = "/subscriptions/${data.azurerm_client_config.current.subscription_id}/resourceGroups/${local.system_rg_name}"
    },
    {
      role_definition      = "Auto-Scale Settings Contributor"
      role_definition_type = "name"
      scope                = "/subscriptions/${data.azurerm_client_config.current.subscription_id}/resourceGroups/${local.system_rg_name}"
    },
    # ... (all other roles)
  ]
}
```

If an application team needs to **add more roles**, they simply extend the map for their application. For example, for `app1`, they can do the following:

```hcl
rbac_roles_map = {
  "default" = [
    # Default roles (as shown earlier)
  ],
  "app1" = [
    {
      role_definition      = "API Management Contributor"
      role_definition_type = "name"
      scope                = "/subscriptions/${data.azurerm_client_config.current.subscription_id}/resourceGroups/${local.system_rg_name}/providers/Microsoft.ApiManagement/service/app1-apim"
    }
  ]
}
```

---

## **Summary of Changes**

### **Component Repo:**
1. **`variables.tf`**: Define `rbac_roles_map` as a `map(list(object))` to handle dynamic roles for each application.
2. **`main.tf`**: Flatten the map and pass the RBAC roles to the `role_assignment` module.
3. **`outputs.tf`**: Ensure outputs for dynamic resource groups and roles are set up.

### **Landing Zone Repo:**
1. **`main.tf`**: Pass `rbac_roles_map` to the **Landing Zone** module and then forward to the **component repo**.
2. **`variables.tf`**: Define `rbac_roles_map` variable to accept the map of roles.
3. **`terraform.tfvars`**: Specify the default roles and additional roles (if any) for each application/team.

---

This map-based approach ensures flexibility and scalability, allowing each team to customize their RBAC roles easily without modifying the core Terraform configuration.

Let me know if you need further assistance or clarification!
