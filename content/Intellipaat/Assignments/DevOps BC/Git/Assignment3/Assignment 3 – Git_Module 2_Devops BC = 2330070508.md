
> [!info] Module 2: Git Assignment - 3
> **Tasks To Be Performed:** 
> 1. Create a Git working directory, with the following branches: 
>    ● Develop 
>    ● F1 
>    ● f2 
> 2. In the master branch, commit main.txt file 
> 3. Put develop.txt in develop branch, f1.txt and f2.txt in f1 and f2 respectively 
> 4. Push all these branches to GitHub 
> 5. On local delete f2 branch 
> 6. Delete the same branch on GitHub as well


> [!tip]
> Will re-use repo `GitAssignment` from [[Assignment 1 – Git_Module 2_Devops BC = 2330070508|Assignment 1 – Git]] and just make sure its empty

### Step 1:
Clone the existing repository
```
git clone https://github.com/YOUR_USERNAME/GitAssignment.git
cd GitAssignment
```

Creating the branches
```
git branch Develop
git branch F1
git branch f2
```

### Step 2: 
Commit main.txt file in the master branch
```
echo "Content of main.txt" > main.txt
git add main.txt
git commit -m "Added main.txt to master branch"
```

### Step 3: 
Commit files in their respective branches

Develop branch:
```
git checkout Develop
echo "Content of develop.txt" > develop.txt
git add develop.txt
git commit -m "Added develop.txt to Develop branch"
```

F1 branch:
```
git checkout F1
echo "Content of f1.txt" > f1.txt
git add f1.txt
git commit -m "Added f1.txt to F1 branch"
```

f2 branch:
```
git checkout f2
echo "Content of f2.txt" > f2.txt
git add f2.txt
git commit -m "Added f2.txt to f2 branch"
```

### Step 4: 
Push all branches to GitHub
```
git push -u origin main
git push -u origin Develop
git push -u origin F1
git push -u origin f2
```

### Step 5: 
Delete f2 branch locally
```
git checkout main  # Switching to another branch first, since I can't delete the branch I'm currently on.
git branch -d f2
```
### Step 6: 
Delete the f2 branch on GitHub
```
git push origin --delete f2
```

<br>![[Pasted image 20231027225915.png]]
In GitHub
<br>![[Pasted image 20231027230027.png|400]]
