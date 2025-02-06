To retrieve all Docker images from all repositories in JFrog Artifactory, we can modify the previous script to iterate through all Docker repositories. Here's an updated Python script that accomplishes this:

```python
import requests
import csv
import base64

# Artifactory configuration
artifactory_url = "https://your-artifactory-instance.jfrog.io/artifactory"
api_key = "your-api-key"

# Authentication
auth = base64.b64encode(f"username:{api_key}".encode()).decode()
headers = {
    "Authorization": f"Basic {auth}",
    "Content-Type": "application/json"
}

# Get list of Docker repositories
repos_url = f"{artifactory_url}/api/repositories?type=docker"
repos_response = requests.get(repos_url, headers=headers)
repos = [repo["key"] for repo in repos_response.json()]

# Prepare CSV file
csv_file = "all_docker_images_report.csv"
csv_headers = ["Docker Image Name", "CSP Placeholder", "Resource Type", "Unique ID", "Digest", "Repo Path"]

with open(csv_file, "w", newline="") as f:
    writer = csv.DictWriter(f, fieldnames=csv_headers)
    writer.writeheader()

    for repo_name in repos:
        # Get list of Docker images for each repository
        catalog_url = f"{artifactory_url}/api/docker/{repo_name}/v2/_catalog"
        response = requests.get(catalog_url, headers=headers)
        images = response.json().get("repositories", [])

        for image in images:
            # Get tags for each image
            tags_url = f"{artifactory_url}/api/docker/{repo_name}/v2/{image}/tags/list"
            tags_response = requests.get(tags_url, headers=headers)
            tags = tags_response.json().get("tags", [])

            for tag in tags:
                # Get manifest for each tag
                manifest_url = f"{artifactory_url}/api/docker/{repo_name}/v2/{image}/manifests/{tag}"
                manifest_response = requests.get(manifest_url, headers=headers)
                digest = manifest_response.headers.get("Docker-Content-Digest", "")

                # Write to CSV
                writer.writerow({
                    "Docker Image Name": f"{image}:{tag}",
                    "CSP Placeholder": "JFrog Artifactory",
                    "Resource Type": "Docker Image",
                    "Unique ID": f"{repo_name}/{image}:{tag}",
                    "Digest": digest,
                    "Repo Path": f"{repo_name}/{image}"
                })

print(f"All Docker images report exported to {csv_file}")
```

This script does the following:

1. It first retrieves a list of all Docker repositories in your Artifactory instance.
2. For each repository, it fetches the list of Docker images.
3. For each image, it retrieves the list of tags.
4. For each tag, it gets the manifest to obtain the digest.
5. Finally, it writes all the collected information to a CSV file.

To use this script:

1. Replace `your-artifactory-instance.jfrog.io` with your actual Artifactory URL.
2. Replace `your-api-key` with your Artifactory API key.
3. Install the `requests` library if you haven't already (`pip install requests`).

Run the script, and it will generate a CSV file named `all_docker_images_report.csv` containing information about all Docker images across all repositories in your Artifactory instance[7][8].

Note: This script assumes you're using API key authentication. If you're using a different authentication method, you may need to adjust the `auth` and `headers` accordingly.

Citations:
[1] https://stackoverflow.com/questions/69075946/artifactory-list-all-docker-images-in-all-repos-using-aql
[2] https://www.reddit.com/r/devops/comments/lypytb/how_to_query_a_docker_repo_on_an_artifactory/
[3] https://jfrog.com/help/r/jfrog-artifactory-documentation/browse-docker-v1-repositories
[4] https://www.reddit.com/r/docker/comments/lxt9pp/how_to_query_a_docker_registry_for_a_list_of/
[5] https://www.docker.com/blog/using-docker-desktop-and-jfrog-artifactory-for-the-enterprise/
[6] https://forums.docker.com/t/docker-hub-in-artifactory-repository-remote/136034
[7] https://jfrog.com/help/r/jfrog-artifactory-documentation/list-docker-images
[8] https://jfrog.com/help/r/jfrog-artifactory-documentation/docker-registry

---
Answer from Perplexity: pplx.ai/share
