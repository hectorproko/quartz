> [!info] Module 2: Git Assignment - 5
> 
> **Tasks To Be Performed:** 
> 1. Create a Git Flow workflow architecture on Git 
> 2. Create all the required branches 
> 3. Starting from the feature branch, push the branch to the master, following the architecture 
> 4. Push a urgent.txt on master using hotfix

First, I'll create a Git working directory and initialize it:
```bash
mkdir GitAssignment5
cd GitAssignment5
git init
```

**Setting Up the Git Flow Architecture**

I'll set up my main branches: master and develop. I'll start with a simple README file:
```bash
echo "Initial content" > README.md
git add README.md
git commit -m "Initial commit"
git branch develop
```

Now, I'll begin working on my feature from the `develop` branch:
```bash
git checkout -b feature/feature1 develop
```

After adding some content for this feature, I'll stage and commit the changes:
```bash
echo "Content for feature1" > feature1.txt
git add feature1.txt
git commit -m "Add feature1.txt"
```

It's time to merge my feature back into the `develop` branch:
```bash
git checkout develop
git merge feature/feature1
```

I can also decide to delete the feature branch after merging:
```bash
git branch -d feature/feature1
```


I'm now preparing for a release, so I'll create a `release` branch from `develop`:
```bash
git checkout -b release/release1 develop
```

After finalizing everything for this release, I'll merge the `release` branch into `master`:
```bash
git checkout master
git merge release/release1
```

Once the merging is done, I can choose to delete the release branch:
```bash
git branch -d release/release1
```

Lastly, there's an urgent change I need to make directly on the `master` branch:
```bash
echo "Urgent content" > urgent.txt
git add urgent.txt
git commit -m "Add urgent.txt due to a hotfix"
```

In this demonstration, we showcased a widely-adopted Git architecture known as the Git Flow model. The essence of this model is the creation and management of various 'feature' branches, each dedicated to individual enhancements or functionalities. These branches are periodically merged into a central 'develop' branch, serving as an accumulation point for the upcoming release. 

Once ready for a production release, changes from the 'develop' branch are merged into a 'release' branch for final refinements and testing. Ultimately, this 'release' branch gets integrated with the 'master', reflecting the production state. This structured approach ensures a systematic and organized workflow, promoting continuous integration while maintaining the stability of the main production code.









%%
> [!question]- why "feature/feature1" format
> **Organization and Clarity**: By using prefixes like `feature/`, `release/`, or `hotfix/`, it's immediately clear what the purpose of a branch is. This becomes especially valuable as the number of branches grows in a repository.
> 
> Absolutely, you're right. The naming convention `feature/feature1`, `feature/feature2` may seem redundant. The convention of using a prefix followed by a slash and then a descriptor (like `feature/`) is more beneficial when the descriptor provides additional context. For example:
> 
> 1. **Descriptive Names**: Consider situations like `feature/user-login`, `feature/payment-gateway`, `hotfix/password-reset-bug`, etc. Here, the prefix gives a category, and the

%%