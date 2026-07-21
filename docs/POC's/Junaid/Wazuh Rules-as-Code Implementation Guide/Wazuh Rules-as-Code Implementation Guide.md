# Wazuh Rules-as-Code (RaC) Implementation Guide using GitLab CI/CD

## 1. Objective

Implement a Rules-as-Code workflow for Wazuh where:

- Wazuh rules and decoders are stored in GitLab.
- Changes are version controlled.
- GitLab CI validates XML syntax.
- GitLab Runner deploys approved changes automatically.
- Wazuh Manager reloads updated rules.
- Backups are created before deployment.
- Rollback is possible if deployment fails.

---

## 2. Architecture
```

Developer (VS Code)

↓

Git Commit

↓

Git Push

↓

GitLab Repository

↓

GitLab Pipeline

↓

XML Validation

↓

Deploy Script

↓

Copy Rules/Decoders

↓

Fix Ownership

↓

Restart Wazuh Manager

↓

Rules Active

```
---

## 3. Prerequisites

### Infrastructure

| **Component** | **Purpose** |
|---|---|
| Ubuntu VM | Wazuh Host |
| Docker | Wazuh Deployment |
| GitLab Repository | Source Control |
| GitLab Runner | CI/CD Execution |
| VS Code | Rule Development |

### Existing Wazuh Environment

Verify:

```bash
sudo docker ps
```

Expected:
```

single-node-wazuh.manager-1

single-node-wazuh.indexer-1

single-node-wazuh.dashboard-1

```
---

## 4. Identify Wazuh Configuration Volume

Check manager mounts:

```bash
sudo docker inspect single-node-wazuh.manager-1 \
  --format '{{json .Mounts}}'
```

Locate:
```

single-node_wazuh_etc

```
Example path:
```

/var/lib/docker/volumes/single-node_wazuh_etc/_data

```
Verify:

```bash
sudo ls /var/lib/docker/volumes/single-node_wazuh_etc/_data
```

Expected:
```

rules

decoders

lists

shared

client.keys

ossec.conf

```
---

## 5. Create Working Repository

Create repository directory:

```bash
mkdir ~/wazuh-rac
cd ~/wazuh-rac
```

Copy required content:

```bash
rsync -av /var/lib/docker/volumes/single-node_wazuh_etc/_data/rules ~/wazuh-rac/
rsync -av /var/lib/docker/volumes/single-node_wazuh_etc/_data/decoders ~/wazuh-rac/
```

---

## 6. Initialize Git Repository

```bash
cd ~/wazuh-rac
git init
git branch -M main
```

Configure Git:

```bash
git config --global user.name "Your Name"
git config --global user.email "your@email.com"
```

---

## 7. Create .gitignore

Create `.gitignore` with the following content:
```

client.keys

ossec.conf

internal_options.conf

local_internal_options.conf

sslmanager.cert

sslmanager.key

localtime

lists/

rootcheck/

shared/

*.bak

*.deb

deploy.sh

rollback.sh

```
---

## 8. Initial Commit

```bash
git add .
git commit -m "Initial Wazuh RaC baseline"
```

---

## 9. Connect GitLab Repository

```bash
git remote add origin https://gitrepo.ityug.com/vidyayug/devops/wazuh-ruleset.git
```

Push:

```bash
git push -u origin main
```

If the repository already contains commits:

```bash
git pull origin main --allow-unrelated-histories --no-rebase
```

Then:

```bash
git push origin main
```

---

## 10. Create Development Branch

```bash
git checkout -b dev
git push -u origin dev
```

Workflow:
```

Developer

↓

dev branch

↓

Validation

↓

Merge Request

↓

main

↓

Deployment

```
---

## 11. Install GitLab Runner

Install package:

```bash
sudo dpkg -i gitlab-runner_amd64.deb
```

Register runner:

```bash
sudo gitlab-runner register
```

Provide:

- **URL**: `https://gitrepo.ityug.com`
- **Token**: `<Runner Registration Token>`
- **Executor**: `shell`

---

## 12. Fix Shell Runner Issue

Default Ubuntu image may fail due to `.bash_logout`. Rename it:

```bash
sudo mv /home/gitlab-runner/.bash_logout /home/gitlab-runner/.bash_logout.bak
```

Restart:

```bash
sudo systemctl restart gitlab-runner
```

Verify:

```bash
sudo systemctl status gitlab-runner
```

---

## 13. Create Validation Pipeline

File: `.gitlab-ci.yml`

```yaml
stages:
  - validate

validate-rules:
  stage: validate
  tags:
    - wazuh
  script:
    - echo "Validating XML files..."
    - |
      for file in $(find rules decoders -name "*.xml"); do
        echo "Checking $file"
        xmllint --noout "$file" || exit 1
      done
    - echo "XML validation successful"
```

---

## 14. Test Validation

Break XML intentionally:

```xml
<broken>
```

Commit:

```bash
git add .
git commit -m "Validation test"
git push origin dev
```

Expected:
```

Pipeline Failed

```
Fix XML, then commit and push again:

```bash
git commit
git push
```

Expected:
```

Pipeline Passed

```
---

## 15. Create Deployment Script

File: `deploy.sh`

```bash
#!/bin/bash
set -e

echo "=== Starting Wazuh Rules Deployment ==="

BACKUP_DIR=/home/shekar/wazuh-backups/$(date +%F-%H%M%S)
mkdir -p "$BACKUP_DIR"

cp -r /var/lib/docker/volumes/single-node_wazuh_etc/_data/rules "$BACKUP_DIR/"
cp -r /var/lib/docker/volumes/single-node_wazuh_etc/_data/decoders "$BACKUP_DIR/"

rsync -av rules/ /var/lib/docker/volumes/single-node_wazuh_etc/_data/rules/
rsync -av decoders/ /var/lib/docker/volumes/single-node_wazuh_etc/_data/decoders/

chown -R 999:999 /var/lib/docker/volumes/single-node_wazuh_etc/_data/rules
chown -R 999:999 /var/lib/docker/volumes/single-node_wazuh_etc/_data/decoders

docker restart single-node-wazuh.manager-1

echo "Deployment completed successfully."
```

Make executable:

```bash
chmod +x deploy.sh
```

---

## 16. Allow Runner Deployment

Add sudo permissions:

```bash
sudo visudo
```

Add:
```

gitlab-runner ALL=(ALL) NOPASSWD: /home/shekar/wazuh-rac/deploy.sh

```
---

## 17. Deployment Pipeline

Add the deploy job to `.gitlab-ci.yml`:

```yaml
stages:
  - validate
  - deploy

deploy-rules:
  stage: deploy
  tags:
    - wazuh
  script:
    - sudo /home/shekar/wazuh-rac/deploy.sh
  only:
    - main
```

---

## 18. Recommended Production Workflow

**Development:**
```

feature branch

↓

dev

↓

validation only

```
**Production:**
```

Merge Request

↓

main

↓

validation

↓

deployment

```
!!! warning
    Never deploy directly from `dev`.

---

## 19. Rollback Strategy

Backups are stored in:
```

/home/shekar/wazuh-backups/

```
Each deployment creates a timestamped snapshot containing:
```

rules/

decoders/

```
Rollback restores the latest backup and restarts Wazuh.

---

## 20. Daily Developer Workflow

Open the repository in VS Code. Edit:
```

rules/*.xml

decoders/*.xml

```
Then:

```bash
git add .
git commit -m "Rule update"
git push origin dev
```

Create a Merge Request: `dev → main`

Pipeline validates. After merge:
```

Deploy

↓

Restart Wazuh

↓

Rules Active

```
!!! note
    No manual copying of files is required.
