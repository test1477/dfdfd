You're right! If the **customer SPN** is only assigned roles to the **App Resource Group (App RG)**, and there is **no outputs.tf in the Landing Zone repo**, we can simplify the updates. Below, I will correctly detail only the **necessary updates** for the Landing Zone and Component repositories while clarifying the correct approach:

---

## **Landing Zone Repo Updates**

Since the Landing Zone repo doesn't have an `outputs.tf` file, and we are only assigning RBAC roles to the **App RG**, we don't need to propagate unnecessary outputs. Instead, we will dynamically construct the scope for the App RG directly in the Component Repo. 

### **Step 1: Update `variables.tf`**

Define the following variables in the **Landing Zone Repo**:

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

Pass the **SPN object ID** and **App RG RBAC roles** to the Landing Zone module:

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

  # RBAC-related variables
  customer_spn_object_id = var.customer_spn_object_id
  app_rbac_roles         = var.app_rbac_roles
}
```

---

### **Step 3: Update `terraform.tfvars`**

Define the SPN and default RBAC roles assigned to the App RG in `terraform.tfvars`.

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

This ensures that all required roles are passed as a list to the Landing Zone module.

---

## **Component Repo Updates**

In the Component Repo, we will use the **App RG name and ID** to dynamically construct the `scope` for the customer SPN role assignments.

---

### **Step 1: Update `variables.tf`**

Add the following variables to accept the **customer SPN object ID** and **App RG RBAC roles**:

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

Use the `app_rbac_roles` and dynamically construct the `scope` for the App Resource Group using its name and ID.

```hcl
module "role_assignment" {
  source = "github.com/cloud-era/terraform-azure-role_assignment?ref=init"

  res_rbac_info = [
    for role in var.app_rbac_roles : {
      principal_id                 = var.customer_spn_object_id
      role_definition              = role.role_definition
      role_definition_type         = role.role_definition_type
      scope                        = "/subscriptions/${data.azurerm_client_config.current.subscription_id}/resourceGroups/${local.app_rg_name}"
      skip_service_principal_aad_check = false
    }
  ]
}
```

#### Explanation:
- **`principal_id`**: The customer SPN object ID.
- **`role_definition`**: The role to be assigned.
- **`scope`**: Dynamically constructed using the **App RG name** (`local.app_rg_name`) and **subscription ID**.
- **`skip_service_principal_aad_check`**: Set to `false` to ensure the SPN is verified.

---

### **Step 3: Update `locals.tf`**

Define `local.app_rg_name` to fetch the App RG name dynamically.

```hcl
locals {
  app_rg_name = "eon${var.eonid}-${var.location}-${var.lz_name}-app-rg"
}
```

This ensures that the `scope` is dynamically generated for the App RG without requiring hardcoding in the variables or module inputs.

---

### **Step 4: Ensure `data.azurerm_client_config` is Available**

Add a `data` block to fetch the subscription ID dynamically.

```hcl
data "azurerm_client_config" "current" {}
```

This provides the subscription ID needed for constructing the `scope`.

---

## **Summary of Files to Update**

### **Landing Zone Repo**
1. **`variables.tf`**
   - Add `customer_spn_object_id` and `app_rbac_roles` variables.

2. **`main.tf`**
   - Pass the `customer_spn_object_id` and `app_rbac_roles` variables to the **Landing Zone module**.

3. **`terraform.tfvars`**
   - Define the SPN object ID and default RBAC roles for the App RG.

---

### **Component Repo**
1. **`variables.tf`**
   - Add `customer_spn_object_id` and `app_rbac_roles` variables.

2. **`main.tf`**
   - Use `app_rbac_roles` to dynamically assign roles to the App RG.

3. **`locals.tf`**
   - Define `local.app_rg_name` to construct the App RG name dynamically.

4. **`data.azurerm_client_config`**
   - Add a data block to fetch the subscription ID.

---

This approach:
- Dynamically constructs the **scope** for the App RG using its name and subscription ID.
- Assigns all default RBAC roles to the **customer SPN** for the App RG.
- Allows flexibility to add additional roles later if needed.

Let me know if further clarification is required!
