You're right, and that's a good observation. If the EON_ID is associated with the image as a whole, rather than individual layers, we can optimize the script to fetch the EON_ID once for each image and apply it to all related artifacts. Here's an updated version of the script that implements this logic:

```python
import requests
import csv
import os
import logging
import argparse
from datetime import datetime
from tqdm import tqdm
from collections import defaultdict

# Logging setup
logging.basicConfig(level=logging.DEBUG, format="%(asctime)s - %(levelname)s - %(message)s")

# Constants (Replace with your JFrog URL)
JFROG_URL = "https://your-jfrog-instance/artifactory"
ARTIFACTORY_API = f"{JFROG_URL}/api/storage"
REPOSITORIES_API = f"{JFROG_URL}/api/repositories"
BUILDS_API = f"{JFROG_URL}/api/build"
SEARCH_API = f"{JFROG_URL}/api/search/aql"
MAX_RETRIES = 3

# ... [keep all the existing functions] ...

def process_repository(repo_name, headers):
    logging.info(f"Processing repository: {repo_name}")
    
    aql_query = f"""items.find(
        {{
            "repo": "{repo_name}",
            "$or": [
                {{"name": {{"$match": "*.tar.gz"}}}},
                {{"name": "manifest.json"}}
            ]
        }}
    ).include("repo", "path", "name")"""
    
    try:
        response = requests.post(SEARCH_API, data=aql_query, headers=headers, verify=False)
        response.raise_for_status()
        artifacts = response.json().get("results", [])
    except requests.RequestException as e:
        logging.error(f"Error fetching artifacts for {repo_name}: {e}")
        return []

    logging.info(f"Found {len(artifacts)} artifacts in {repo_name}")

    # Group artifacts by image
    image_artifacts = defaultdict(list)
    for artifact in artifacts:
        path_parts = artifact.get("path", "").split('/')
        if len(path_parts) >= 2:
            image_name = '/'.join(path_parts[:-1])
            image_artifacts[image_name].append(artifact)

    repo_details = []
    for image_name, image_artifacts_list in tqdm(image_artifacts.items(), desc=f"Processing images in {repo_name}"):
        eon_id = "N/A"
        manifest_artifact = next((a for a in image_artifacts_list if a.get("name") == "manifest.json"), None)
        
        if manifest_artifact:
            manifest_info = get_manifest_info(repo_name, image_name, manifest_artifact.get("path", "").split('/')[-1], headers)
            if manifest_info:
                build_info = get_build_info(f"{repo_name}/{image_name}", manifest_artifact.get("path", "").split('/')[-1], headers)
                if build_info:
                    eon_id = get_eon_id_from_build_info(build_info)

        for artifact in image_artifacts_list:
            result = get_artifact_info(repo_name, artifact, headers, eon_id)
            if result:
                repo_details.append(result)

    logging.info(f"Processed {len(repo_details)} valid artifacts in {repo_name}")
    return repo_details

def get_artifact_info(repo_name, artifact, headers, eon_id="N/A"):
    path_parts = artifact.get("path", "").split('/')
    if len(path_parts) < 2:
        logging.warning(f"Unexpected path structure for {artifact}")
        return None

    tag = path_parts[-1]
    image_name = '/'.join(path_parts[:-1])

    if artifact.get("name") == "manifest.json":
        manifest_info = get_manifest_info(repo_name, image_name, tag, headers)
        if not manifest_info:
            logging.warning(f"No manifest found for {image_name}:{tag}")
            return None
        digest = manifest_info.get("actual_sha1", "")
    else:
        digest = artifact.get("actual_sha1", "")

    if not digest:
        logging.warning(f"No digest found for {image_name}:{tag}")
        return None

    formatted_digest = f"sha256:{digest}"

    return {
        "Resource_Name": f"{image_name}:{tag}",
        "CSP": "placeholder",
        "Resource_Type": "Container Image",
        "Unique_ID": formatted_digest,
        "EON_ID": eon_id,
        "Digest": formatted_digest,
        "Registry": f"{repo_name}/{image_name}"
    }

# ... [keep the main function and other parts unchanged] ...
```

This updated script does the following:

1. Groups artifacts by image name in the `process_repository` function.
2. Fetches the EON_ID once per image using the manifest.json file.
3. Applies the same EON_ID to all artifacts (layers) belonging to that image.
4. Processes both manifest.json and other artifacts (like layer tarballs) with the same EON_ID.

This approach should ensure that all artifacts related to a single Docker image have the same EON_ID, regardless of whether they are manifest.json files or individual layer tarballs.

---
Answer from Perplexity: pplx.ai/share
