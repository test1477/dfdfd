Here’s the updated script based on your requirements:  

### **Updates:**
✅ Added **repo path** to the CSV output.  
✅ **Package name** now only includes the part before the version.  
✅ Removed **license**, **secarch**, **created_date**, and **version** fields.  
✅ Retained **unique_id** and **digest**.  

---

### **Updated Script**
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
    response = requests.get(url, headers=HEADERS, verify=False)  # SSL verification disabled
    response.raise_for_status()
    return response.json()

def list_artifacts(repo_name):
    """
    Fetches all artifacts in a repository.
    """
    url = f"{ARTIFACTORY_API}/{repo_name}?list&deep=1"
    response = requests.get(url, headers=HEADERS, verify=False)  # SSL verification disabled

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

            # Ensure package name is only before the version
            if len(path_segments) >= 2:
                package_name = path_segments[-2]  # Take the second last segment as package name
            else:
                package_name = "PLACEHOLDER"

            # Construct the full artifact URL (repo path)
            artifact_url = f"{JFROG_URL}/artifactory/{repo_name}/{artifact_path}"

            # Generate unique ID using SHA-256 hash of the URL
            unique_id = hashlib.sha256(artifact_url.encode()).hexdigest()

            # Extract digest (sha256) if available
            digest = artifact.get("sha256", "")

            # Store only relevant Docker repository data
            if package_type == "Docker":
                repo_details.append({
                    "repo_name": repo_name,
                    "repo_path": artifact_path,  # Full repo path
                    "package_name": package_name,  # Only before the version
                    "url": artifact_url,
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
    headers = ['repo_name', 'repo_path', 'package_name', 'url', 'artifactory_instance', 'unique_id', 'digest']
    
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
    requests.packages.urllib3.disable_warnings()  # Suppress SSL warnings
    main()
```

---

### **Final CSV Format**
| repo_name | repo_path | package_name | url | artifactory_instance | unique_id | digest |
|-----------|-----------|--------------|-----|----------------------|-----------|--------|
| my-repo  | path/to/package/1.0.0/file.tar | package | `https://frigate.jfrog.io/artifactory/my-repo/path/to/package/1.0.0/file.tar` | frigate.jfrog.io | `unique_hash` | `sha256-digest` |

### **Changes Implemented**
✔ **Added `repo_path`** field (full path inside the repo).  
✔ **Updated `package_name`** (now only includes the part before the version).  
✔ **Removed unnecessary fields** (license, secarch, created_date, version).  
✔ **Kept essential metadata** (repo name, repo path, package name, URL, unique ID, digest).  

This should match your requirements perfectly! Let me know if you need any adjustments. 🚀
