# Project Setup: Automated Submodule Update and Configuration Handling

## Overview
This project involves setting up a system where changes in the frontend and backend repositories are automatically reflected in a fullstack repository. Additionally, a configuration file in the fullstack repository is accessed by the frontend and backend repositories.

## Repository Structure

### Repositories Created
- [frontend-repo](https://github.com/hadigghazi/frontend-repo.git)
- [backend-repo](https://github.com/hadigghazi/backend-repo.git)
- [fullstack-repo](https://github.com/hadigghazi/fullstack-repo.git)

### Cloning Repositories
All repositories were cloned locally.

## Initial Setup

### Adding Python Files
`client.py` was added to `frontend-repo` and `server.py` was added to `backend-repo`.

```python
import yaml

class Client:
    def __init__(self, config_file):
        with open(config_file, 'r') as file:
            self.config = yaml.safe_load(file)
        self.server_ip = self.config['ServerIPAddress']
        self.run_localhost = self.config['run_localhost']

    def check_localhost(self):
        if self.run_localhost:
            print("Error: run_localhost is set to True")

    def get_server_ip(self):
        return self.server_ip

if __name__ == "__main__":
    client = Client('../config.yaml')
    client.check_localhost()
    print(client.get_server_ip())
```

These files were then pushed to GitHub.

### Configuring Fullstack Repository
- Added the other repositories as submodules using the `git submodule add` command.
- Created a `config.yaml` file with the following content:
  ```yaml
  ServerIPAddress: 127.0.0.1
  run_localhost: true
  ```
- Pushed these changes to GitHub.

## Automating Submodule Updates

### Attempted Cron Job
A cron job was created to run every hour to push changes from `frontend-repo` and `backend-repo` to `fullstack-repo`, but it was unsuccessful.

### Post-Receive Hook in Fullstack Repository
Added a `post-receive` script to the `fullstack-repo` at `.git/hooks/post-receive` with the following content:

```bash
#!/bin/bash -e

UPDATED_BRANCHES="^(main|develop)$"
UPDATED_REPOS="^(https:\/\/github.com\/hadigghazi\/frontend-repo.git|https:\/\/github.com\/hadigghazi\/backend-repo.git)$"

# Determine what branch gets modified
read REV_OLD REV_NEW FULL_REF
BRANCH=${FULL_REF##refs/heads/}

if [[ "${BRANCH}" =~ ${UPDATED_BRANCHES} ]] && [[ "${GL_REPO}" =~ ${UPDATED_REPOS} ]];
then
    # Determine the name of the branch in the super repository
    SUPERBRANCH=$FULL_REF
    SUBMODULE_NAME=${GL_REPO##https://github.com/hadigghazi/}
    # Clean the submodule repo related environment
    unset $(git rev-parse --local-env-vars)
    # Move to the super repository
    cd C:/Users/mycom/Downloads/fullstack-repo  # Updated path

    echo "Automatically updating the '$SUBMODULE_NAME' reference in the super repository..."
    # Modify the index - replace the submodule reference hash
    git ls-tree $SUPERBRANCH | \
        sed "s/\([0-9]*\) commit \([0-9a-f]*\)\t$SUBMODULE_NAME/\1 commit $REV_NEW\t$SUBMODULE_NAME/g" | \
        git update-index --index-info

    # Write the tree containing the modified index
    TREE_NEW=$(git write-tree)
    COMMIT_OLD=$(git show-ref --hash $SUPERBRANCH)

    # Write the tree to a new commit and use the current commit as its parent
    COMMIT_NEW=$(echo "Auto-update submodule: $SUBMODULE_NAME" | git commit-tree $TREE_NEW -p $COMMIT_OLD)

    # Update the branch reference
    git update-ref $SUPERBRANCH $COMMIT_NEW
fi
```

This script allowed updates from `frontend-repo` and `backend-repo` to be reflected in the online `fullstack-repo` after running `git submodule update --remote`, committing, and pushing the changes to GitHub.

### Automating Submodule Update on Push
A script was added to the `post-receive` file in both `frontend-repo` and `backend-repo` with the following content:

```bash
#!/bin/bash

# Update submodule in fullstack-repo
cd C:/Users/mycom/Downloads/fullstack-repo
git submodule update --remote --merge backend

# Commit and push changes in fullstack-repo
git add backend
git commit -m "Update submodule: backend-repo"
git push origin main
```

## Tutorial: Setting Up Automated Submodule Updates

### Step 1: Create Repositories
1. Create three repositories on GitHub: `frontend-repo`, `backend-repo`, and `fullstack-repo`.
2. Clone these repositories to your local machine.

### Step 2: Add Python Files
1. Create `client.py` in `frontend-repo` and `server.py` in `backend-repo`.
2. Push these files to their respective repositories.

### Step 3: Add Submodules
1. In `fullstack-repo`, add `frontend-repo` and `backend-repo` as submodules:
   ```bash
   git submodule add https://github.com/yourusername/frontend-repo.git
   git submodule add https://github.com/yourusername/backend-repo.git
   ```

### Step 4: Create Configuration File
1. In `fullstack-repo`, create a `config.yaml` file with the following content:
   ```yaml
   ServerIPAddress: 127.0.0.1
   run_localhost: true
   ```
2. Push the changes to GitHub.

### Step 5: Setup Post-Receive Hooks
1. In `fullstack-repo`, add the following script to `.git/hooks/post-receive`:

   ```bash
   #!/bin/bash -e
   UPDATED_BRANCHES="^(main|develop)$"
   UPDATED_REPOS="^(https:\/\/github.com\/hadigghazi\/frontend-repo.git|https:\/\/github.com\/hadigghazi\/backend-repo.git)$"
   read REV_OLD REV_NEW FULL_REF
   BRANCH=${FULL_REF##refs/heads/}
   if [[ "${BRANCH}" =~ ${UPDATED_BRANCHES} ]] && [[ "${GL_REPO}" =~ ${UPDATED_REPOS} ]];
   then
       SUPERBRANCH=$FULL_REF
       SUBMODULE_NAME=${GL_REPO##https://github.com/hadigghazi/}
       unset $(git rev-parse --local-env-vars)
       cd C:/Users/mycom/Downloads/fullstack-repo
       echo "Automatically updating the '$SUBMODULE_NAME' reference in the super repository..."
       git ls-tree $SUPERBRANCH | \
           sed "s/\([0-9]*\) commit \([0-9a-f]*\)\t$SUBMODULE_NAME/\1 commit $REV_NEW\t$SUBMODULE_NAME/g" | \
           git update-index --index-info
       TREE_NEW=$(git write-tree)
       COMMIT_OLD=$(git show-ref --hash $SUPERBRANCH)
       COMMIT_NEW=$(echo "Auto-update submodule: $SUBMODULE_NAME" | git commit-tree $TREE_NEW -p $COMMIT_OLD)
       git update-ref $SUPERBRANCH $COMMIT_NEW
   fi
   ```

2. In both `frontend-repo` and `backend-repo`, add the following script to `.git/hooks/post-receive`:

   ```bash
   #!/bin/bash
   echo "Executing post-receive hook for repository: $GL_REPO"
   echo "SUPERREPO_DIR is set to: $SUPERREPO_DIR"
   cd C:/Users/mycom/Downloads/fullstack-repo
   git submodule update --remote --merge backend
   git add backend
   git commit -m "Update submodule: backend-repo"
   git push origin main
   echo "Post-receive hook executed successfully."
   ```

### Step 6: Test the Setup
1. Push changes to `frontend-repo` and `backend-repo`.
2. Verify that the changes are reflected in `fullstack-repo`.

## Graphical Representation
![Graph](https://github.com/hadigghazi/fullstack-repo/blob/main/graph.jpg)

## Current Status
- The `config.yaml` file is accessed by both `frontend-repo` and `backend-repo`.
- Changes in `frontend-repo` and `backend-repo` can be reflected in the online `fullstack-repo` using a simple command.
- Automated updates upon pushing to `frontend-repo` and `backend-repo` are under testing and refinement.

## Conclusion
This setup ensures that changes in the `frontend` and `backend` repositories are automatically propagated to the `fullstack` repository, maintaining synchronization and central configuration management. Further refinement and testing of the automation scripts will ensure seamless updates.
