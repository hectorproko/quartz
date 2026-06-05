
> [!info] Module 2: Git Assignment - 1
> **Tasks To Be Performed:** 
> 1. Based on what you have learnt in the class, do the following steps: 
>    `a.` Create a new folder 
>    `b.` Put the following files in the folder 
>    - Code.txt 
>    - Log.txt 
>    - Output.txt 
>
>    `c.` Stage the Code.txt and Output.txt files 
>    `d.` Commit them 
>    `e.` And finally push them to GitHub 
> 2. Please share the commands for the above points

1. **I create a new folder**:

```bash
mkdir GitAssignment
```

2. **I navigate to the new folder**:

```bash
cd GitAssignment
```

3. **I create the required files in this folder**:

```bash
touch Code.txt Log.txt Output.txt
```

4. **I initialize a new Git repository in this folder**:

```bash
git init
```

5. **I stage `Code.txt` and `Output.txt` for the commit**:

```bash
git add Code.txt Output.txt
```

6. **I commit the staged files with a message**:

```bash
git commit -m "I added Code.txt and Output.txt files"
```

7. **I have already set up the remote repository**:
  - I've previously created a repository on GitHub named `GitAssignment`.
  - I've also configured SSH, enabling me to clone and push to the repository without entering my credentials each time.

8. **I link my local repository to the remote GitHub repository**:

```bash
git remote add origin git@github.com:hectorproko/GitAssignment.git
```

9. **I rename the default branch to `main`**:

```bash
git branch -M main
```
*GitHub repository's default branch is already named `main`*

10. **I push my commit to the remote GitHub repository**:

```bash
git push -u origin main
```

> [!success]
> Now, I've successfully added my files to the GitHub repository.
> 
> <br>![[Pasted image 20231027214429.png]]
> <br>![[Pasted image 20231027214512.png|300]]