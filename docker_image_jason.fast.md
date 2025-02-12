Yes, we can modify the script to fetch the EON_ID from JFrog Xray. Here's how you can update the `get_artifact_info` function and add a new `get_eon_id_from_xray` function:

```python
import requests
import logging

def get_eon_id_from_xray(repo_name, artifact_path, headers):
    """Fetch EON ID from JFrog Xray API."""
    xray_url = f"{JFROG_URL}/xray/api/v1/artifacts/{repo_name}/{artifact_path}"
    try:
        response = requests.get(xray_url, headers=headers, verify=False)
        if response.status_code == 200:
            xray_data = response.json()
            eon_id = xray_data.get("eon_id", "N/A")
            if eon_id == "N/A":
                logging.warning(f"EON_ID not found for {artifact_path} in Xray")
            return eon_id
        else:
            logging.warning(f"Failed to fetch EON_ID for {artifact_path} from Xray: {response.text}")
    except requests.RequestException as e:
        logging.warning(f"Failed to fetch EON_ID for {artifact_path} from Xray: {e}")
    return "N/A"

def get_artifact_info(repo_name, artifact, headers):
    """Retrieve artifact details, including EON ID if available."""
    artifact_path = artifact.get("uri", "").lstrip("/")
    if ".jfrog" in artifact_path:
        return None

    url = f"{ARTIFACTORY_API}/{repo_name}/{artifact_path}"
    for _ in range(MAX_RETRIES):
        try:
            response = requests.get(url, headers=headers, verify=False, timeout=10)
            response.raise_for_status()
            data = response.json()
            digest = data.get("checksums", {}).get("sha256")
            if not digest:
                return None

            path_parts = artifact_path.split('/')
            if len(path_parts) < 2:
                logging.warning(f"Unexpected path structure for {artifact_path}")
                return None

            tag = path_parts[-2] if len(path_parts) > 2 else "latest"
            image_name = '/'.join(path_parts[:-2]) if len(path_parts) > 2 else path_parts[0]

            # Fetch EON ID from Xray
            eon_id = get_eon_id_from_xray(repo_name, artifact_path, headers)

            # Ensure the digest is correctly formatted
            formatted_digest = f"sha256:{digest}" if not digest.startswith("sha256:") else digest

            return {
                "Resource_Name": f"{image_name}:{tag}",
                "CSP": "placeholder",
                "Resource_Type": "Container Image",
                "Unique_ID": formatted_digest,
                "EON_ID": eon_id,
                "Digest": formatted_digest,
                "Registry": f"{repo_name}/{image_name}"
            }
        except requests.RequestException as e:
            logging.warning(f"Error fetching {url}: {e}. Retrying...")
        except IndexError as e:
            logging.error(f"IndexError processing {artifact_path}: {e}")
            return None

    logging.error(f"Failed to fetch {url} after {MAX_RETRIES} attempts")
    return None
```

This modification adds a new function `get_eon_id_from_xray` that attempts to fetch the EON_ID from the JFrog Xray API. The `get_artifact_info` function now calls this new function instead of the previous `get_eon_id` function.

Make sure to replace `JFROG_URL` with the actual URL of your JFrog instance. Also, ensure that your API token has the necessary permissions to access the Xray API.

Note that this implementation assumes that the Xray API endpoint and response structure are as described. You may need to adjust the URL or response parsing if the actual API differs from this assumption[1][3].

Citations:
[1] https://docs.brinqa.com/docs/connectors/jfrog-xray/
[2] https://jfrog.com/help/r/jfrog-security-documentation/xray-scan-results
[3] https://help.vulcancyber.com/en/articles/3572045-jfrog-connector
[4] https://jfrog.com/help/r/xray-how-to-find-the-artifactory-id-which-is-required-by-some-of-the-xray-rest-api
[5] https://jfrog.com/help/r/jfrog-artifactory-documentation/view-xray-data-on-artifacts
[6] https://stackoverflow.com/questions/75513400/jfrog-xray-api-to-retrieve-reports
[7] https://jfrog.com/help/r/jfrog-security-documentation/xray-dependencies-scan
[8] https://jfrog.com/help/r/jfrog-rest-apis/trace-artifact-retrieval

---
Answer from Perplexity: pplx.ai/share
