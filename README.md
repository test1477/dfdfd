To incorporate **RBAC role assignment for the customer SPN** into the Landing Zone while leveraging the existing `role_assignment` module, you need to ensure that the `main.tf`, `variables.tf`, and `terraform.tfvars` files in the Landing Zone repository are correctly configured to pass the necessary values dynamically. Here's the step-by-step guide:

---

### **1. Define Variables in `variables.tf`**

Add input variables to the Landing Zone's `variables.tf` file to allow dynamic configuration of RBAC role assignments.

```hcl
variable "customer_spn_rbac_list" {
  description = "List of RBAC role assignments for the customer SPN"
  type = list(object({
    group_name       = string  # Key to map the SPN or group
    role_definition  = string  # Role to assign
    scope            = string  # Scope for the role assignment
  }))
  default = []
}

variable "customer_spn_object_id" {
  description = "The object ID of the customer SPN"
  type        = string
}
```

---

### **2. Update `main.tf` to Use the `role_assignment` Module**

Add the `role_assignment` module call in your Landing Zone's `main.tf` file to process the roles dynamically:

```hcl
module "role_assignment" {
  source = "github.com/Eaton-Vance-Corp/terraform-azure-role_assignment?ref=init"

  res_rbac_info = [
    for rbac in var.customer_spn_rbac_list : {
      principal_id                     = var.customer_spn_object_id
      role_definition_type             = "name"
      role_definition                  = rbac.role_definition
      scope                            = rbac.scope
      skip_service_principal_aad_check = false
    }
  ]
}
```

---

### **3. Pass Values Dynamically in `terraform.tfvars`**

The `terraform.tfvars` file should provide the specific RBAC role assignments for the customer SPN. Example:

```hcl
customer_spn_object_id = "spn-object-id"

customer_spn_rbac_list = [
  {
    group_name      = "group1"
    role_definition = "Website Contributor"
    scope           = "/subscriptions/<subscription-id>/resourceGroups/rg-name"
  },
  {
    group_name      = "group2"
    role_definition = "Key Vault Administrator"
    scope           = "/subscriptions/<subscription-id>/resourceGroups/rg-name/providers/Microsoft.KeyVault/vaults/keyvault-name"
  },
  {
    group_name      = "group3"
    role_definition = "API Management Service Contributor"
    scope           = "/subscriptions/<subscription-id>/resourceGroups/rg-name/providers/Microsoft.ApiManagement/service/apim-name"
  }
]
```

---

### **4. Ensure Correct Integration with Other Modules**

If you need to apply RBAC at specific modules (e.g., `akv`, `sql`, etc.), pass the `customer_spn_object_id` and the relevant role assignments (`customer_spn_rbac_list`) for those modules dynamically.

For example, for the **Key Vault (AKV)** module:
```hcl
module "akv" {
  source       = "github.com/cloud-era/terraform-azure-component-akv?ref=init"
  depends_on   = [module.landing_zone]

  system_rg_name  = module.landing_zone.mod_out_resource_group_names
  app_rg_name     = module.landing_zone.mod_out_app_resource_group_names
  location        = local.location
  env             = var.env_name
  keyvault_name   = var.keyvault_name

  spn_rbac = {
    spn_object_ids = [var.customer_spn_object_id]
  }

  privateendpoint_subnetid = module.subnets.mod_out_subnet_ids[1]
}
```

---

### **5. Adjust Dependencies if Needed**

If role assignments depend on the creation of specific resources (e.g., resource groups or subnets), ensure you add `depends_on` in the `role_assignment` module:

```hcl
module "role_assignment" {
  source = "github.com/Eaton-Vance-Corp/terraform-azure-role_assignment?ref=init"

  depends_on = [module.landing_zone, module.subnets]

  res_rbac_info = [
    for rbac in var.customer_spn_rbac_list : {
      principal_id                     = var.customer_spn_object_id
      role_definition_type             = "name"
      role_definition                  = rbac.role_definition
      scope                            = rbac.scope
      skip_service_principal_aad_check = false
    }
  ]
}
```

---

### **6. Example Full Files**

#### **`variables.tf`**
```hcl
variable "eonid" {
  description = "The EON ID for the deployment"
  type        = string
}

variable "location" {
  description = "Azure region for resources"
  type        = string
}

variable "customer_spn_object_id" {
  description = "The object ID of the customer SPN"
  type        = string
}

variable "customer_spn_rbac_list" {
  description = "List of RBAC role assignments for the customer SPN"
  type = list(object({
    group_name       = string
    role_definition  = string
    scope            = string
  }))
  default = []
}
```

#### **`main.tf`**
```hcl
module "landing_zone" {
  source = "github.com/cloud-era/terrafora-azure-component-landing-zone?ref=init"

  eonid               = var.eonid
  location            = var.location
  lz_name             = var.lz_name
  short_name          = var.short_name
  short_env           = var.short_env
  env                 = var.env_name
  vnet_address_prefix = var.vnet_address_prefix

  tags = local.core_tagging
}

module "role_assignment" {
  source = "github.com/Eaton-Vance-Corp/terraform-azure-role_assignment?ref=init"

  res_rbac_info = [
    for rbac in var.customer_spn_rbac_list : {
      principal_id                     = var.customer_spn_object_id
      role_definition_type             = "name"
      role_definition                  = rbac.role_definition
      scope                            = rbac.scope
      skip_service_principal_aad_check = false
    }
  ]
}
```

#### **`terraform.tfvars`**
```hcl
eonid = "123456"
location = "eastus"

customer_spn_object_id = "spn-object-id"

customer_spn_rbac_list = [
  {
    group_name      = "group1"
    role_definition = "Website Contributor"
    scope           = "/subscriptions/<subscription-id>/resourceGroups/rg-name"
  },
  {
    group_name      = "group2"
    role_definition = "Key Vault Administrator"
    scope           = "/subscriptions/<subscription-id>/resourceGroups/rg-name/providers/Microsoft.KeyVault/vaults/keyvault-name"
  },
  {
    group_name      = "group3"
    role_definition = "API Management Service Contributor"
    scope           = "/subscriptions/<subscription-id>/resourceGroups/rg-name/providers/Microsoft.ApiManagement/service/apim-name"
  }
]
```

---

### **7. Testing and Validation**

1. **Run Terraform Commands:**
   - Initialize Terraform: `terraform init`
   - Plan Deployment: `terraform plan -var-file=terraform.tfvars`
   - Apply Changes: `terraform apply -var-file=terraform.tfvars`

2. **Validate Role Assignments:**
   - Check the customer SPN roles in the Azure portal under **Access Control (IAM)** for the specified scopes.

---

### **Summary**

1. **Define variables** for the customer SPN and its role assignments in `variables.tf`.
2. Use the **`role_assignment` module** to assign roles dynamically in `main.tf`.
3. Provide **input values in `terraform.tfvars`** for specific roles and scopes.
4. Ensure **dependencies** are set up for resource creation before role assignment.
5. Test and validate the setup in your Azure environment.

Let me know if you need further clarification or assistance!
