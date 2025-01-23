To configure **RBAC role assignments for the customer SPN** dynamically from the **Landing Zone's `main.tf`** while leveraging the existing `role_assignment` module in the **component repo**, follow these steps:

---

## **Step 1: Update the Component Repository**
You already have the `role_assignment` module in the component repo. Ensure that it supports dynamically passing the RBAC roles and SPN details.

### **Component Repo (`main.tf`) Updates**

Verify the `role_assignment` module invocation is already dynamic:
```hcl
module "role_assignment" {
  source = "github.com/cloud-era/terraform-azure-role_assignment?ref=init"

  res_rbac_info = [
    for key, value in var.rbac_list : {
      group_name                   = module.constants.groups[value.group_name]
      principal_id                 = value.principal_id
      role_definition              = value.role_definition
      role_definition_type         = value.role_definition_type
      scope                        = value.scope
      skip_service_principal_aad_check = false
    }
  ]
}
```

### **Component Repo (`variables.tf`)**

Ensure the following variables are defined for dynamic RBAC handling:

```hcl
variable "rbac_list" {
  description = "List of RBAC role assignments for the SPN"
  type = list(object({
    principal_id         = string
    group_name           = string
    role_definition      = string
    role_definition_type = string
    scope                = string
  }))
  default = []
}
```

If these already exist, no further changes are needed in the component repo.

---

## **Step 2: Update the Landing Zone Repository**

You need to pass RBAC role assignments from the **Landing Zone's `main.tf`** to the **component's `role_assignment` module**.

### **Landing Zone (`main.tf`)**

Modify the `landing_zone` module invocation in the Landing Zone to include the SPN and RBAC list variables.

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

  # RBAC parameters
  customer_spn_object_id = var.customer_spn_object_id
  customer_rbac_list     = var.customer_rbac_list
}
```

---

### **Step 3: Add Variables for RBAC in the Landing Zone**

In the **Landing Zone's `variables.tf`**, define the necessary variables to handle the SPN and RBAC details dynamically:

```hcl
variable "customer_spn_object_id" {
  description = "The object ID of the customer SPN"
  type        = string
}

variable "customer_rbac_list" {
  description = "List of RBAC role assignments for the customer SPN"
  type = list(object({
    role_definition = string
    scope           = string
  }))
  default = []
}
```

---

### **Step 4: Pass RBAC Details in `terraform.tfvars`**

In the **Landing Zone's `terraform.tfvars`**, provide the customer SPN and its RBAC roles. Example:

```hcl
customer_spn_object_id = "spn-object-id"

customer_rbac_list = [
  {
    role_definition = "Website Contributor"
    scope           = "/subscriptions/<subscription-id>/resourceGroups/<resource-group-name>"
  },
  {
    role_definition = "Key Vault Administrator"
    scope           = "/subscriptions/<subscription-id>/resourceGroups/<resource-group-name>/providers/Microsoft.KeyVault/vaults/<keyvault-name>"
  },
  {
    role_definition = "API Management Service Contributor"
    scope           = "/subscriptions/<subscription-id>/resourceGroups/<resource-group-name>/providers/Microsoft.ApiManagement/service/<apim-name>"
  }
]
```

---

## **Step 5: Modify the Component Repo to Accept Inputs**

In the **Component Repo**, ensure the `role_assignment` module accepts the inputs from the Landing Zone:

### **Modify `variables.tf`**
Ensure the `rbac_list` variable accepts the RBAC roles and the SPN object ID passed from the Landing Zone:

```hcl
variable "customer_spn_object_id" {
  description = "The object ID of the customer SPN"
  type        = string
}

variable "customer_rbac_list" {
  description = "List of RBAC role assignments for the customer SPN"
  type = list(object({
    role_definition = string
    scope           = string
  }))
}
```

### **Update `main.tf` in Component Repo**
Update the `role_assignment` module to dynamically generate the `res_rbac_info` using the SPN and RBAC list:

```hcl
module "role_assignment" {
  source = "github.com/cloud-era/terraform-azure-role_assignment?ref=init"

  res_rbac_info = [
    for rbac in var.customer_rbac_list : {
      principal_id                 = var.customer_spn_object_id
      role_definition              = rbac.role_definition
      role_definition_type         = "name" # Assuming roles are defined by name
      scope                        = rbac.scope
      skip_service_principal_aad_check = false
    }
  ]
}
```

---

## **Step 6: Validate Dependencies**

Since the `role_assignment` module depends on the **resource group creation**, ensure the proper dependency chain is established. For example:

```hcl
module "role_assignment" {
  source = "github.com/cloud-era/terraform-azure-role_assignment?ref=init"

  depends_on = [module.resource_groups]

  res_rbac_info = [
    for rbac in var.customer_rbac_list : {
      principal_id                 = var.customer_spn_object_id
      role_definition              = rbac.role_definition
      role_definition_type         = "name"
      scope                        = rbac.scope
      skip_service_principal_aad_check = false
    }
  ]
}
```

---

## **Step 7: Test and Validate**

1. **Initialize Terraform:**
   ```bash
   terraform init
   ```

2. **Plan the Changes:**
   ```bash
   terraform plan -var-file="terraform.tfvars"
   ```

3. **Apply the Changes:**
   ```bash
   terraform apply -var-file="terraform.tfvars"
   ```

4. **Validate the Assignments:**
   - Verify that the roles are assigned to the SPN in Azure Portal under **Access Control (IAM)** for the specified scopes.

---

### **Summary of Updates**

1. **Landing Zone Repo:**
   - Update `main.tf` to pass `customer_spn_object_id` and `customer_rbac_list`.
   - Add corresponding variables in `variables.tf`.
   - Define the RBAC assignments in `terraform.tfvars`.

2. **Component Repo:**
   - Ensure the `role_assignment` module dynamically generates `res_rbac_info`.
   - Add variables for `customer_spn_object_id` and `customer_rbac_list`.

3. **Validate Dependencies:**
   - Ensure `role_assignment` depends on resource group creation to avoid timing issues.

---

Let me know if you need further clarification or help with testing the setup!
