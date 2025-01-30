Using the approach you mentioned is a reasonable choice, especially if file updates are infrequent (once every 1-2 months). Here's how you can implement it in your workflow:

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
      filename="${file// /%20}"
      raw_url=$(curl -sS -L -X GET "https://api.github.com/repos/Eaton-Vance-Corp/Tibco/contents/$filename" \
        -H "Authorization: token $GITHUB_TOKEN" \
        -H "Accept: application/vnd.github.v3+json")
      download_url=$(echo "$raw_url" | jq -r '.download_url')
      echo "Raw URL: $download_url"
      
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

This method fetches the raw URL for each file using the GitHub API, which is suitable for repositories with infrequent updates[1].

Citations:
[1] https://api.github.com/repos/Eaton-Vance-Corp/Tibco/contents/

---
Answer from Perplexity: pplx.ai/share
