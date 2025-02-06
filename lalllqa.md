To list all repositories in an organization using the Docker Registry HTTP API v2, you can use the following curl command:

```bash
curl -H "Authorization: Bearer ${token}" https://registry-1.docker.io/v2/_catalog
```

This command will return a JSON response containing a list of all repositories in the organization[1][2]. The response will be in the following format:

```json
{
  "repositories": [
    "repo1",
    "repo2",
    ...
  ]
}
```

Before running this command, you'll need to obtain an authentication token. You can get the token using:

```bash
curl -s "https://auth.docker.io/token?service=registry.docker.io&scope=repository:organization/repository:pull" | jq -r .token
```

Replace "organization/repository" with your organization name and a specific repository within it.

If you're using a private registry or a different container registry service, you may need to adjust the URL and authentication method accordingly[3][5].

It's important to note that some registries may limit the number of results returned in a single request. In such cases, you might need to implement pagination to retrieve the complete list of repositories[3].

Citations:
[1] https://distribution.github.io/distribution/spec/api/
[2] https://gist.github.com/windsting/f823dcea5ad460aa1caa6d47b2390aad
[3] https://docs.gitlab.com/ee/api/container_registry.html
[4] https://forums.docker.com/t/list-of-all-repositories-in-registry/27632
[5] https://stackoverflow.com/questions/56193110/how-can-i-use-docker-registry-http-api-v2-to-obtain-a-list-of-all-repositories-i
[6] https://www.reddit.com/r/docker/comments/qgu0ue/help_is_there_a_consistent_api_that_can_list_tags/
[7] https://forums.docker.com/t/does-docker-hub-api-provides-a-solution-for-listing-down-the-organizations-associated-with-my-account/45728
[8] https://stackoverflow.com/questions/31251356/how-to-get-a-list-of-images-on-docker-registry-v2

---
Answer from Perplexity: pplx.ai/share
