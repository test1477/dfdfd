Based on the files you shared, your **component repo** is already well-structured with data sources, module outputs, and `locals` for managing resource group details and other configuration. Here's how we can integrate the **RBAC role assignment for the customer SPN** step by step while leveraging the existing outputs and avoiding redundancy:

---

## **Step-by-Step Integration**

### **Step 1: Update `variables.tf`**
Add new variables for the customer SPN object ID and the RBAC roles to be assigned:

```hcl
variable "customer_spn_object_id" {
  description = "Object ID of the customer SPN"
  type        = string
}

variable "app_rbac_roles" {
  description = "List of RBAC roles for the customer SPN in the App RG"
  type = list(object({
    role_definition      = string
    role_definition_type = string
  }))
}
```

---

### **Step 2: Update `main.tf`**
Leverage the existing App RG details (`local.app_rg_name`) and dynamically assign the roles to the customer SPN.

#### Add/Modify the `role_assignment` Module
In `main.tf`, update the `role_assignment` module to assign roles dynamically based on the inputs:

```hcl
module "role_assignment" {
  source = "github.com/cloud-era/terraform-azure-role_assignment?ref=init"

  res_rbac_info = [
    for role in var.app_rbac_roles : {
      principal_id                 = var.customer_spn_object_id
      role_definition              = role.role_definition
      role_definition_type         = role.role_definition_type
      scope                        = "/subscriptions/${data.azurerm_subscription.current.subscription_id}/resourceGroups/${local.app_rg_name}"
      skip_service_principal_aad_check = false
    }
  ]
}
```

#### Key Points:
- **Dynamic Scope**: The `scope` uses the `local.app_rg_name` combined with the subscription ID from `data.azurerm_subscription.current`.
- **Flexible Roles**: The roles are passed dynamically via `var.app_rbac_roles`.
- **SPN**: The `customer_spn_object_id` is used as the principal ID.

---

### **Step 3: Update `terraform.tfvars`**
Provide values for the customer SPN and the RBAC roles in `terraform.tfvars`.

#### Example:
```hcl
customer_spn_object_id = "spn-object-id"

app_rbac_roles = [
  {
    role_definition      = "Website Contributor"
    role_definition_type = "name"
  },
  {
    role_definition      = "Key Vault Administrator"
    role_definition_type = "name"
  },
  {
    role_definition      = "API Management Service Contributor"
    role_definition_type = "name"
  },
  {
    role_definition      = "Storage Blob Data Contributor"
    role_definition_type = "name"
  }
]
```

---

### **Step 4: Verify `locals.tf`**
Ensure `locals.tf` already contains the `local.app_rg_name`, which it does in your setup:

```hcl
app_rg_name = data.azurerm_resource_group.app_rg.name
```

If this is correctly defined, the `role_assignment` module will dynamically pick up the App RG name.

---

### **Step 5: No Change Required in `output.tf`**
Your current `output.tf` file already includes outputs for App RG names and IDs:

```hcl
output "mod_out_app_resource_group_names" {
  value = module.app_resource_groups.res_out_rg_names[0]
}

output "mod_out_app_resource_group_ids" {
  value = module.app_resource_groups.res_out_rg_ids[0]
}
```

These outputs can be used in downstream modules or pipelines if needed, but for this specific implementation, they are not directly required.

---

### **Step 6: Landing Zone Repo Integration**
To ensure the customer SPN and RBAC roles are passed to the component repo, you need to make the following changes in the **Landing Zone repo**:

#### **Update `variables.tf`**
Add variables to accept the customer SPN and RBAC roles:

```hcl
variable "customer_spn_object_id" {
  description = "Object ID of the customer SPN"
  type        = string
}

variable "app_rbac_roles" {
  description = "List of RBAC roles for the customer SPN in the App RG"
  type = list(object({
    role_definition      = string
    role_definition_type = string
  }))
}
```

---

#### **Update `main.tf`**
Pass the variables to the component module:

```hcl
module "component" {
  source = "github.com/cloud-era/terraform-azure-component-repo?ref=init"

  # Existing parameters
  eonid               = var.eonid
  location            = var.location
  lz_name             = var.lz_name
  tags                = var.tags

  # New RBAC-related parameters
  customer_spn_object_id = var.customer_spn_object_id
  app_rbac_roles         = var.app_rbac_roles
}
```

---

#### **Update `terraform.tfvars`**
Provide values for the SPN and roles:

```hcl
customer_spn_object_id = "spn-object-id"

app_rbac_roles = [
  {
    role_definition      = "Website Contributor"
    role_definition_type = "name"
  },
  {
    role_definition      = "Key Vault Administrator"
    role_definition_type = "name"
  },
  {
    role_definition      = "API Management Service Contributor"
    role_definition_type = "name"
  },
  {
    role_definition      = "Storage Blob Data Contributor"
    role_definition_type = "name"
  }
]
```

---

### **Step 7: Test and Validate**
1. Run `terraform plan` to ensure that the dynamic `scope` and RBAC role assignments are applied correctly.
2. Deploy the configuration using `terraform apply`.
3. Verify in Azure Portal:
   - The **App Resource Group** should have the specified roles assigned to the **customer SPN**.

---

## **Summary of Changes**

### **Landing Zone Repo**
1. **`variables.tf`**: Add `customer_spn_object_id` and `app_rbac_roles`.
2. **`main.tf`**: Pass these variables to the component module.
3. **`terraform.tfvars`**: Provide values for the SPN and roles.

---

### **Component Repo**
1. **`variables.tf`**: Add `customer_spn_object_id` and `app_rbac_roles`.
2. **`main.tf`**: Update `role_assignment` to dynamically assign roles using `local.app_rg_name`.
3. **`locals.tf`**: Ensure `app_rg_name` is derived from `data.azurerm_resource_group.app_rg.name`.

---

This approach ensures flexibility, avoids redundancy, and dynamically computes the RBAC scope. Let me know if you'd like more details or adjustments!
