
> [!info] Module 2: Case Study - Resolving Merge Conflicts
> **Training Problem Statement:** 
> You work for Zendrix Software & Co. You have been assigned the task of updating the master branch of their Git repository with all the features from the feature branches. 
> 
> Following is the GitHub account: 
> https://github.com/devops-intellipaat/merge-conflict.git 
> 
> Consider:
> - Feature1 branch to be a public branch
> - Feature2 branch to be a private branch 
>
>The company relies on a monolithic architecture and for now all the code resides in one file “main.c”. 
>
>The respective features have been added in the feature branches for main.c. Meanwhile, a security patch was made to the master branch, and now feature1 and feature2 branches are behind from master by 1 commit. 
>
>**Tasks To Be Performed:** 
>1. Update Feature1 and Feature2 branch with the Security Patch 
>2. Apply changes of Feature1 and Feature2 branches on master 
>3. Finally push all the branches to GitHub 
> 
> For solving this, please fork the repository to your GitHub account and then work. As a solution, please submit your GitHub’s repository link.


### Step 1: Fork the Repository

1. I navigate to [https://github.com/devops-intellipaat/merge-conflict.git](https://github.com/devops-intellipaat/merge-conflict.git).
2. I click on the "Fork" button at the top right of the page to fork the repository to my GitHub account.

### Step 2: Clone the Forked Repository

3. Once forked, I navigate to the forked repository in my GitHub account.
4. I click the "Code" button and copy the repository URL.
5. On my local machine, I run the following command to clone the repository:
```
git clone git@github.com:hectorproko/merge-conflict.git
```


### Step 3: Update Feature1 and Feature2 with the Security Patch from Master

6. I change to the `feature1` branch:
```
git checkout feature1
```

7. I pull the changes from the master branch to get the security patch:
```
git merge master
```
<br>![[Pasted image 20231030120431.png]]
8. If any merge conflicts arise, I resolve them manually in the `main.c` file, save the changes, then stage and commit the resolved file:

Open text editor
<br>![[Pasted image 20231030122332.png]]

<br>![[Pasted image 20231030122308.png]]
<br>![[Pasted image 20231030122813.png]]

```
git add main.c 
git commit -m "Resolved merge conflict and integrated security patch in feature1"
```

9. I repeat the above steps for the `feature2` branch:
```
git checkout feature2 
git merge master
```

Again, if merge conflicts occur, I resolve them manually and commit the resolved changes.


### Step 4: Merge the Feature Branches into Master

10. I checkout to the `master` branch:
```
git checkout master
```

11. I merge the `feature1` branch into `master`:
```
git merge feature1
```
<br>![[Pasted image 20231030125711.png]]
*No conflicts*

12. If any conflicts were to arise, I resolve them manually and commit the changes.

13. I then merge the `feature2` branch into `master`:



```
git merge feature2
```
<br>![[Pasted image 20231030125838.png]]
Conflicts found

I open `main.c` with Visual Studio Code to solved the conflicts
<br>![[Pasted image 20231030130028.png]]
I resolve the conflicts, add and commit the changes


### Step 5: Push All Branches to GitHub

14. Finally, I push all the branches to my forked GitHub repository:

```
git push origin master 
git push origin feature1 
git push origin feature2
```

### Solution Submission:

https://github.com/hectorproko/merge-conflict/tree/master

%%Remember, while working with real-world projects, it's essential to communicate with the team, especially when dealing with merge conflicts. Ensure you understand the changes made in all branches before integrating them to avoid breaking the application.
%%

%%
> [!question]- Yes, there's a reason behind the chosen approach:
> 1. **Security Patch on Master**: In the context you provided, the master branch received a security patch, which means the master branch has a commit that the feature branches do not have. Before integrating the feature branches into master, it's essential that these branches are up-to-date with such crucial updates from the master.
>     
> 2. **Ensuring Features Work with Latest Master**: By first updating the feature branches with the latest changes from the master (i.e., the security patch), you're ensuring that each feature works as expected with the updated master branch. This helps catch and fix any potential compatibility issues on the feature branch itself rather than on the master.
>     
> 3. **Keep Master Stable**: The master branch (or main, in some workflows) is often considered the stable or production-ready branch. By merging master into the feature branches first, you're doing the potentially unstable work (resolving conflicts, ensuring compatibility, etc.) in the feature branches. This way, the master remains stable until the feature is confirmed to work correctly with the updated master content.
>     
> 4. **Cleaner History**: Merging master into feature branches first (and resolving any conflicts there) can lead to a cleaner, more linear history when you finally merge the feature branches back into master.
>     
> 5. **Easier Rollbacks**: If something goes wrong during the merge in a feature branch, it's easier to reset or rollback changes in a feature branch without affecting the stability of the master branch.

%%