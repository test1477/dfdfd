Here is a detailed step-by-step breakdown of the required updates for both the **Landing Zone Repo** and the **Component Repo**. This will ensure the implementation of dynamic RBAC role assignments using the **map-based approach** with the ability to compute scopes dynamically.

---

## **Landing Zone Repo Updates**

### **Step 1: Update `variables.tf`**
Add variables for `rbac_roles_map` and `customer_spn_object_id`.

```hcl
variable "customer_spn_object_id" {
  description = "Object ID of the customer SPN"
  type        = string
}

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

### **Step 2: Update `main.tf`**
Pass the `rbac_roles_map` and `customer_spn_object_id` variables into the `landing_zone` module.

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

  # Pass RBAC-related variables
  rbac_roles_map         = var.rbac_roles_map
  customer_spn_object_id = var.customer_spn_object_id
}
```

---

### **Step 3: Update `terraform.tfvars`**
Provide values for `rbac_roles_map` and `customer_spn_object_id`. Use placeholders for scopes, which will be replaced dynamically.

```hcl
customer_spn_object_id = "spn-object-id"

rbac_roles_map = {
  "default" = [
    {
      role_definition      = "Website Contributor"
      role_definition_type = "name"
      scope                = "placeholder-scope"
    },
    {
      role_definition      = "Key Vault Administrator"
      role_definition_type = "name"
      scope                = "placeholder-scope"
    },
    {
      role_definition      = "API Management Service Contributor"
      role_definition_type = "name"
      scope                = "placeholder-scope"
    },
    {
      role_definition      = "Storage Blob Data Contributor"
      role_definition_type = "name"
      scope                = "placeholder-scope"
    }
  ]
}
```

---

### **Step 4: Update `outputs.tf`**
Expose the `rbac_roles_map` and `customer_spn_object_id` outputs (if needed by downstream modules or teams).

```hcl
output "rbac_roles_map" {
  value = var.rbac_roles_map
}

output "customer_spn_object_id" {
  value = var.customer_spn_object_id
}
```

---

## **Component Repo Updates**

### **Step 1: Update `variables.tf`**
Add variables for `rbac_roles_map` and `customer_spn_object_id` in the component repo.

```hcl
variable "customer_spn_object_id" {
  description = "Object ID of the customer SPN"
  type        = string
}

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

### **Step 2: Update `main.tf`**
Dynamically construct `scope` and pass RBAC assignments to the `role_assignment` module.

```hcl
module "role_assignment" {
  source = "github.com/cloud-era/terraform-azure-role_assignment?ref=init"

  res_rbac_info = flatten([
    for app, roles in var.rbac_roles_map : [
      for role in roles : {
        principal_id                 = var.customer_spn_object_id
        role_definition              = role.role_definition
        role_definition_type         = role.role_definition_type
        scope                        = replace(
          role.scope,
          "placeholder-scope",
          "/subscriptions/${data.azurerm_client_config.current.subscription_id}/resourceGroups/${local.system_rg_name}"
        )
        skip_service_principal_aad_check = false
      }
    ]
  ])
}
```

#### Key Details:
- **Dynamic Scope**: The `replace` function dynamically replaces `placeholder-scope` with the correct subscription ID and resource group name.
- **Flexibility**: The `rbac_roles_map` can contain roles for multiple applications, with `"default"` acting as the base configuration.

---

### **Step 3: Update `outputs.tf`**
Expose any outputs needed by the `landing_zone` module.

```hcl
output "mod_out_resource_group_names" {
  value = module.resource_groups.res_out_rg_names[0]
}

output "mod_out_resource_group_ids" {
  value = module.resource_groups.res_out_rg_ids[0]
}
```

---

### **Step 4: Update `locals.tf`**
Define `local.system_rg_name` and other dynamic variables used in the `scope` construction.

```hcl
locals {
  system_rg_name    = data.azurerm_resource_group.system_rg.name
  app_rg_name       = data.azurerm_resource_group.app_rg.name
  subscription_id   = data.azurerm_client_config.current.subscription_id
}
```

---

### **Step 5: Update `terraform.tfvars`**
Provide values for any required variables in `terraform.tfvars`.

```hcl
customer_spn_object_id = "spn-object-id"

rbac_roles_map = {
  "default" = [
    {
      role_definition      = "Website Contributor"
      role_definition_type = "name"
      scope                = "placeholder-scope"
    },
    {
      role_definition      = "Key Vault Administrator"
      role_definition_type = "name"
      scope                = "placeholder-scope"
    }
  ]
}
```

---

## **Summary of Files to Update**

### **Landing Zone Repo**
1. **`variables.tf`**
   - Add `customer_spn_object_id` and `rbac_roles_map`.

2. **`main.tf`**
   - Pass `customer_spn_object_id` and `rbac_roles_map` to the `landing_zone` module.

3. **`terraform.tfvars`**
   - Provide values for `customer_spn_object_id` and `rbac_roles_map`.

4. **`outputs.tf`**
   - Expose `rbac_roles_map` and `customer_spn_object_id` if needed.

---

### **Component Repo**
1. **`variables.tf`**
   - Add `customer_spn_object_id` and `rbac_roles_map`.

2. **`main.tf`**
   - Dynamically construct `scope` and pass it to the `role_assignment` module.

3. **`outputs.tf`**
   - Expose required outputs (e.g., `resource group names` and `IDs`).

4. **`locals.tf`**
   - Define locals for subscription ID and resource group names.

5. **`terraform.tfvars`**
   - Provide values for `customer_spn_object_id` and `rbac_roles_map`.

---

This structure ensures:
- **Reusability**: Default roles (`default`) are applied across teams, and additional roles can be easily added.
- **Scalability**: New applications or roles are added without modifying the core logic.
- **Dynamic Scope**: The `scope` is computed dynamically using the subscription ID and resource group names.

Let me know if you'd like me to expand on any step or provide further details!
