Certainly! Here's the updated full script with a description header including the author information:

```python
"""
JFrog Artifactory Docker Image Metadata Exporter

This script retrieves Docker image metadata from JFrog Artifactory and exports it to a CSV file.
It fetches information about all Docker repositories, images, and tags, including their digests.

Author: JAM
Created: February 6, 2025
Version: 1.0

Usage:
1. Set the artifactory_url and api_key variables with your Artifactory instance details.
2. Run the script to generate a CSV report of all Docker images in your Artifactory.

Output:
- CSV file with columns: Docker Image Name, CSP Placeholder, Resource Type, Unique ID, Digest, Repo Path
"""

import os
import requests
import csv
import urllib3

# Disable SSL warnings
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

# Artifactory configuration
artifactory_url = "https://your-artifactory-instance.jfrog.io/artifactory"  # Replace with your Artifactory URL
api_key = "your-api-key"  # Replace with your Artifactory API key

# Authentication
headers = {
    "X-JFrog-Art-Api": api_key,
    "Content-Type": "application/json"
}

def fetch_docker_repositories():
    """Fetches a list of all Docker repositories in Artifactory."""
    repos_url = f"{artifactory_url}/api/repositories?type=docker"
    print(f"Fetching Docker repositories from {repos_url}...")
    response = requests.get(repos_url, headers=headers, verify=False)
    response.raise_for_status()
    return [repo["key"] for repo in response.json()]

def fetch_docker_images(repo_name):
    """Fetches a list of all Docker images in a specific repository."""
    catalog_url = f"{artifactory_url}/api/docker/{repo_name}/v2/_catalog"
    print(f"Fetching Docker images from {catalog_url}...")
    response = requests.get(catalog_url, headers=headers, verify=False)
    response.raise_for_status()
    return response.json().get("repositories", [])

def fetch_docker_tags(repo_name, image_name):
    """Fetches a list of all tags for a specific Docker image in a repository."""
    tags_url = f"{artifactory_url}/api/docker/{repo_name}/v2/{image_name}/tags/list"
    print(f"Fetching tags for image {image_name} in repository {repo_name}...")
    response = requests.get(tags_url, headers=headers, verify=False)
    response.raise_for_status()
    return response.json().get("tags", [])

def fetch_docker_manifest(repo_name, image_name, tag):
    """Fetches the manifest for a specific Docker image tag and returns the digest."""
    manifest_url = f"{artifactory_url}/api/docker/{repo_name}/v2/{image_name}/manifests/{tag}"
    print(f"Fetching manifest for image {image_name}:{tag} in repository {repo_name}...")
    response = requests.get(manifest_url, headers=headers, verify=False)
    response.raise_for_status()
    return response.headers.get("Docker-Content-Digest", "")

def generate_csv_report(output_file="docker_images_report.csv"):
    """Generates a CSV report of all Docker images in Artifactory."""
    repos = fetch_docker_repositories()
    if not repos:
        print("No Docker repositories found. Exiting.")
        return

    csv_headers = ["Docker Image Name", "CSP Placeholder", "Resource Type", "Unique ID", "Digest", "Repo Path"]
    with open(output_file, "w", newline="") as f:
        writer = csv.DictWriter(f, fieldnames=csv_headers)
        writer.writeheader()

        for repo_name in repos:
            print(f"Processing repository: {repo_name}")
            images = fetch_docker_images(repo_name)
            for image in images:
                tags = fetch_docker_tags(repo_name, image)
                for tag in tags:
                    try:
                        digest = fetch_docker_manifest(repo_name, image, tag)
                        writer.writerow({
                            "Docker Image Name": f"{image}:{tag}",
                            "CSP Placeholder": "JFrog Artifactory",
                            "Resource Type": "Docker Image",
                            "Unique ID": f"{repo_name}/{image}:{tag}",
                            "Digest": digest,
                            "Repo Path": f"{repo_name}/{image}"
                        })
                    except requests.RequestException as e:
                        print(f"Error processing {image}:{tag}: {str(e)}")

    print(f"All Docker images report exported to {output_file}")

if __name__ == "__main__":
    generate_csv_report()
```

This updated script includes:

1. A comprehensive header with information about the script's purpose, author (JAM), creation date, version, and usage instructions.
2. Improved error handling using `response.raise_for_status()` to catch HTTP errors.
3. A try-except block in the main loop to catch and report any errors during the processing of individual images or tags.
4. Cleaner function definitions with docstrings for better code documentation.

To use this script:

1. Replace `your-artifactory-instance.jfrog.io` with your actual Artifactory URL.
2. Replace `your-api-key` with your Artifactory API key.
3. Run the script to generate the CSV report.

This script should provide a robust solution for retrieving Docker image metadata from JFrog Artifactory and exporting it to a CSV file with the specified columns.

---
Answer from Perplexity: pplx.ai/share
