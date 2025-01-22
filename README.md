To configure **RBAC role assignments for customer SPNs** using a base Terraform template with GitHub Actions, you can skip CI/CD-level dynamic variable injection since you'll use `.tfvars` for inputs. Here's the **step-by-step guide**:

---

### **Step 1: Define Variables for Role Assignments**
In your **`variables.tf`** file, define a variable to manage role assignments dynamically:

```hcl
variable "app_apprg_role_assignments" {
  description = "List of role assignments for customer SPNs"
  type = list(object({
    spn_object_id   = string
    role_definition = string
    scope           = string
  }))
}
```

---

### **Step 2: Implement Role Assignments in `main.tf`**
Add the following logic in your `main.tf` file to assign roles to the customer SPNs dynamically:

```hcl
resource "azurerm_role_assignment" "app_apprg_roles" {
  for_each = { for idx, role_assignment in var.app_apprg_role_assignments : idx => role_assignment }

  scope                = each.value.scope
  role_definition_name = each.value.role_definition
  principal_id         = each.value.spn_object_id
}
```

---

### **Step 3: Prepare `terraform.tfvars`**
Use the **`terraform.tfvars`** file to define the SPNs, their roles, and scopes.

```hcl
app_apprg_role_assignments = [
  {
    spn_object_id   = "spn-object-id-1"
    role_definition = "Website Contributor"
    scope           = "/subscriptions/<subscription-id>/resourceGroups/app-rg"
  },
  {
    spn_object_id   = "spn-object-id-2"
    role_definition = "Key Vault Administrator"
    scope           = "/subscriptions/<subscription-id>/resourceGroups/app-rg/providers/Microsoft.KeyVault/vaults/my-keyvault"
  },
  {
    spn_object_id   = "spn-object-id-3"
    role_definition = "API Management Service Contributor"
    scope           = "/subscriptions/<subscription-id>/resourceGroups/app-rg/providers/Microsoft.ApiManagement/service/my-apim"
  }
]
```

---

### **Step 4: Configure GitHub Actions Workflow**
Define a GitHub Actions workflow to automate the application of Terraform code.

#### **Sample GitHub Actions Workflow (`.github/workflows/terraform.yml`):**

```yaml
name: Terraform Deployment

on:
  push:
    branches:
      - main

jobs:
  terraform:
    runs-on: ubuntu-latest

    permissions:
      id-token: write
      contents: read

    steps:
    - name: Checkout Repository
      uses: actions/checkout@v4

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v2
      with:
        terraform_wrapper: false

    - name: Initialize Terraform
      run: terraform init

    - name: Validate Terraform
      run: terraform validate

    - name: Plan Terraform Changes
      run: terraform plan -var-file="terraform.tfvars"

    - name: Apply Terraform Changes
      run: terraform apply -auto-approve -var-file="terraform.tfvars"
```

---

### **Step 5: Assign Required Permissions to Master SPN**
The **Master Landing Zone SPN** must have sufficient permissions to assign roles to customer SPNs.

#### **Permissions Required**:
1. **`User Access Administrator` Role**:
   - Required for managing role assignments (RBAC) in Azure.
   - Assign this role at the **scope** where the role assignments will be created (e.g., subscription or resource group level).

2. **`Owner` Role** (optional):
   - If you prefer not to assign `User Access Administrator`, the `Owner` role can also perform role assignments but includes additional permissions that might not be necessary.

---

### **Step 6: Test with a Sample Use Case**
1. Update the **`terraform.tfvars`** file with a sample role assignment for a customer SPN.
2. Push changes to the GitHub repository to trigger the GitHub Actions workflow.
3. Verify that:
   - The role assignments are created successfully.
   - The SPNs have access to the specified scopes with the assigned roles.

---

### **Folder Structure**

Ensure the repository structure is clear and modular for reusability:

```
/.github/workflows/terraform.yml         # GitHub Actions workflow
/main.tf                                 # Main Terraform configuration
/variables.tf                            # Input variables
/terraform.tfvars                        # Role assignments input
/modules/role_assignment/                # Optional module for role assignment
```

---

### **Summary of Steps**

1. **Define variables** for role assignments in `variables.tf`.
2. **Implement logic** for dynamic role assignments in `main.tf`.
3. Use **`terraform.tfvars`** to define the customer SPNs, roles, and scopes.
4. Create a **GitHub Actions workflow** to automate Terraform deployment.
5. Assign **`User Access Administrator`** or equivalent permissions to the master SPN.
6. Test the workflow with a sample `terraform.tfvars`.

---

Let me know if you need help configuring specific scopes or debugging your GitHub Actions workflow!
