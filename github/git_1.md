**GitHub & GitHub Actions Documentation**

**Q1)What is GitHub Actions, and why do we use it? What is GitHub Actions, and why do we use it?**

Ans-\>GitHub Actions is a CI/CD tool provided by GitHub that helps automate software workflows.  
It is mainly used to build, test, and deploy applications automatically whenever code changes are pushed to a repository.  
We use it to reduce manual work, improve consistency, and enable continuous integration and continuous deployment.

**2)What is a workflow in GitHub Actions?**

A workflow in GitHub Actions is a YAML file that defines an automated process.  
It is stored inside the .github/workflows directory of a repository.  
A workflow contains jobs and steps that run when specific events occur, like push or pull request.

<img src="./media/image1.png" style="width:5.12572in;height:1.21892in" />

**3)Trigger Configuration:**

This workflow runs automatically when code is pushed to the dev or master branches

<img src="./media/image2.png" style="width:2.08362in;height:1.23976in" />

**4)What are jobs and steps in GitHub Actions?**

Jobs are a collection of steps that run on the same runner (machine).  
Each job can run independently or in sequence with other jobs.

Steps are individual tasks inside a job, such as running a command, installing dependencies, or using an action.

<img src="./media/image3.png" style="width:3.16711in;height:2.18781in" />

**5) What is a runner in GitHub Actions**

A runner is a machine (virtual server) that executes the jobs in a GitHub Actions workflow.  
It creates a temporary environment, checks out the repository code, and runs all the steps like build, test, and deployment.  
GitHub provides hosted runners (like Ubuntu), and we can also use self-hosted runners.

- **runs-on: ubuntu-latest**

**6) Uses**

The uses keyword is used to reuse existing actions or workflows in GitHub Actions.

- **steps:**

  - **uses: actions/checkout@v4**

**7)with**

The **with** keyword is used to provide input parameters to an action, allowing customization of its behavior.

<img src="./media/image4.png" style="width:3.16627in;height:2.18723in" />

**8) upload-artifact**

This step uses actions/upload-artifact to store build outputs or files generated during the workflow.

<img src="./media/image5.png" style="width:3.22962in;height:1.25017in" />

**9)Build**

The Build job compiles the application and prepares it for testing, scanning, and deployment.

<img src="./media/image6.png" style="width:3.24003in;height:3.40673in" />

**10) Trivy Security Scan**

**🔹 Description**

- Trivy is used to scan the application for security vulnerabilities in dependencies and source code.

**🔹 Configuration**

<img src="./media/image7.png" style="width:3.7401in;height:1.31268in" />

**🔹 Behavior**

- Scans the project files for vulnerabilities

- Focuses on HIGH and CRITICAL severity issues

- If any such vulnerabilities are found → ❌ pipeline fails immediately

- Prevents insecure code from moving to next stages (artifact, deploy)

**🔹 Handling Vulnerabilities**

- Developers review the Trivy report

- Identify affected dependencies

- Fix by:

  - Updating packages

  - Removing unused libraries

  - Applying recommended fixes

Large Trivy reports are analyzed by filtering high and critical vulnerabilities and focusing on fixable issues to efficiently resolve security risks.

**10)** **SonarQube**

is used to analyze the code for code quality issues, bugs, and security vulnerabilities

<img src="./media/image8.png" style="width:7.26806in;height:3.52847in" />

**SONAR_TOKEN:** is a secure authentication token used to connect GitHub Actions with SonarQube.

**Go to SonarQube / SonarCloud**

Generate token from:

- Profile → Security → Generate Token

**Add it in GitHub:**

- Repository → Settings → Secrets → Actions

- Add SONAR_TOKEN

- 

**The project repository is linked to SonarQube to enable code quality analysis.**

Add your repository to SonarQube / SonarCloud

Open the project dashboard

Go to **Project Information**

Copy the following details:

- sonar.projectKey

- sonar.organization (for SonarCloud)

**11)Deployment Approval**

> Use GitHub Environments with required reviewers to manually approve deployments before production.

- Go to Settings → Environments → New environment

- Name it (e.g., production)

- Enable Required reviewers and add approvers

<img src="./media/image9.png" style="width:4.36519in;height:1.48979in" />

**12)GitHub Secret Management**

GitHub Secrets – Secure storage of sensitive credentials for CI/CD workflow

Keep sensitive information like:

- API tokens (SONAR_TOKEN)

- Deployment keys (DEPLOY_KEY)

- Environment passwords

- Go to your **GitHub repository** → **Settings**

- Click **Secrets and variables → Actions → New repository secret**

- Give a **name** (e.g., SONAR_TOKEN)

- Paste the **secret value**

- Click **Add secret**

**Deploy Job**

- Deploy the production-ready build to a remote server using SSH.

- Automatically updates the server content and restarts services (e.g., Nginx).

<img src="./media/image10.png" style="width:7.26806in;height:9.34583in" />

- **Needs:** Job will start only after these dependent jobs are complete.

- **environment:** in GitHub can have **required reviewers**

- **Download Build Artifact Step**

- **Determine environment & secrets**

- **Setup SSH**

- **Deploy to server**

**Github-CMD**

**1)Git Configuration**

| **Command**                                      | **Description**                     |
|--------------------------------------------------|-------------------------------------|
| git config --global user.name "Your Name"        | Set your Git username.              |
| git config --global user.email "you@example.com" | Set your Git email.                 |
| git config --global core.editor "nano"           | Set default editor for Git commits. |
| git config --list                                | List all Git configurations.        |

**2. Creating and Cloning Repositories**

| **Command**            | **Description**                                             |
|------------------------|-------------------------------------------------------------|
| git init               | Initialize a new Git repository in the current directory.   |
| git clone \<repo_url\> | Clone an existing repository from GitHub or another remote. |

**3)Checking Status and Making Commits**

| **Command**                | **Description**                               |
|----------------------------|-----------------------------------------------|
| git status                 | Check the status of modified/untracked files. |
| git add \<file\>           | Stage a specific file for commit.             |
| git add .                  | Stage all changes in the repository.          |
| git commit -m "Message"    | Commit staged changes with a message.         |
| git commit --amend         | Modify the last commit message or content.    |
| git push origin \<branch\> | Push commits to the remote repository.        |
| git pull origin \<branch\> | Pull latest changes from the remote branch.   |

**4) Branching and Merging**

| **Command**                                   | **Description**                               |
|-----------------------------------------------|-----------------------------------------------|
| git branch                                    | List all branches in the repository.          |
| git branch \<branch_name\>                    | Create a new branch.                          |
| git checkout \<branch\>                       | Switch to a different branch.                 |
| git checkout -b \<branch\>                    | Create a new branch and switch to it.         |
| git merge \<branch\>                          | Merge another branch into the current branch. |
| git branch -d \<branch\>                      | Delete a branch locally.                      |
| git push origin --delete \<branch\>           | Delete a branch remotely.                     |
| git branch -m old-branch-name new-branch-name | Rename a branch locally.                      |

**5. Rebasing and Cherry-Picking**

| **Command**                | **Description**                                         |
|----------------------------|---------------------------------------------------------|
| git rebase \<branch\>      | Move your commits on top of another branch.             |
| git rebase -i \<commit\>   | Interactive rebase to edit, squash, or reorder commits. |
| git cherry-pick \<commit\> | Apply a specific commit from another branch.            |

**6. Remote Repository Management**

| **Command**                   | **Description**                            |
|-------------------------------|--------------------------------------------|
| git remote -v                 | Show remote repository URLs.               |
| git remote add origin \<url\> | Add a remote repository.                   |
| git fetch origin              | Fetch changes from remote without merging. |
| git pull origin \<branch\>    | Fetch and merge changes from remote.       |
| git push origin \<branch\>    | Push commits to the remote branch.         |

**7. Undoing Changes**

| **Command**                 | **Description**                                      |
|-----------------------------|------------------------------------------------------|
| git checkout -- \<file\>    | Discard changes in the working directory for a file. |
| git reset \<file\>          | Unstage a file from the staging area.                |
| git reset --soft \<commit\> | Undo commits but keep changes staged.                |
| git reset --hard \<commit\> | Undo commits and discard all changes.                |
| git revert \<commit\>       | Create a new commit to undo a previous commit.       |

**8. Stashing Changes**

| **Command**                 | **Description**                                                |
|-----------------------------|----------------------------------------------------------------|
| git stash                   | Save uncommitted changes temporarily.                          |
| git stash list              | Show all stashed changes.                                      |
| git stash apply             | Apply the last stashed changes without removing it from stash. |
| git stash pop               | Apply and remove the last stashed changes.                     |
| git stash branch \<branch\> | Create a new branch from a stash.                              |

**9. Viewing Commit History and Differences**

| **Command**                     | **Description**                                  |
|---------------------------------|--------------------------------------------------|
| git log                         | Show full commit history.                        |
| git log --oneline               | Show a short summary of commits.                 |
| git log --graph --oneline --all | Show a visual graph of all commits and branches. |
| git diff                        | Show changes not staged for commit.              |
| git diff --staged               | Show changes staged for commit.                  |

**10. Tags**

| **Command**                 | **Description**                      |
|-----------------------------|--------------------------------------|
| git tag                     | List all tags in the repository.     |
| git tag \<tagname\>         | Create a new tag.                    |
| git push origin \<tagname\> | Push a tag to the remote repository. |

| **Command**         | **Description**                                    |
|---------------------|----------------------------------------------------|
| git blame \<file\>  | Show which user last modified each line in a file. |
| git show \<commit\> | Show details of a specific commit.                 |
| git clean -f        | Remove untracked files from the working directory. |
| git shortlog        | Summarize commits per author.                      |

**✅ Problem**

- The remote repository (GitHub) has **new updates**

- You also have **your own local changes**

- You want to **combine them WITHOUT using rebase**

1.Commit your local changes

- git add .

- git commit -m "my changes"

2\. Pull latest changes from remote

git pull origin master

- Downloads latest changes from GitHub

- **Merges** them with your local commits

3\. If conflicts happen

- Git will show a **merge conflict**

- Open the files and fix conflicts manually

Then

- git add .

- git commit -m "resolved conflict"

- git push origin master
