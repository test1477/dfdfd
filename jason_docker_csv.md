Certainly! Here's the full updated script incorporating all the changes we've discussed:

```python
"""
JFrog Artifactory Docker Image Metadata Exporter

This script retrieves metadata for all Docker images across all Docker repositories
in a JFrog Artifactory instance. It generates a CSV report containing information
about each image, including resource name, CSP placeholder, resource type, unique ID,
digest, and repository path. The script excludes .jfrog cache folders and handles
artifacts blocked by Xray policies.

Author: [Your Name]
Date: [Current Date]
Version: 1.2

Usage:
1. Set the JFROG_URL and API_KEY variables with your Artifactory instance details.
2. Run the script to generate a CSV report of all Docker images across all repositories.

Output: A CSV file named 'Docker_Images_Report_[DATE].csv' in the same directory as the script.
"""

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

# JFrog Artifactory configuration
JFROG_URL = "https://your-artifactory-instance.jfrog.io/artifactory"  # Replace with your Artifactory URL
API_KEY = "your-api-key-here"  # Replace with your API key

# Headers for the API request
headers = {
    'X-JFrog-Art-Api': API_KEY,
    'Accept': 'application/json',
}

def make_request(url):
    try:
        response = requests.get(url, headers=headers, verify=False)
        response.raise_for_status()
        return response
    except requests.RequestException as e:
        logging.error(f"Request failed for URL {url}: {str(e)}")
        if hasattr(e, 'response') and e.response is not None:
            logging.error(f"Response content: {e.response.text}")
        return None

def fetch_all_repositories():
    url = f"{JFROG_URL}/api/repositories"
    response = make_request(url)
    if response:
        repos = response.json()
        logging.info(f"Found {len(repos)} total repositories")
        return repos
    return []

def fetch_docker_repositories():
    all_repos = fetch_all_repositories()
    docker_repos = [repo['key'] for repo in all_repos if repo.get('packageType') == 'docker']
    logging.info(f"Found {len(docker_repos)} Docker repositories")
    return docker_repos

def fetch_docker_images(repo_name):
    url = f"{JFROG_URL}/api/docker/{repo_name}/v2/_catalog"
    response = make_request(url)
    if response:
        images = response.json().get("repositories", [])
        filtered_images = [img for img in images if not img.startswith('.jfrog')]
        logging.info(f"Found {len(filtered_images)} images in repository {repo_name}")
        return filtered_images
    return []

def fetch_docker_tags(repo_name, image_name):
    url = f"{JFROG_URL}/api/docker/{repo_name}/v2/{image_name}/tags/list"
    response = make_request(url)
    if response:
        tags = response.json().get("tags", [])
        logging.info(f"Found {len(tags)} tags for image {image_name} in {repo_name}")
        return tags
    return []

def fetch_docker_manifest(repo_name, image_name, tag):
    url = f"{JFROG_URL}/api/docker/{repo_name}/v2/{image_name}/manifests/{tag}"
    response = make_request(url)
    if response:
        digest = response.headers.get("Docker-Content-Digest", "")
        logging.info(f"Retrieved digest for {image_name}:{tag} in {repo_name}")
        return digest
    elif response and response.status_code == 403:
        logging.warning(f"Download blocked for {image_name}:{tag} in {repo_name} due to Xray policy")
        return "BLOCKED"
    return ""

def process_docker_repositories():
    all_image_details = []
    
    # Debug: Print all repositories
    all_repos = fetch_all_repositories()
    print("All repository names:")
    for repo in all_repos:
        print(f"- {repo['key']} (Type: {repo.get('packageType', 'Unknown')})")

    # Debug: Print Docker repositories
    docker_repos = fetch_docker_repositories()
    print("Docker repository names:")
    for repo in docker_repos:
        print(f"- {repo}")

    for repo_name in docker_repos:
        logging.info(f"Processing repository: {repo_name}")
        images = fetch_docker_images(repo_name)
        for image in images:
            tags = fetch_docker_tags(repo_name, image)
            for tag in tags:
                digest = fetch_docker_manifest(repo_name, image, tag)
                if digest != "BLOCKED":
                    all_image_details.append({
                        "Resource Name": image.split('/')[-1],
                        "CSP Placeholder": "JFrog Artifactory",
                        "Resource Type": "Container Image",
                        "Unique ID": digest,
                        "Digest": digest,
                        "Repo Path": f"{repo_name}/{image}"
                    })
                else:
                    logging.warning(f"Skipping blocked artifact: {repo_name}/{image}:{tag}")

    logging.info(f"Total Docker images processed: {len(all_image_details)}")
    return all_image_details

def save_to_csv(data, filename):
    headers = ['Resource Name', 'CSP Placeholder', 'Resource Type', 'Unique ID', 'Digest', 'Repo Path']
    
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

This updated script includes all the modifications we've discussed:

1. It fetches all repositories and filters for Docker repositories.
2. It excludes .jfrog cache folders when processing Docker images.
3. It handles artifacts blocked by Xray policies, logging warnings and skipping them.
4. It includes debug print statements to show all repository names and their types, as well as the filtered Docker repository names.

To use this script:

1. Replace `"https://your-artifactory-instance.jfrog.io/artifactory"` with your actual JFrog Artifactory URL.
2. Replace `"your-api-key-here"` with your actual API key.
3. Run the script.

This script should now handle the issues you were facing, including excluding cache folders and dealing with blocked artifacts. It will provide a comprehensive report of all accessible Docker images in your Artifactory instance.

---
Answer from Perplexity: pplx.ai/share
