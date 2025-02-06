Certainly. Here's the full script with SSL verification disabled using `verify=False`. Please note that this approach is not recommended for production environments as it compromises security. Use it only for testing purposes or in controlled, secure environments.

```python
import requests
import csv
import base64
import urllib3

# Disable SSL warnings
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

# Artifactory configuration
artifactory_url = "https://your-artifactory-instance.jfrog.io/artifactory"
api_key = "your-api-key"

# Authentication
auth = base64.b64encode(f"username:{api_key}".encode()).decode()
headers = {
    "Authorization": f"Basic {auth}",
    "Content-Type": "application/json"
}

# Get list of Docker repositories
repos_url = f"{artifactory_url}/api/repositories?type=docker"
repos_response = requests.get(repos_url, headers=headers, verify=False)
repos = [repo["key"] for repo in repos_response.json()]

# Prepare CSV file
csv_file = "all_docker_images_report.csv"
csv_headers = ["Docker Image Name", "CSP Placeholder", "Resource Type", "Unique ID", "Digest", "Repo Path"]

with open(csv_file, "w", newline="") as f:
    writer = csv.DictWriter(f, fieldnames=csv_headers)
    writer.writeheader()

    for repo_name in repos:
        # Get list of Docker images for each repository
        catalog_url = f"{artifactory_url}/api/docker/{repo_name}/v2/_catalog"
        response = requests.get(catalog_url, headers=headers, verify=False)
        images = response.json().get("repositories", [])

        for image in images:
            # Get tags for each image
            tags_url = f"{artifactory_url}/api/docker/{repo_name}/v2/{image}/tags/list"
            tags_response = requests.get(tags_url, headers=headers, verify=False)
            tags = tags_response.json().get("tags", [])

            for tag in tags:
                # Get manifest for each tag
                manifest_url = f"{artifactory_url}/api/docker/{repo_name}/v2/{image}/manifests/{tag}"
                manifest_response = requests.get(manifest_url, headers=headers, verify=False)
                digest = manifest_response.headers.get("Docker-Content-Digest", "")

                # Write to CSV
                writer.writerow({
                    "Docker Image Name": f"{image}:{tag}",
                    "CSP Placeholder": "JFrog Artifactory",
                    "Resource Type": "Docker Image",
                    "Unique ID": f"{repo_name}/{image}:{tag}",
                    "Digest": digest,
                    "Repo Path": f"{repo_name}/{image}"
                })

print(f"All Docker images report exported to {csv_file}")
```

Key changes in this version:

1. Added `import urllib3` at the beginning of the script.
2. Disabled SSL warnings with `urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)`.
3. Added `verify=False` to all `requests.get()` calls.

To use this script:

1. Replace `your-artifactory-instance.jfrog.io` with your actual Artifactory URL.
2. Replace `your-api-key` with your Artifactory API key.
3. Install the required libraries if you haven't already:
   ```
   pip install requests urllib3
   ```

Run the script, and it will generate a CSV file named `all_docker_images_report.csv` containing information about all Docker images across all repositories in your Artifactory instance, bypassing SSL verification.

Remember, using `verify=False` is not secure and should only be used for testing or in controlled environments. For production use, it's crucial to properly configure SSL certificates or use one of the other methods mentioned earlier to handle SSL verification correctly.

---
Answer from Perplexity: pplx.ai/share
