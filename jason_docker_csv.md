I apologize for the confusion. I'll update the script to use the correct column names and data structure you specified. Here's the revised version of the script:

```python
import requests
import csv
import warnings
from datetime import datetime
import logging
from requests.packages.urllib3.exceptions import InsecureRequestWarning

# Set up logging
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

# Suppress SSL verification warnings
warnings.simplefilter('ignore', InsecureRequestWarning)

# Hardcoded JFrog Artifactory configuration
JFROG_URL = "https://your-artifactory-instance.jfrog.io/artifactory"
API_KEY = "your-api-key-here"

headers = {
    'X-JFrog-Art-Api': API_KEY,
    'Accept': 'application/json',
}

def fetch_repositories():
    url = f"{JFROG_URL}/api/repositories"
    try:
        response = requests.get(url, headers=headers, verify=False)
        response.raise_for_status()
        return response.json()
    except requests.RequestException as e:
        logging.error(f"Failed to fetch repositories: {e}")
        return []

def list_artifacts(repo_name):
    url = f"{JFROG_URL}/api/storage/{repo_name}?list&deep=1"
    try:
        response = requests.get(url, headers=headers, verify=False)
        response.raise_for_status()
        return response.json().get("files", [])
    except requests.RequestException as e:
        logging.error(f"Failed to list artifacts for {repo_name}: {e}")
        return []

def fetch_docker_repositories():
    all_repos = fetch_repositories()
    docker_repos = [repo['key'] for repo in all_repos if repo.get('packageType') == 'docker']
    logging.info(f"Found {len(docker_repos)} Docker repositories")
    return docker_repos

def fetch_docker_manifest(repo_name, image_name, tag):
    url = f"{JFROG_URL}/api/docker/{repo_name}/v2/{image_name}/manifests/{tag}"
    try:
        response = requests.get(url, headers=headers, verify=False)
        response.raise_for_status()
        return response.headers.get("Docker-Content-Digest", "")
    except requests.RequestException as e:
        logging.error(f"Failed to fetch manifest for {repo_name}/{image_name}:{tag}: {e}")
        return ""

def process_docker_repositories():
    all_image_details = []
    docker_repos = fetch_docker_repositories()
    
    for repo_name in docker_repos:
        logging.info(f"Processing repository: {repo_name}")
        artifacts = list_artifacts(repo_name)
        for artifact in artifacts:
            if not artifact['uri'].startswith('/.jfrog'):
                image_name = artifact['uri'].split(':')[0].strip('/')
                tag = artifact['uri'].split(':')[-1] if ':' in artifact['uri'] else 'latest'
                digest = fetch_docker_manifest(repo_name, image_name, tag)
                all_image_details.append({
                    "Docker Image Name": f"{image_name}:{tag}",
                    "CSP Placeholder": "JFrog Artifactory",
                    "Resource Type": "Docker Image",
                    "Unique ID": f"{repo_name}/{image_name}:{tag}",
                    "Digest": digest,
                    "Repo Path": f"{repo_name}/{image_name}"
                })

    logging.info(f"Total Docker images processed: {len(all_image_details)}")
    return all_image_details

def save_to_csv(data, filename):
    headers = ['Docker Image Name', 'CSP Placeholder', 'Resource Type', 'Unique ID', 'Digest', 'Repo Path']
    
    with open(filename, mode='w', newline='', encoding='utf-8') as file:
        writer = csv.DictWriter(file, fieldnames=headers)
        writer.writeheader()
        writer.writerows(data)
    
    logging.info(f"Data has been written to {filename}")

def main():
    logging.info("Starting Docker image metadata retrieval for all repositories")
    all_image_details = process_docker_repositories()

    if not all_image_details:
        logging.warning("No Docker image details were retrieved. The output file will be empty.")

    current_date = datetime.now().strftime('%Y-%m-%d')
    output_file = f"Docker_Images_Report_{current_date}.csv"
    save_to_csv(all_image_details, output_file)

    logging.info("Script execution completed")

if __name__ == '__main__':
    main()
```

This script now uses the correct column names and data structure you specified. It fetches Docker repositories, lists artifacts in each repository, and then processes these artifacts to extract Docker image information. The script also fetches the manifest for each image to get the digest.

Remember to replace the `JFROG_URL` and `API_KEY` with your actual Artifactory URL and API key before running the script.

---
Answer from Perplexity: pplx.ai/share
