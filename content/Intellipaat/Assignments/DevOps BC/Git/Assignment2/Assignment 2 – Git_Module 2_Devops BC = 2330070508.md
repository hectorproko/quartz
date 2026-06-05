
> [!info] Module 2: Git Assignment - 2
> **Tasks To Be Performed:** 
> 1. Create a Git working directory with feature1.txt and feature2.txt in the master branch 
> 2. Create 3 branches develop, feature1 and feature2 
> 3. In develop branch create develop.txt, do not stage or commit it 
> 4. Stash this file and check out to feature1 branch 
> 5. Create new.txt file in feature1 branch, stage and commit this file 
> 6. Checkout to develop, unstash this file and commit 
> 7. Please submit all the Git commands used to do the above steps 


### Step 1
```bash
mkdir git_directory
cd git_directory
git init
echo "This is feature 1" > feature1.txt
echo "This is feature 2" > feature2.txt
git add .
git commit -m "Initial commit with feature1.txt and feature2.txt"
```

### Step 2
```bash
git branch develop
git branch feature1
git branch feature2
```

### Step 3
```bash
git checkout develop
echo "This is develop file" > develop.txt
```

### Step 4
```bash
git stash push -u #-u for untracked files
git checkout feature1
```
*to prevent carrying it over when i switch branches*
### Step 5
```bash
echo "This is a new file in feature1" > new.txt
git add new.txt
git commit -m "Added new.txt in feature1 branch"
```

### Step 6
```bash
git checkout develop
git stash pop
git add develop.txt
git commit -m "Added develop.txt in develop branch"
```
