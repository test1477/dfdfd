You're right. Let's modify the script to first get all Docker repositories, then extract the Docker images list from each repository. I'll also add brief comments explaining each step. Here's the updated script:

```python
import requests
import csv
import base64
import urllib3
import json
import logging

# Set up logging
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

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

def get_json_safely(response):
    try:
        return response.json()
    except json.JSONDecodeError:
        logging.error(f"Failed to decode JSON from response: {response.text}")
        return {}

def make_request(url):
    try:
        response = requests.get(url, headers=headers, verify=False)
        response.raise_for_status()
        return response
    except requests.RequestException as e:
        logging.error(f"Request failed for URL {url}: {str(e)}")
        return None

# Step 1: Get list of all repositories
logging.info("Fetching list of all repositories")
all_repos_url = f"{artifactory_url}/api/repositories"
all_repos_response = make_request(all_repos_url)
all_repos = get_json_safely(all_repos_response)

# Step 2: Filter for Docker repositories
docker_repos = [repo for repo in all_repos if repo.get('packageType') == 'docker']
logging.info(f"Found {len(docker_repos)} Docker repositories")

# Prepare CSV file
csv_file = "all_docker_images_report.csv"
csv_headers = ["Docker Image Name", "CSP Placeholder", "Resource Type", "Unique ID", "Digest", "Repo Path"]

with open(csv_file, "w", newline="") as f:
    writer = csv.DictWriter(f, fieldnames=csv_headers)
    writer.writeheader()

    for repo in docker_repos:
        repo_name = repo['key']
        logging.info(f"Processing repository: {repo_name}")
        
        # Step 3: Get list of Docker images for each repository
        catalog_url = f"{artifactory_url}/api/docker/{repo_name}/v2/_catalog"
        catalog_response = make_request(catalog_url)
        if catalog_response:
            catalog_data = get_json_safely(catalog_response)
            images = catalog_data.get("repositories", [])
            logging.info(f"Found {len(images)} images in repository {repo_name}")
        else:
            images = []
            logging.error(f"Failed to retrieve images for repository {repo_name}")

        for image in images:
            logging.info(f"Processing image: {image}")
            
            # Step 4: Get tags for each image
            tags_url = f"{artifactory_url}/api/docker/{repo_name}/v2/{image}/tags/list"
            tags_response = make_request(tags_url)
            if tags_response:
                tags_data = get_json_safely(tags_response)
                tags = tags_data.get("tags", [])
                logging.info(f"Found {len(tags)} tags for image {image}")
            else:
                tags = []
                logging.error(f"Failed to retrieve tags for image {image}")

            for tag in tags:
                # Step 5: Get manifest for each tag
                manifest_url = f"{artifactory_url}/api/docker/{repo_name}/v2/{image}/manifests/{tag}"
                manifest_response = make_request(manifest_url)
                if manifest_response:
                    digest = manifest_response.headers.get("Docker-Content-Digest", "")
                else:
                    digest = ""
                    logging.error(f"Failed to retrieve manifest for {image}:{tag}")

                # Step 6: Write data to CSV
                writer.writerow({
                    "Docker Image Name": f"{image}:{tag}",
                    "CSP Placeholder": "JFrog Artifactory",
                    "Resource Type": "Docker Image",
                    "Unique ID": f"{repo_name}/{image}:{tag}",
                    "Digest": digest,
                    "Repo Path": f"{repo_name}/{image}"
                })
                logging.info(f"Wrote data for {image}:{tag}")

logging.info(f"All Docker images report exported to {csv_file}")
```

This script does the following:

1. Fetches a list of all repositories from Artifactory.
2. Filters the repositories to include only those with `packageType` set to 'docker'.
3. For each Docker repository, retrieves the list of images.
4. For each image, fetches the list of tags.
5. For each tag, retrieves the manifest to get the digest.
6. Writes the collected information to a CSV file.

Make sure to replace `your-artifactory-instance.jfrog.io` with your actual Artifactory URL and `your-api-key` with your Artifactory API key.

This approach should ensure that you're capturing all Docker images across all Docker repositories in your Artifactory instance.

---
Answer from Perplexity: pplx.ai/share
