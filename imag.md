Here’s the full updated Python script that fetches artifact details from JFrog Artifactory and generates a CSV report, including **unique ID** (SHA-256 hash of artifact URL) and **digest** (SHA-256 from API response).  

---

### **Full Script**
```python
import requests
import csv
import os
import hashlib

# Constants
JFROG_URL = "https://frigate.jfrog.io"
ARTIFACTORY_API = f"{JFROG_URL}/artifactory/api/storage"
ARTIFACTORY_TOKEN = "your_artifactory_token_here"

# Headers for API authentication
HEADERS = {
    "Authorization": f"Bearer {ARTIFACTORY_TOKEN}",
    "Accept": "application/json"
}

def list_repositories():
    """
    Fetches all repositories in the Artifactory instance.
    """
    url = f"{JFROG_URL}/artifactory/api/repositories"
    response = requests.get(url, headers=HEADERS)
    response.raise_for_status()
    return response.json()

def list_artifacts(repo_name):
    """
    Fetches all artifacts in a repository.
    """
    url = f"{ARTIFACTORY_API}/{repo_name}?list&deep=1"
    response = requests.get(url, headers=HEADERS)
    
    if response.status_code == 200:
        return response.json().get("files", [])
    else:
        print(f"Error fetching artifacts for {repo_name}: {response.text}")
        return []

def process_repository(repo_name, package_type):
    """
    Processes all artifacts in a repository while avoiding `.jfrog` cache folders.
    """
    try:
        artifact_list = list_artifacts(repo_name)
        repo_details = []

        for artifact in artifact_list:
            artifact_path = artifact.get("uri", "").lstrip("/")  # Remove leading slash if present

            # Skip .jfrog cache folder
            if ".jfrog" in artifact_path:
                continue

            # Extract path segments
            path_segments = artifact_path.split("/")

            # Extract version (last directory before filename)
            if len(path_segments) >= 2:
                version = path_segments[-2]
            else:
                version = ""

            # Extract package name (everything after repo_name but before the version)
            package_name = "/".join(path_segments[:-2]) if len(path_segments) > 2 else ""

            # Ensure package name is correctly formatted
            if package_name.startswith(repo_name + "/"):
                package_name = package_name[len(repo_name) + 1:]

            # Construct the full artifact URL
            artifact_url = f"{JFROG_URL}/artifactory/{repo_name}/{artifact_path}"

            # Generate unique ID using SHA-256 hash of the URL
            unique_id = hashlib.sha256(artifact_url.encode()).hexdigest()

            # Extract digest (sha256) if available
            digest = artifact.get("sha256", "")

            # Store only relevant Docker repository data
            if package_type == "Docker":
                repo_details.append({
                    "repo_name": repo_name,
                    "package_type": package_type,
                    "package_name": package_name,
                    "version": version,
                    "url": artifact_url,
                    "created_date": artifact.get("lastModified", ""),
                    "license": "",  # Leave blank if not available
                    "secarch": "",  # Leave blank if not available
                    "artifactory_instance": "frigate.jfrog.io",
                    "unique_id": unique_id,
                    "digest": digest
                })

        return repo_details

    except Exception as e:
        print(f"Skipping repository {repo_name} due to error: {e}")
        return []  # Continue with other repositories

def save_to_csv(data, filename):
    """
    Saves extracted metadata to a CSV file.
    """
    headers = ['repo_name', 'package_type', 'package_name', 'version', 'url', 'created_date', 
               'license', 'secarch', 'artifactory_instance', 'unique_id', 'digest']
    
    os.makedirs(os.path.dirname(filename), exist_ok=True)  # Ensure output directory exists

    with open(filename, mode='w', newline='', encoding='utf-8') as file:
        writer = csv.DictWriter(file, fieldnames=headers)
        writer.writeheader()
        writer.writerows(data)
    
    print(f"Data has been written to {filename}")

def main():
    """
    Main function to process repositories and generate the CSV report.
    """
    repositories = list_repositories()
    all_data = []

    for repo in repositories:
        repo_name = repo.get("key", "")
        package_type = repo.get("packageType", "")

        # Process only Docker repositories
        if package_type == "Docker":
            print(f"Processing repository: {repo_name}")
            repo_data = process_repository(repo_name, package_type)
            all_data.extend(repo_data)

    # Save data to CSV
    save_to_csv(all_data, "c:/sre/jfrog_report.csv")

if __name__ == "__main__":
    main()
```

---

### **How This Works**
1. **Fetches all repositories** in JFrog Artifactory (`list_repositories`).
2. **Filters only Docker repositories** before processing.
3. **Retrieves artifacts** from each repository (`list_artifacts`).
4. **Extracts metadata**, including:
   - `repo_name`
   - `package_type`
   - `package_name`
   - `version`
   - `url`
   - `created_date`
   - `unique_id` (SHA-256 hash of URL)
   - `digest` (SHA-256 from API response)
5. **Saves all data into a CSV file** (`c:/sre/jfrog_report.csv`).

---

### **Expected Output (CSV Example)**
| repo_name     | package_type | package_name | version | url | created_date | unique_id | digest |
|--------------|-------------|-------------|---------|----|-------------|------------|--------|
| my-docker-repo | Docker | my-app | 1.0.0 | `https://frigate.jfrog.io/artifactory/my-docker-repo/my-app/1.0.0/image.tar` | 2024-08-21T12:34:56Z | `c3fcd3d76192e4007dfb496cca67e13b1dd4f503` | `abc123...` |
| my-docker-repo | Docker | my-app | 1.1.0 | `https://frigate.jfrog.io/artifactory/my-docker-repo/my-app/1.1.0/image.tar` | 2024-08-22T14:00:21Z | `f8e3c2b7d3...` | `def456...` |

---

### **Key Enhancements**
✅ **Unique ID**: Generated using SHA-256 hash of the artifact URL.  
✅ **Digest**: Extracted from JFrog API (`artifact.get("sha256")`).  
✅ **Filters only Docker repositories** before processing.  
✅ **CSV output includes all relevant metadata** for tracking artifacts.  

Let me know if you need any modifications! 🚀
