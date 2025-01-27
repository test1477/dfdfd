Certainly! Here's the full updated script incorporating the changes to improve package name and version extraction for different package types:

```python
import requests
import csv
import warnings
from urllib.parse import urljoin
from datetime import datetime

# Suppress SSL verification warnings
from requests.packages.urllib3.exceptions import InsecureRequestWarning
warnings.simplefilter('ignore', InsecureRequestWarning)

# JFrog Artifactory instance details
JFROG_URL = 'https://frigate.jfrog.io/artifactory'  # Base URL for JFrog
ARTIFACTORY_TOKEN = 'your_artifactory_api_token'  # Replace with your API token

# Headers for the API request
headers = {
    'Authorization': f'Bearer {ARTIFACTORY_TOKEN}',
    'Accept': 'application/json',
}

def fetch_repositories():
    """
    Fetches a list of all repositories in the Artifactory instance.
    """
    url = f"{JFROG_URL}/api/repositories"
    response = requests.get(url, headers=headers, verify=False)

    if response.status_code == 200:
        return response.json()
    else:
        print(f"Failed to fetch repositories: {response.status_code}")
        return []

def list_artifacts(repo_name):
    """
    Recursively fetches all artifacts in the specified repository.
    """
    url = f"{JFROG_URL}/api/storage/{repo_name}?list&deep=1"
    response = requests.get(url, headers=headers, verify=False)

    if response.status_code == 200:
        return response.json().get("files", [])
    else:
        print(f"Failed to list artifacts for {repo_name}: {response.status_code}")
        return []

def extract_package_info(artifact_path, package_type, repo_name):
    segments = artifact_path.strip("/").split("/")
    
    if package_type == "npm":
        if len(segments) >= 3:
            package_name = segments[0] if not segments[0].startswith("@") else f"{segments[0]}/{segments[1]}"
            version = segments[-2]
        else:
            package_name = repo_name
            version = segments[-1] if segments else ""
    elif package_type == "maven":
        if len(segments) >= 3:
            package_name = ".".join(segments[:-2])
            version = segments[-2]
        else:
            package_name = repo_name
            version = segments[-1] if segments else ""
    elif package_type == "docker":
        if len(segments) >= 2:
            package_name = "/".join(segments[:-1])
            version = segments[-1]
        else:
            package_name = repo_name
            version = ""
    else:
        # Default handling for other package types
        if len(segments) >= 2:
            package_name = "/".join(segments[:-1])
            version = segments[-1]
        else:
            package_name = repo_name
            version = ""
    
    return package_name, version

def process_repository(repo_name, package_type):
    """
    Processes all artifacts in a repository using batch metadata from the recursive listing.
    """
    artifact_list = list_artifacts(repo_name)
    repo_details = []

    for artifact in artifact_list:
        artifact_path = artifact.get("uri", "")
        artifact_url = urljoin(f"{JFROG_URL}/{repo_name}", artifact_path)
        created_date = artifact.get("lastModified", "")

        package_name, version = extract_package_info(artifact_path, package_type, repo_name)

        repo_details.append({
            "repo_name": repo_name,
            "package_type": package_type,
            "package_name": package_name,
            "version": version,
            "url": artifact_url,
            "created_date": created_date,
            "license": "",  # Leave blank if not available
            "secarch": "",  # Leave blank if not available
            "artifactory_instance": "frigate.jfrog.io"
        })
    
    return repo_details

def save_to_csv(data, filename):
    """
    Saves extracted metadata to a CSV file.
    """
    headers = ['repo_name', 'package_type', 'package_name', 'version', 'url', 'created_date', 'license', 'secarch', 'artifactory_instance']
    
    with open(filename, mode='w', newline='', encoding='utf-8') as file:
        writer = csv.DictWriter(file, fieldnames=headers)
        writer.writeheader()
        writer.writerows(data)
    
    print(f"Data has been written to {filename}")

def main():
    """
    Main function to process all repositories and extract metadata.
    """
    repositories = fetch_repositories()
    all_repo_details = []

    for repo in repositories:
        repo_name = repo.get("key")
        package_type = repo.get("packageType", "")  # Dynamically fetch the package type
        print(f"Processing repository: {repo_name} (Type: {package_type})")
        
        # Process repository artifacts in batches
        repo_details = process_repository(repo_name, package_type)
        all_repo_details.extend(repo_details)

    # Save all details to CSV with a dynamic file name
    current_date = datetime.now().strftime('%Y-%m-%d')  # Current date in YYYY-MM-DD format
    output_file = f"EV_EOL_{current_date}.csv"
    save_to_csv(all_repo_details, output_file)

if __name__ == '__main__':
    main()
```

This script includes the following key features:

1. Fetches all repositories from the JFrog Artifactory instance.
2. Processes each repository, extracting metadata for all artifacts.
3. Handles package name and version extraction differently for npm, Maven, Docker, and other package types.
4. Saves the extracted data to a CSV file with a date-stamped filename.

To use this script:

1. Replace `'your_artifactory_api_token'` with your actual JFrog Artifactory API token.
2. Ensure you have the `requests` library installed (`pip install requests`).
3. Run the script in a Python environment.

The script will process all repositories in your JFrog Artifactory instance, attempt to extract accurate package names and versions for different package types, and save the results to a CSV file.
