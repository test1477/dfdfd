If you can see Docker images under the **package type `docker`** in the JFrog Artifactory UI, but the script is not fetching any data, the issue might be with how the script is querying the Artifactory API. Since the `curl` commands (1 and 2) are working and returning data, we can use the same logic in the script to fix it.

Let’s update the script to ensure it correctly fetches Docker images and their metadata.

---

### **Updated Python Script**

Here’s the updated script with fixes and improvements:

```python
import requests
import csv
import os

# ==================================================
# Configuration Section
# ==================================================

# JFrog Artifactory credentials and URL
ARTIFACTORY_URL = "https://<your-artifactory-domain>/artifactory"  # Replace with your Artifactory URL
API_KEY = "<your-api-key>"  # Replace with your Artifactory API key
CA_CERT_PATH = "/path/to/artifactory-ca.crt"  # Path to your CA certificate (if using self-signed SSL)

# Headers for API requests
headers = {
    "X-JFrog-Art-Api": API_KEY,  # API key for authentication
    "Content-Type": "application/json"  # Specify JSON content type
}

# Disable SSL verification if CA certificate is not available
VERIFY_SSL = True
if not os.path.exists(CA_CERT_PATH):
    print("CA certificate not found. Disabling SSL verification.")
    VERIFY_SSL = False

# ==================================================
# Function Definitions
# ==================================================

def get_all_repositories():
    """
    Fetch all Docker repositories in the organization.
    Returns a list of repository names.
    """
    url = f"{ARTIFACTORY_URL}/api/repositories"
    print(f"Fetching repositories from {url}...")
    response = requests.get(url, headers=headers, verify=VERIFY_SSL)
    if response.status_code != 200:
        raise Exception(f"Failed to fetch repositories: {response.text}")
    repositories = [repo["key"] for repo in response.json() if repo["type"] == "DOCKER"]
    print(f"Found {len(repositories)} Docker repositories: {repositories}")
    return repositories

def get_docker_images(repo_name):
    """
    Fetch all Docker images in a specific repository.
    Returns a list of image names.
    """
    url = f"{ARTIFACTORY_URL}/api/docker/{repo_name}/v2/_catalog"
    print(f"Fetching Docker images from {url}...")
    response = requests.get(url, headers=headers, verify=VERIFY_SSL)
    if response.status_code != 200:
        raise Exception(f"Failed to fetch Docker images from {repo_name}: {response.text}")
    images = response.json().get("repositories", [])
    print(f"Found {len(images)} images in repository {repo_name}: {images}")
    return images

def get_image_tags(repo_name, image_name):
    """
    Fetch all tags for a specific Docker image in a repository.
    Returns a list of tags.
    """
    url = f"{ARTIFACTORY_URL}/api/docker/{repo_name}/v2/{image_name}/tags/list"
    print(f"Fetching tags for image {image_name} in repository {repo_name}...")
    response = requests.get(url, headers=headers, verify=VERIFY_SSL)
    if response.status_code != 200:
        raise Exception(f"Failed to fetch tags for {image_name} in {repo_name}: {response.text}")
    tags = response.json().get("tags", [])
    print(f"Found {len(tags)} tags for image {image_name}: {tags}")
    return tags

def get_image_manifest(repo_name, image_name, tag):
    """
    Fetch the manifest for a specific Docker image tag.
    Returns the image digest.
    """
    url = f"{ARTIFACTORY_URL}/api/docker/{repo_name}/v2/{image_name}/manifests/{tag}"
    print(f"Fetching manifest for image {image_name}:{tag} in repository {repo_name}...")
    response = requests.get(url, headers=headers, verify=VERIFY_SSL)
    if response.status_code != 200:
        raise Exception(f"Failed to fetch manifest for {image_name}:{tag} in {repo_name}: {response.text}")
    digest = response.headers.get("Docker-Content-Digest")
    print(f"Digest for image {image_name}:{tag}: {digest}")
    return digest

def generate_csv_report(output_file="docker_images_report.csv"):
    """
    Generate a CSV report of all Docker images in the organization.
    The report includes the following columns:
    - Resource_Name: Name of the Docker image
    - CSP Placeholder: Placeholder for CSP information
    - Resource Type: Type of resource (e.g., Container Image)
    - Unique ID: Tag of the Docker image
    - Digest: Digest of the Docker image
    - Registry: Full path to the Docker image in Artifactory
    """
    print("Starting CSV report generation...")
    repositories = get_all_repositories()
    if not repositories:
        print("No Docker repositories found. Exiting.")
        return

    with open(output_file, mode="w", newline="") as csv_file:
        writer = csv.writer(csv_file)
        # Write the header row
        writer.writerow(["Resource_Name", "CSP Placeholder", "Resource Type", "Unique ID", "Digest", "Registry"])

        # Iterate over all repositories
        for repo_name in repositories:
            try:
                print(f"Processing repository: {repo_name}")
                images = get_docker_images(repo_name)
                if not images:
                    print(f"No images found in repository {repo_name}. Skipping.")
                    continue

                # Iterate over all images in the repository
                for image_name in images:
                    tags = get_image_tags(repo_name, image_name)
                    if not tags:
                        print(f"No tags found for image {image_name}. Skipping.")
                        continue

                    # Iterate over all tags for the image
                    for tag in tags:
                        digest = get_image_manifest(repo_name, image_name, tag)
                        registry_path = f"{ARTIFACTORY_URL}/{repo_name}/{image_name}:{tag}"
                        # Write the row to the CSV file
                        writer.writerow([image_name, "CSP_PLACEHOLDER", "Container Image", tag, digest, registry_path])
            except Exception as e:
                print(f"Error processing repository {repo_name}: {e}")

    print(f"CSV report generated: {output_file}")

# ==================================================
# Main Execution
# ==================================================

if __name__ == "__main__":
    print("Starting script...")
    generate_csv_report()
    print("Script completed.")
```

---

### **Key Fixes**
1. **Repository Filtering**:
   - The script now correctly filters repositories by type `DOCKER` using the `/api/repositories` endpoint.

2. **Debugging Statements**:
   - Added `print` statements to log the progress of the script and help identify issues (e.g., no repositories, no images, etc.).

3. **Error Handling**:
   - The script skips repositories or images that encounter errors and continues processing the next repository.

4. **CSV Report**:
   - The script generates a CSV report with the following columns:
     - `Resource_Name`: Name of the Docker image.
     - `CSP Placeholder`: Placeholder for CSP information.
     - `Resource Type`: Type of resource (e.g., Container Image).
     - `Unique ID`: Tag of the Docker image.
     - `Digest`: Digest of the Docker image.
     - `Registry`: Full path to the Docker image in Artifactory.

---

### **Running the Script**
1. Save the script to a file, e.g., `generate_docker_report_org.py`.
2. Replace the placeholders (`<your-artifactory-domain>`, `<your-api-key>`, etc.) with your actual values.
3. Run the script:
   ```bash
   python generate_docker_report_org.py
   ```
4. Check the console output for debugging information.
5. The CSV report will be saved as `docker_images_report.csv` in the current directory.

---

### **Example Output**
#### Console Output:
```
Starting script...
Fetching repositories from https://artifactory.example.com/artifactory/api/repositories...
Found 2 Docker repositories: ['docker-repo-1', 'docker-repo-2']
Starting CSV report generation...
Processing repository: docker-repo-1
Fetching Docker images from https://artifactory.example.com/artifactory/api/docker/docker-repo-1/v2/_catalog...
Found 3 images in repository docker-repo-1: ['image1', 'image2', 'image3']
Fetching tags for image image1 in repository docker-repo-1...
Found 2 tags for image image1: ['latest', 'v1.0']
Fetching manifest for image image1:latest in repository docker-repo-1...
Digest for image image1:latest: sha256:abc123...
...
CSV report generated: docker_images_report.csv
Script completed.
```

#### CSV Output:
| Resource_Name | CSP Placeholder | Resource Type   | Unique ID | Digest                                      | Registry                                              |
|---------------|-----------------|-----------------|-----------|---------------------------------------------|-------------------------------------------------------|
| image1        | CSP_PLACEHOLDER | Container Image | latest    | sha256:abc123...                            | https://artifactory.example.com/artifactory/docker-repo-1/image1:latest |
| image1        | CSP_PLACEHOLDER | Container Image | v1.0      | sha256:def456...                            | https://artifactory.example.com/artifactory/docker-repo-1/image1:v1.0   |

---

### **Troubleshooting**
- If the script still doesn’t fetch data:
  - Verify that the API key has the necessary permissions.
  - Check the Artifactory logs for any errors.
  - Use the `curl` commands to manually verify the API responses.

Let me know if you need further assistance!
