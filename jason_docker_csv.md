If you're getting an empty report, it could be due to several reasons. Let's modify the script to add more debugging information and error handling. This will help us understand where the process is failing. Here's an updated version of the script with additional logging:

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

# Get list of Docker repositories
repos_url = f"{artifactory_url}/api/repositories?type=docker"
repos_response = make_request(repos_url)
if repos_response:
    repos_data = get_json_safely(repos_response)
    repos = [repo["key"] for repo in repos_data if isinstance(repo, dict) and "key" in repo]
    logging.info(f"Found {len(repos)} Docker repositories")
else:
    repos = []
    logging.error("Failed to retrieve Docker repositories")

# Prepare CSV file
csv_file = "all_docker_images_report.csv"
csv_headers = ["Docker Image Name", "CSP Placeholder", "Resource Type", "Unique ID", "Digest", "Repo Path"]

with open(csv_file, "w", newline="") as f:
    writer = csv.DictWriter(f, fieldnames=csv_headers)
    writer.writeheader()

    for repo_name in repos:
        logging.info(f"Processing repository: {repo_name}")
        
        # Get list of Docker images for each repository
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
            
            # Get tags for each image
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
                # Get manifest for each tag
                manifest_url = f"{artifactory_url}/api/docker/{repo_name}/v2/{image}/manifests/{tag}"
                manifest_response = make_request(manifest_url)
                if manifest_response:
                    digest = manifest_response.headers.get("Docker-Content-Digest", "")
                else:
                    digest = ""
                    logging.error(f"Failed to retrieve manifest for {image}:{tag}")

                # Write to CSV
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

This updated script includes:

1. More detailed logging at each step.
2. Error handling for API requests.
3. Checks to ensure we're not processing empty lists.

To use this script:

1. Replace `your-artifactory-instance.jfrog.io` with your actual Artifactory URL.
2. Replace `your-api-key` with your Artifactory API key.
3. Run the script and check the console output for any error messages or unexpected behavior.

If you still get an empty report, please check the console output for any error messages. The most common reasons for an empty report could be:

1. Incorrect Artifactory URL or API key.
2. No Docker repositories found in your Artifactory instance.
3. API permissions issues (the API key might not have sufficient permissions).
4. Network connectivity issues.

If you see any specific error messages in the console output, please share them, and I can provide more targeted assistance.

---
Answer from Perplexity: pplx.ai/share
