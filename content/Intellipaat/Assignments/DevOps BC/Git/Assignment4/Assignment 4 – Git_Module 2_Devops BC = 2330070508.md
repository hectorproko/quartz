
> [!info] Module 2: Git Assignment - 4
> **Tasks To Be Performed:** 
> 1. Put master.txt on master branch, stage and commit 
> 2. Create 3 branches: public 1, public 2 and private 
> 3. Put public1.txt on public 1 branch, stage and commit 
> 4. Merge public 1 on master branch 
> 5. Merge public 2 on master branch 
> 6. Edit master.txt on private branch, stage and commit 
> 7. Now update branch public 1 and public 2 with new master code in private 
> 8. Also update new master code on master 
> 9. Finally update all the code on the private branch


Initialization: Create a Git working directory 
```
mkdir GitAssignment4 
cd GitAssignment4 
git init
```
### Step 1: 
Add master.txt on master branch, stage, and commit
```
echo "Content of master.txt" > master.txt
git add master.txt
git commit -m "Added master.txt to master branch"
```

### Step 2: 
Create 3 branches: public 1, public 2, and private
```
git branch public1
git branch public2
git branch private
```

### Step 3: 
Add public1.txt on public 1 branch, stage, and commit
```
git checkout public1
echo "Content of public1.txt" > public1.txt
git add public1.txt
git commit -m "Added public1.txt to public1 branch"
```

### Step 4: 
Merge public1 branch into master branch
```
git checkout master
git merge public1
```
<br>![[Pasted image 20231027232406.png]]
### Step 5: 
Public2 has no contents:
```
git merge public2
```
<br>![[Pasted image 20231027232350.png]]
### Step 6: 
Edit master.txt on private branch, stage, and commit
```
git checkout private
echo "Edited content of master.txt" >> master.txt
git add master.txt
git commit -m "Edited master.txt in private branch"
```

### Step 7: 
Update public1 and public2 branches with changes from private branch
```
git checkout public1
git merge private
```

```
git checkout public2
git merge private
```

### Step 8: 
Update master with the new code from private branch
```
git checkout master
git merge private
```

### Step 9: 
```
git checkout private
git merge master
```
<br>![[Pasted image 20231027233312.png]]
The `Fast-forward` indicates that `private` simply moved forward to the latest commit of `master`. As a result, the file `public1.txt` was added to the `private` branch.