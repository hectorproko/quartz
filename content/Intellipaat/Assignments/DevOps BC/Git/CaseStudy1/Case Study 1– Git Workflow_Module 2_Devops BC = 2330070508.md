
> [!info] Module 2: Case Study - Git
> **Training Problem Statement:** 
> You work as a DevOps Architect in Zendrix Softwares. The company has been struggling to manage their product releases. The releases should happen on the 25th of every month. Suggest a Git workflow architecture for this requirement. Simulate this workflow by creating pseudo code files and branches and upload the same to your GitHub account. 
> 
> As a part of the solution, share the link to your GitHub repository.


To address the release cycle requirement of Zendrix Softwares, the Git Flow model is an excellent match because it has been specifically designed to handle regular releases while also accommodating feature development and hotfixes.


%%## Suggested Git Workflow for Zendrix Softwares:

1. **Master Branch**: Represents the production code. Any code in master is deployable.
2. **Develop Branch**: A branch where features destined for the next release are integrated. Represents the latest delivered development changes for the next release.
3. **Feature Branches**: Branches created to develop new features for the upcoming or a distant future release. These branches only exist until the feature is merged into develop.
4. **Release Branches**: Branches that are created to prepare a new production release. They allow for last-minute dotting of i's and crossing t's. Bug fixes, documentation generation, and other release-oriented tasks should be done here.
5. **Hotfix Branches**: When an undesired situation occurs in a live production version, and it cannot wait for the next release, you branch off a hotfix branch from master.%%

**Zendrix Softwares Git Workflow Simulation**

1. **Initialization**:
- On GitHub, I create a new repository named "ZendrixRepo."
- I clone the repository locally:
```bash
git clone git@github.com:hectorproko/ZendrixRepo.git
cd ZendrixRepo
```

2. **Setting up the Master and Develop Branches**:
- I initialize the repository with a README file:
```bash
echo "Zendrix Software Repository" > README.md
git add README.md
git commit -m "Initialize repository"
```

- I create a develop branch:
```bash
git branch develop
```

3. **Working on a New Feature**:
- I check out a new feature branch from develop:
```bash
git checkout -b feature/new-feature develop
```

- I add code for the new feature:
```bash
echo "Code for new feature" > new-feature.txt 
git add new-feature.txt 
git commit -m "Implement new feature"
```

- I merge the feature branch into develop and delete it:
```bash
git checkout develop 
git merge feature/new-feature 
git branch -d feature/new-feature
```

4. **Preparing a Release**:
- Around the 23rd or 24th, I prepare for the release:
```bash
git checkout -b release/v1.0 develop
```

- I make final fixes and add release notes:
```bash
echo "Release notes for v1.0" > release-notes-v1.0.txt 
git add release-notes-v1.0.txt 
git commit -m "Add release notes for v1.0"
```

5. **Performing the Release**:
- On release day (the 25th), it's time to finalize the changes and bring them to the production environment. The release branch contains the accumulation of work that's ready for release.
- First, I merge the release branch into the master (production) branch. This action moves the changes into the stable production codebase:
```bash
git checkout main 
git merge release/v1.0
```

- After merging, I add a tag to the master branch. Tags in Git are used to capture a point in history, which is of interest. In this case, it signifies a release point. By doing this, anyone can easily pull down a specific release version:
```bash
git tag -a v1.0 -m "Version 1.0"
```

- With the release deployed, it's also a good practice to merge the release branch back into the develop branch. This ensures that the develop branch has all the code that's in production:
```bash
git checkout develop
git merge release/v1.0
```

- Finally, I no longer need the release branch, so I delete it. This cleanup keeps the repository tidy:
```bash
git branch -d release/v1.0
```

6. **Creating a Hotfix**:
- If there's an urgent issue, I create a hotfix:
```bash
git checkout -b hotfix/fix-issue master
```

- I implement the hotfix:
```bash
echo "Hotfix applied" > hotfix.txt 
git add hotfix.txt 
git commit -m "Fix a critical issue"
```

- I merge the hotfix into both master and develop:
```bash
git checkout master 
git merge hotfix/fix-issue 
git checkout develop 
git merge hotfix/fix-issue 
git branch -d hotfix/fix-issue
```

7. **Pushing to GitHub**:
- I link my repository to my GitHub account. Then, I push both the master and develop branches:
```bash
git push origin main 
git push origin develop
```


---

> [!success]
> The link to my [GitHub repository](https://github.com/hectorproko/ZendrixRepo) 

