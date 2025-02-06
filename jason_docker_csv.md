Certainly! Let's modify the script to focus on a single Docker repository. We'll hardcode the repository name and simplify the script to retrieve images and tags from just that repository. Here's the updated script:

```python
import requests
import csv
import warnings
from datetime import datetime
import logging

# Set up logging
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

# Suppress SSL verification warnings
from requests.packages.urllib3.exceptions import InsecureRequestWarning
warnings.simplefilter('ignore', InsecureRequestWarning)

# JFrog Artifactory configuration
JFROG_URL = "https://your-artifactory-instance.jfrog.io/artifactory"  # Replace with your Artifactory URL
API_KEY = "your-api-key-here"  # Replace with your API key
REPO_NAME = "your-docker-repo-name"  # Replace with your Docker repository name

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

def fetch_docker_images():
    url = f"{JFROG_URL}/api/docker/{REPO_NAME}/v2/_catalog"
    response = make_request(url)
    if response:
        images = response.json().get("repositories", [])
        logging.info(f"Found {len(images)} images in repository {REPO_NAME}")
        return images
    return []

def fetch_docker_tags(image_name):
    url = f"{JFROG_URL}/api/docker/{REPO_NAME}/v2/{image_name}/tags/list"
    response = make_request(url)
    if response:
        tags = response.json().get("tags", [])
        logging.info(f"Found {len(tags)} tags for image {image_name}")
        return tags
    return []

def fetch_docker_manifest(image_name, tag):
    url = f"{JFROG_URL}/api/docker/{REPO_NAME}/v2/{image_name}/manifests/{tag}"
    response = make_request(url)
    if response:
        digest = response.headers.get("Docker-Content-Digest", "")
        logging.info(f"Retrieved digest for {image_name}:{tag}")
        return digest
    return ""

def process_docker_images():
    all_image_details = []
    images = fetch_docker_images()

    for image in images:
        logging.info(f"Processing image: {image}")
        tags = fetch_docker_tags(image)
        for tag in tags:
            digest = fetch_docker_manifest(image, tag)
            all_image_details.append({
                "Docker Image Name": f"{image}:{tag}",
                "CSP Placeholder": "JFrog Artifactory",
                "Resource Type": "Docker Image",
                "Unique ID": f"{REPO_NAME}/{image}:{tag}",
                "Digest": digest,
                "Repo Path": f"{REPO_NAME}/{image}"
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
    logging.info(f"Starting Docker image metadata retrieval for repository: {REPO_NAME}")
    all_image_details = process_docker_images()

    if not all_image_details:
        logging.warning("No Docker image details were retrieved. The output file will be empty.")

    # Save all details to CSV with a dynamic file name
    current_date = datetime.now().strftime('%Y-%m-%d')
    output_file = f"Docker_Images_Report_{REPO_NAME}_{current_date}.csv"
    save_to_csv(all_image_details, output_file)

    logging.info("Script execution completed")

if __name__ == '__main__':
    main()
```

To use this script:

1. Replace `"https://your-artifactory-instance.jfrog.io/artifactory"` with your actual JFrog Artifactory URL.
2. Replace `"your-api-key-here"` with your actual API key.
3. Replace `"your-docker-repo-name"` with the name of the specific Docker repository you want to process.
4. Run the script and check the console output for detailed logging information.

This script focuses on a single Docker repository, which should simplify the process and help identify any specific issues. If you're still encountering problems, please run the script and share the log output. This will provide more detailed information about where the process might be failing.

---
Answer from Perplexity: pplx.ai/share
