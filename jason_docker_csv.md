Here's the updated script with the requested changes:

```python
import requests
import csv
import warnings
from datetime import datetime
import logging

logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

warnings.simplefilter('ignore', InsecureRequestWarning)

JFROG_URL = "https://your-artifactory-instance.jfrog.io/artifactory"
API_KEY = "your-api-key-here"
REPO_NAME = "your-docker-repo-name"

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
                "Resource Name": image.split('/')[-1],
                "CSP Placeholder": "JFrog Artifactory",
                "Resource Type": "Container Image",
                "Unique ID": digest,
                "Digest": digest,
                "Repo Path": f"{REPO_NAME}/{image}"
            })

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
    logging.info(f"Starting Docker image metadata retrieval for repository: {REPO_NAME}")
    all_image_details = process_docker_images()

    if not all_image_details:
        logging.warning("No Docker image details were retrieved. The output file will be empty.")

    current_date = datetime.now().strftime('%Y-%m-%d')
    output_file = f"Docker_Images_Report_{REPO_NAME}_{current_date}.csv"
    save_to_csv(all_image_details, output_file)

    logging.info("Script execution completed")

if __name__ == '__main__':
    main()
```

This updated script includes the following changes:

1. "Docker Image Name" column has been renamed to "Resource Name".
2. The "Resource Name" now only includes the last part of the image name (e.g., "frontend_angular-apm-dd").
3. "Resource Type" is now set to "Container Image".
4. "Unique ID" is now set to the same value as the "Digest".

Remember to replace the placeholder values for JFROG_URL, API_KEY, and REPO_NAME with your actual Artifactory details before running the script.

Citations:
[1] https://developer.harness.io/docs/continuous-integration/use-ci/build-and-upload-artifacts/build-and-push/build-and-push-to-docker-jfrog/
[2] https://docs.docker.com/scout/integrations/registry/artifactory/
[3] https://stackoverflow.com/questions/44753357/how-to-get-the-dockertag-label-via-the-artifactory-api
[4] https://jfrog.com/help/r/jfrog-artifactory-documentation/jfrog-container-registry
[5] https://github.com/jfrog/Evidence-Examples
[6] https://jfrog.com/help/r/jfrog-artifactory-documentation/docker-registry
[7] https://jfrog.com/blog/manage-your-docker-builds-with-jfrog-cli-in-5-easy-steps/
[8] https://jfrog.com/help/r/jfrog-rest-apis/artifactory-rest-apis
[9] https://jfrog.com/help/r/jfrog-artifactory-documentation/docker-labels
[10] https://jfrog.com/integrations/docker-registry/

---
Answer from Perplexity: pplx.ai/share
