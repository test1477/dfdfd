To modify the script to fetch Docker images from **all repositories within an organization** in JFrog Artifactory (instead of a single repository) and update the column names as per your requirements (`Resource_Name` instead of `Docker Image Name` and `Registry` instead of `Repo Path`), here's the updated script:

---

### **Updated Python Script**

```python
import requests
import csv

# JFrog Artifactory credentials and URL
ARTIFACTORY_URL = "https://<your-artifactory-domain>/artifactory"
API_KEY = "<your-api-key>"

# Headers for API requests
headers = {
    "X-JFrog-Art-Api": API_KEY,
    "Content-Type": "application/json"
}

# Function to get all repositories in the organization
def get_all_repositories():
    url = f"{ARTIFACTORY_URL}/api/repositories"
    response = requests.get(url, headers=headers)
    if response.status_code != 200:
        raise Exception(f"Failed to fetch repositories: {response.text}")
    return [repo["key"] for repo in response.json() if repo["type"] == "DOCKER"]

# Function to get Docker image details from a repository
def get_docker_images(repo_name):
    url = f"{ARTIFACTORY_URL}/api/docker/{repo_name}/v2/_catalog"
    response = requests.get(url, headers=headers)
    if response.status_code != 200:
        raise Exception(f"Failed to fetch Docker images from {repo_name}: {response.text}")
    return response.json().get("repositories", [])

# Function to get image tags and details
def get_image_tags(repo_name, image_name):
    url = f"{ARTIFACTORY_URL}/api/docker/{repo_name}/v2/{image_name}/tags/list"
    response = requests.get(url, headers=headers)
    if response.status_code != 200:
        raise Exception(f"Failed to fetch tags for {image_name} in {repo_name}: {response.text}")
    return response.json().get("tags", [])

# Function to get image manifest and digest
def get_image_manifest(repo_name, image_name, tag):
    url = f"{ARTIFACTORY_URL}/api/docker/{repo_name}/v2/{image_name}/manifests/{tag}"
    response = requests.get(url, headers=headers)
    if response.status_code != 200:
        raise Exception(f"Failed to fetch manifest for {image_name}:{tag} in {repo_name}: {response.text}")
    return response.headers.get("Docker-Content-Digest")

# Function to generate CSV report
def generate_csv_report(output_file="docker_images_report.csv"):
    repositories = get_all_repositories()
    with open(output_file, mode="w", newline="") as csv_file:
        writer = csv.writer(csv_file)
        writer.writerow(["Resource_Name", "CSP Placeholder", "Resource Type", "Unique ID", "Digest", "Registry"])

        for repo_name in repositories:
            try:
                images = get_docker_images(repo_name)
                for image_name in images:
                    tags = get_image_tags(repo_name, image_name)
                    for tag in tags:
                        digest = get_image_manifest(repo_name, image_name, tag)
                        registry_path = f"{ARTIFACTORY_URL}/{repo_name}/{image_name}:{tag}"
                        writer.writerow([image_name, "CSP_PLACEHOLDER", "Container Image", tag, digest, registry_path])
            except Exception as e:
                print(f"Error processing repository {repo_name}: {e}")

    print(f"CSV report generated: {output_file}")

# Main execution
if __name__ == "__main__":
    generate_csv_report()
```

---

### **Key Changes**
1. **Fetch All Repositories**:
   - The `get_all_repositories` function retrieves all Docker-type repositories in the organization using the `/api/repositories` endpoint.

2. **Process All Repositories**:
   - The script iterates over all Docker repositories and fetches Docker images, tags, and digests for each repository.

3. **Updated Column Names**:
   - Changed `Docker Image Name` to `Resource_Name`.
   - Changed `Repo Path` to `Registry`.

4. **Registry Path**:
   - The `Registry` column now contains the full path to the Docker image in the format: `https://<artifactory-domain>/artifactory/<repo-name>/<image-name>:<tag>`.

---

### **Example CSV Output**
The generated CSV file will look like this:

| Resource_Name | CSP Placeholder | Resource Type   | Unique ID | Digest                                      | Registry                                              |
|---------------|-----------------|-----------------|-----------|---------------------------------------------|-------------------------------------------------------|
| my-image-1    | CSP_PLACEHOLDER | Container Image | latest    | sha256:abc123...                            | https://artifactory.example.com/artifactory/repo1/my-image-1:latest |
| my-image-2    | CSP_PLACEHOLDER | Container Image | v1.0      | sha256:def456...                            | https://artifactory.example.com/artifactory/repo2/my-image-2:v1.0   |

---

### **Prerequisites**
1. **Python Libraries**:
   - Install the `requests` library if not already installed:
     ```bash
     pip install requests
     ```

2. **JFrog Artifactory API Key**:
   - Generate an API key from your JFrog Artifactory account.

3. **Artifactory Domain**:
   - Replace `<your-artifactory-domain>` with your actual Artifactory domain.

---

### **Running the Script**
1. Save the script to a file, e.g., `generate_docker_report_org.py`.
2. Run the script:
   ```bash
   python generate_docker_report_org.py
   ```
3. The CSV report will be saved as `docker_images_report.csv` in the current directory.

---

### **Customization**
- **CSP Placeholder**: Replace `"CSP_PLACEHOLDER"` with the actual CSP (Cloud Service Provider) placeholder if needed.
- **Resource Type**: Update the resource type if you are working with other types of artifacts.
- **Error Handling**: The script skips repositories that encounter errors and continues processing the next repository.

---

Let me know if you need further assistance!
