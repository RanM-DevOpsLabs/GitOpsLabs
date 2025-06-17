# Multi-Account Git Configuration

This setup allows you to automatically use the correct Git user configuration and SSH keys based on the directory where your repository is located.

## Directory Structure

```
~/git_accounts/
├── personal/
│   └── github/          # Personal GitHub repositories
├── work/
│   ├── github/          # Work GitHub repositories
│   └── gitlab/          # Work GitLab repositories
```

## How It Works

- **Conditional Includes**: Git automatically applies different configurations based on directory location
- **SSH Key Management**: Each account type uses its own SSH key
- **URL Rewriting**: SSH host aliases ensure the correct key is used for each service

## Usage

### Clone Personal GitHub Repository
```bash
cd ~/git_accounts/personal/github
git clone github-personal:username/repo.git
# OR using the automatic URL rewriting:
git clone git@github.com:username/repo.git
```

### Clone Work GitHub Repository
```bash
cd ~/git_accounts/work/github
git clone github-work:company/repo.git
```

### Clone Work GitLab Repository
```bash
cd ~/git_accounts/work/gitlab
git clone gitlab-work:company/repo.git
```

## SSH Host Aliases

- `github-personal`: Personal GitHub account
- `github-work`: Work GitHub account  
- `gitlab-work`: Work GitLab account

## Configuration Files

- `~/.gitconfig`: Main configuration with conditional includes
- `~/.gitconfig-personal`: Personal account settings
- `~/.gitconfig-work-github`: Work GitHub account settings
- `~/.gitconfig-work-gitlab`: Work GitLab account settings
- `~/.ssh/config`: SSH key management

## Setup Steps Completed

✅ Created directory structure under `~/git_accounts/`  
✅ Updated SSH configuration with host aliases  
✅ Created separate git configurations for each account  
✅ Set up conditional includes in global git config  

## Next Steps

1. **Update work email addresses** in:
   - `~/.gitconfig-work-github`
   - `~/.gitconfig-work-gitlab`

2. **Test the configuration**:
   ```bash
   # Test in personal directory
   cd ~/git_accounts/personal/github
   git config user.email  # Should show: markovich.ran@gmail.com
   
   # Test in work directory
   cd ~/git_accounts/work/github
   git config user.email  # Should show: your work email
   ```

3. **Move existing repositories** to the appropriate directories

## Troubleshooting

- If you get permission denied errors, ensure your SSH keys are added to the respective GitHub/GitLab accounts
- Run `ssh -T github-personal` / `ssh -T github-work` / `ssh -T gitlab-work` to test SSH connections
- Use `git config --show-origin user.email` to see which config file is being used 