Here's the updated script with all symbols removed:  

```python
import os
import requests
import csv
import warnings
from urllib.parse import urljoin
from datetime import datetime
import argparse

# Suppress SSL verification warnings
from requests.packages.urllib3.exceptions import InsecureRequestWarning
warnings.simplefilter('ignore', InsecureRequestWarning)

# Parse command-line arguments
parser = argparse.ArgumentParser(description="Generate JFrog reports for multiple organizations.")
parser.add_argument('--org', required=True, help="The organization name (e.g., 'ev' or 'ppa').")
parser.add_argument('--jfrog-url', required=True, help="The JFrog Artifactory URL.")
parser.add_argument('--output', required=True, help="The output directory for the generated report.")
args = parser.parse_args()

# Get JFrog Artifactory instance details
JFROG_URL = args.jfrog_url
ARTIFACTORY_TOKEN = os.getenv(f'JFROG_API_KEY_{args.org.upper()}')  # Get API key for the given organization

if not ARTIFACTORY_TOKEN:
    print(f"Error: JFROG_API_KEY_{args.org.upper()} is not set in environment variables.")
    exit(1)

# Headers for the API request
headers = {
    'Authorization': f'Bearer {ARTIFACTORY_TOKEN}',
    'Accept': 'application/json',
}

# Function to fetch all repositories
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

# Function to list all artifacts in a repository
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

# Function to process repository artifacts without extra API calls
def process_repository(repo_name, package_type):
    """
    Processes all artifacts in a repository using batch metadata from the recursive listing.
    """
    try:
        artifact_list = list_artifacts(repo_name)
        repo_details = []

        for artifact in artifact_list:
            artifact_path = artifact.get("uri", "")
            artifact_url = urljoin(f"{JFROG_URL}/{repo_name}", artifact_path)
            created_date = artifact.get("lastModified", "")

            # Parse package name and version from artifact path
            segments = artifact_path.strip("/").split("/")
            if len(segments) >= 2:
                package_name = f"{repo_name}/{'/'.join(segments[:-2])}"  # Up to the version folder
                version = segments[-2]  # Second-to-last segment
            else:
                package_name = repo_name
                version = ""

            # Append details only for Docker repositories
            if package_type == "Docker":
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

    except Exception as e:
        print(f"Skipping repository {repo_name} due to error: {e}")
        return []  # Return an empty list to avoid breaking execution

# Function to save data to CSV
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

# Main function to process all repositories
def main():
    """
    Main function to process all repositories and extract metadata.
    """
    repositories = fetch_repositories()
    all_repo_details = []
    skipped_repositories = []  # To store skipped repositories

    for repo in repositories:
        repo_name = repo.get("key")
        package_type = repo.get("packageType", "")  # Dynamically fetch the package type
        print(f"Processing repository: {repo_name} (Type: {package_type})")

        try:
            if package_type == "Docker":
                repo_details = process_repository(repo_name, package_type)
                all_repo_details.extend(repo_details)
        except Exception as e:
            print(f"Error processing {repo_name}: {e}")
            skipped_repositories.append(repo_name)

    # Save all details to CSV with a dynamic file name
    current_date = datetime.now().strftime('%Y-%m-%d')  # Current date in YYYY-MM-DD format
    output_file = os.path.join(args.output, f"EV_EOL_{current_date}_docker.csv")  # Save only Docker reports
    save_to_csv(all_repo_details, output_file)

    # Show skipped repositories in GitHub Actions output
    if skipped_repositories:
        print("\nSkipped Repositories:")
        for repo in skipped_repositories:
            print(repo)

if __name__ == '__main__':
    main()
```

This version is **fully cleaned up** with no symbols. Let me know if you need any further modifications!
