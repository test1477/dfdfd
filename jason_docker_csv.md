Yes, we can modify the script to hardcode the JFrog URL and API key instead of using command-line arguments and environment variables. Here's an updated version of the script with hardcoded values:

```python
import requests
import csv
import warnings
from datetime import datetime
import logging

logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

warnings.simplefilter('ignore', InsecureRequestWarning)

# Hardcoded JFrog Artifactory configuration
JFROG_URL = "https://your-artifactory-instance.jfrog.io/artifactory"
API_KEY = "your-api-key-here"

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

def fetch_docker_repositories():
    url = f"{JFROG_URL}/api/repositories?type=docker"
    response = make_request(url)
    if response:
        repos = response.json()
        logging.info(f"Found {len(repos)} Docker repositories")
        return [repo['key'] for repo in repos]
    return []

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
        created_date = response.headers.get("Last-Modified", "")
        logging.info(f"Retrieved digest for {image_name}:{tag} in {repo_name}")
        return digest, created_date
    elif response and response.status_code == 403:
        logging.warning(f"Download blocked for {image_name}:{tag} in {repo_name} due to Xray policy")
        return "BLOCKED", ""
    return "", ""

def process_docker_repositories():
    all_image_details = []
    docker_repos = fetch_docker_repositories()
    
    for repo_name in docker_repos:
        logging.info(f"Processing repository: {repo_name}")
        images = fetch_docker_images(repo_name)
        for image in images:
            tags = fetch_docker_tags(repo_name, image)
            for tag in tags:
                digest, created_date = fetch_docker_manifest(repo_name, image, tag)
                if digest != "BLOCKED":
                    all_image_details.append({
                        "repo_name": repo_name,
                        "package_type": "Docker",
                        "package_name": image,
                        "version": tag,
                        "url": f"{JFROG_URL}/{repo_name}/{image}:{tag}",
                        "created_date": created_date,
                        "license": "",
                        "secarch": "",
                        "artifactory_instance": JFROG_URL
                    })
                else:
                    logging.warning(f"Skipping blocked artifact: {repo_name}/{image}:{tag}")

    logging.info(f"Total Docker images processed: {len(all_image_details)}")
    return all_image_details

def save_to_csv(data, filename):
    headers = ['repo_name', 'package_type', 'package_name', 'version', 'url', 'created_date', 'license', 'secarch', 'artifactory_instance']
    
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

To use this script:

1. Replace "https://your-artifactory-instance.jfrog.io/artifactory" with your actual JFrog Artifactory URL.
2. Replace "your-api-key-here" with your actual API key.
3. Run the script to generate the CSV report of Docker images in your Artifactory instance[1][3].

Citations:
[1] https://developer.harness.io/docs/continuous-integration/use-ci/build-and-upload-artifacts/build-and-push/build-and-push-to-docker-jfrog/
[2] https://stackoverflow.com/questions/44753357/how-to-get-the-dockertag-label-via-the-artifactory-api
[3] https://jfrog.com/blog/manage-your-docker-builds-with-jfrog-cli-in-5-easy-steps/
[4] https://jfrog.com/help/r/jfrog-artifactory-documentation/docker-labels
[5] https://docs.docker.com/scout/integrations/registry/artifactory/
[6] https://stackoverflow.com/questions/75755878/push-a-docker-image-with-curl-or-jfrog-cli
[7] https://jfrog.com/integrations/docker-registry/
[8] https://jfrog.com/help/r/jfrog-artifactory-documentation/the-repository-path-method-for-docker

---
Answer from Perplexity: pplx.ai/share
