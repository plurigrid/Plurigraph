[[Technology]]

# [[Technology/Plurigraph]]
https://github.com/Plurigrid/knowledge (this repository)
## Prerequisites
- [Obsidian.md](https://obsidian.md) installed locally
- GitHub configured (see above)

## Prerequisites
- GitHub.com account
- [SSH key added to GH](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account) + [2FA enabled](https://docs.github.com/en/authentication/securing-your-account-with-two-factor-authentication-2fa)
### SSH key
- SSH, if known, can be added in GH Settings
- If not, consult [SSH Generate Guide](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent) 
- clone `knowledge` repository using the SSH URI
- open directory as a Vault in Obsidian
- add yourself to the people 
- For further troubleshooting, use [Warp](https://docs.warp.dev/features/ai-command-search) for AI-aided controls
...

### When ready to publish changes
Command Palette
`Obsidian Git: Commit all changes`
`Obsidian Git: Push`