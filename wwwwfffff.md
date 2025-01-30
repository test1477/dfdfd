To update the "Get Raw URL(s)" step for Octopus CLI 2.14.0, you need to make a few changes. Here's the updated version:

```yaml
- name: Get Raw URL(s) and Create Release
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    OCTO_API_KEY: ${{ secrets.OCTO_TIBCODEPLOY_API_KEY }}
  run: |
    files=$(git diff-tree --no-commit-id --diff-filter=ACMRT --name-only -r ${{ steps.sha.outputs.sha_short }})
    echo "$files"
    IFS=$'\n'
    for file in $files; do
      if [[ $file != *.car ]]; then
        continue
      fi
      filename="${file//\//%20}"
      raw_url="https://raw.githubusercontent.com/Eaton-Vance-Corp/Tibco/${{ github.sha }}/$file"
      echo "Raw URL: $raw_url"
      
      octopus release create \
        --project "Tibco" \
        --deployTo UAT \
        --space Spaces-1 \
        --variable "FileName:$filename" \
        --variable "Git_Author:${{ steps.author.outputs.email }}" \
        --releaseNotes "${{ steps.sha.outputs.sha_short }}" \
        --server "https://ev-octo-server.octopus.app" \
        --apiKey "$OCTO_API_KEY"
    done
```

Key changes:

1. The raw URL is now constructed directly using the repository name, commit SHA, and file path, eliminating the need for a separate API call[1].

2. The `octo` command has been replaced with `octopus`, and the command structure has been updated to match the new CLI syntax[3][5].

3. The `create-release` command now uses hyphens instead of equals signs for its arguments[7].

4. The `--deployTo` flag is used instead of `--deployto` for consistency with the new CLI syntax[7].

5. The `--releaseNotes` flag is used instead of `--releasenotes`[7].

6. Environment variables are used for sensitive information like `GITHUB_TOKEN` and `OCTO_API_KEY`[2].

These changes should resolve the issues with getting the raw URL and create a release using the updated Octopus CLI 2.14.0[5].

Citations:
[1] https://github.com/orgs/community/discussions/22537
[2] https://github.com/OctopusDeploy/install-octopus-cli-action/issues/450
[3] https://github.com/OctopusDeploy/cli
[4] https://git-scm.com/docs/user-manual
[5] https://github.com/OctopusDeploy/cli/releases
[6] https://github.com/OctopusDeploy/install-octopus-cli-action/blob/main/src/octopus-cli.ts
[7] https://octopus.com/docs/octopus-rest-api/octopus-cli
[8] https://formulae.brew.sh/formula/

---
Answer from Perplexity: pplx.ai/share
