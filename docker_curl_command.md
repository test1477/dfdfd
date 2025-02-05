If the script is reporting **"No Docker images found"**, it could be due to one of the following reasons:

1. **No Docker Repositories**: There might be no Docker repositories in your JFrog Artifactory organization.
2. **No Images in Repositories**: The Docker repositories might exist, but they might not contain any images.
3. **API Access Issues**: The script might not have the necessary permissions to access the repositories or images.
4. **Incorrect API Endpoint**: The API endpoint being used might not be correct for your Artifactory setup.

To debug this issue, you can use `curl` commands to manually fetch metadata in JSON format from JFrog Artifactory. This will help you verify if the issue is with the script or the Artifactory setup.

---

### **Using `curl` to Fetch Docker Image Metadata**

Below are the `curl` commands you can use to fetch metadata for Docker images in JFrog Artifactory.

#### **1. List All Repositories**
To list all repositories in your Artifactory instance:
```bash
curl -X GET -H "X-JFrog-Art-Api: <your-api-key>" \
     -H "Content-Type: application/json" \
     "https://<your-artifactory-domain>/artifactory/api/repositories"
```

- Replace `<your-api-key>` with your Artifactory API key.
- Replace `<your-artifactory-domain>` with your Artifactory domain.

#### **2. List Docker Images in a Repository**
To list all Docker images in a specific repository:
```bash
curl -X GET -H "X-JFrog-Art-Api: <your-api-key>" \
     -H "Content-Type: application/json" \
     "https://<your-artifactory-domain>/artifactory/api/docker/<repo-name>/v2/_catalog"
```

- Replace `<repo-name>` with the name of your Docker repository.

#### **3. List Tags for a Docker Image**
To list all tags for a specific Docker image in a repository:
```bash
curl -X GET -H "X-JFrog-Art-Api: <your-api-key>" \
     -H "Content-Type: application/json" \
     "https://<your-artifactory-domain>/artifactory/api/docker/<repo-name>/v2/<image-name>/tags/list"
```

- Replace `<image-name>` with the name of the Docker image.

#### **4. Fetch Manifest for a Docker Image Tag**
To fetch the manifest (metadata) for a specific Docker image tag:
```bash
curl -X GET -H "X-JFrog-Art-Api: <your-api-key>" \
     -H "Content-Type: application/json" \
     "https://<your-artifactory-domain>/artifactory/api/docker/<repo-name>/v2/<image-name>/manifests/<tag>"
```

- Replace `<tag>` with the tag of the Docker image.

#### **5. Fetch Image Digest**
To fetch the digest of a Docker image tag:
```bash
curl -I -X GET -H "X-JFrog-Art-Api: <your-api-key>" \
     -H "Content-Type: application/json" \
     "https://<your-artifactory-domain>/artifactory/api/docker/<repo-name>/v2/<image-name>/manifests/<tag>"
```

- The `Docker-Content-Digest` header in the response contains the digest.

---

### **Example Output**

#### **List All Repositories**
```json
[
  {
    "key": "docker-repo",
    "type": "DOCKER",
    "url": "https://artifactory.example.com/artifactory/docker-repo"
  },
  {
    "key": "generic-repo",
    "type": "GENERIC",
    "url": "https://artifactory.example.com/artifactory/generic-repo"
  }
]
```

#### **List Docker Images in a Repository**
```json
{
  "repositories": [
    "my-image-1",
    "my-image-2"
  ]
}
```

#### **List Tags for a Docker Image**
```json
{
  "name": "my-image-1",
  "tags": [
    "latest",
    "v1.0"
  ]
}
```

#### **Fetch Manifest for a Docker Image Tag**
```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
  "config": {
    "mediaType": "application/vnd.docker.container.image.v1+json",
    "size": 7023,
    "digest": "sha256:abc123..."
  },
  "layers": [
    {
      "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
      "size": 32654,
      "digest": "sha256:def456..."
    }
  ]
}
```

#### **Fetch Image Digest**
```bash
HTTP/2 200
content-type: application/vnd.docker.distribution.manifest.v2+json
docker-content-digest: sha256:abc123...
```

---

### **Debugging Steps**
1. **Verify Repositories**:
   - Use the `curl` command to list all repositories and check if any Docker repositories exist.

2. **Verify Docker Images**:
   - Use the `curl` command to list Docker images in a specific repository and check if any images exist.

3. **Verify Tags**:
   - Use the `curl` command to list tags for a specific Docker image and check if any tags exist.

4. **Check Permissions**:
   - Ensure that the API key has the necessary permissions to access the repositories and images.

5. **Check Artifactory Setup**:
   - Verify that Docker repositories are properly configured in Artifactory.

---

### **Integrating `curl` with Python**
If you want to use `curl` commands within your Python script, you can use the `subprocess` module to execute shell commands.

Example:
```python
import subprocess
import json

def run_curl_command(command):
    result = subprocess.run(command, shell=True, capture_output=True, text=True)
    if result.returncode != 0:
        raise Exception(f"curl command failed: {result.stderr}")
    return json.loads(result.stdout)

# Example: List Docker images in a repository
repo_name = "docker-repo"
response = run_curl_command(f'curl -X GET -H "X-JFrog-Art-Api: <your-api-key>" -H "Content-Type: application/json" "https://<your-artifactory-domain>/artifactory/api/docker/{repo_name}/v2/_catalog"')
print(response)
```

---

### **Next Steps**
1. Use the `curl` commands to manually verify the presence of Docker repositories, images, and tags.
2. Update the Python script based on the findings from the `curl` commands.
3. If the issue persists, check Artifactory logs or contact your Artifactory administrator.

Let me know if you need further assistance!
