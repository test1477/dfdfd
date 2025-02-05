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
    url = f"{JFROG_URL}/api/repositories"
    try:
        response = requests.get(url, headers=headers, verify=False)
        response.raise_for_status()
        return response.json()
    except requests.RequestException as e:
        print(f"Failed to fetch repositories: {e}")
        return []

# Function to list all artifacts in a repository
def list_artifacts(repo_name):
    url = f"{JFROG_URL}/api/storage/{repo_name}?list&deep=1"
    try:
        response = requests.get(url, headers=headers, verify=False)
        response.raise_for_status()
        return response.json().get("files", [])
    except requests.RequestException as e:
        print(f"Failed to list artifacts for {repo_name}: {e}")
        return []

# Function to process repository artifacts
def process_repository(repo_name, package_type):
    try:
        artifact_list = list_artifacts(repo_name)
        repo_details = []

        for artifact in artifact_list:
            artifact_path = artifact.get("uri", "")

            # Exclude .jfrog cache folder
            if ".jfrog" in artifact_path:
                continue

            artifact_url = urljoin(f"{JFROG_URL}/{repo_name}", artifact_path)
            created_date = artifact.get("lastModified", "")

            segments = artifact_path.strip("/").split("/")
            if len(segments) >= 3:
                package_name = " ".join(segments[-3:-1])
                version = segments[-2]
            else:
                package_name = repo_name
                version = ""

            if package_type == "Docker":
                repo_details.append({
                    "repo_name": repo_name,
                    "package_type": package_type,
                    "package_name": package_name,
                    "version": version,
                    "url": artifact_url,
                    "created_date": created_date,
                    "license": "",
                    "secarch": "",
                    "artifactory_instance": "frigate.jfrog.io"
                })

        return repo_details

    except Exception as e:
        print(f"Skipping repository {repo_name} due to error: {e}")
        return []

# Function to save data to CSV
def save_to_csv(data, filename):
    headers = ['repo_name', 'package_type', 'package_name', 'version', 'url', 'created_date', 'license', 'secarch', 'artifactory_instance']
    os.makedirs(os.path.dirname(filename), exist_ok=True)

    with open(filename, mode='w', newline='', encoding='utf-8') as file:
        writer = csv.DictWriter(file, fieldnames=headers)
        writer.writeheader()
        writer.writerows(data)
    
    print(f"Data has been written to {filename}")

# Main function to process all repositories
def main():
    repositories = fetch_repositories()
    all_repo_details = []

    if not repositories:
        print("No repositories found or failed to fetch repositories.")
        return

    for repo in repositories:
        repo_name = repo.get("key")
        package_type = repo.get("packageType", "")

        print(f"Processing repository: {repo_name} (Type: {package_type})")

        try:
            if package_type == "Docker":
                repo_details = process_repository(repo_name, package_type)
                all_repo_details.extend(repo_details)
        except Exception as e:
            print(f"Skipping repository {repo_name} due to unexpected error: {e}")

    current_date = datetime.now().strftime('%Y-%m-%d')
    output_file = os.path.join(args.output, f"EV_EOL_{current_date}_docker.csv")
    save_to_csv(all_repo_details, output_file)

if __name__ == '__main__':
    main()
