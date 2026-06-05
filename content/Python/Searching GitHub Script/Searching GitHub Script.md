---
tags:
  - Python
---

> [!info]
> This Python script connects to GitHub, scans each repository, and identifies files containing a specified keyword.


## 1. Connect to GitHub API

**1. Install the `requests` library:**

- Open your command prompt or terminal.
- Make sure you've activated your [[Python Virtual Environment#Windows|virtual environment]] virtual environment (if you created one).
- Type the following command and press Enter:

```bash
pip install requests
pip install termcolor
```

**2. Obtain a personal access token:**

[[Personal access tokens GitHub]]

## 3. Write the Python code:

### Connecting to API

[connectingAPI](https://github.com/hectorproko/github_keyword_search/blob/connectingAPI/github_keyword_search.py) Branch


```python
import requests

personal_access_token = "your_personal_access_token"
base_url = "https://api.github.com"

headers = {"Authorization": f"token {personal_access_token}"}

# Test the connection by getting your user information
user_url = f"{base_url}/user"
response = requests.get(user_url, headers=headers)

if response.status_code == 200:
    print("Connection successful!")
    print("Your GitHub username:", response.json()["login"])
else:
    print("Connection failed with status code:", response.status_code)
```


![[Pasted image 20240131192711.png]]

### Rest of code

Uses libraries
```python
from termcolor import colored
import os
```

```python
# Iterate through each repository and search its contents
for repo in repos:
    repo_name = repo["name"]
    print(repo_name)
    contents_url = f"{base_url}/repos/{repo['owner']['login']}/{repo['name']}/contents"
    #print(contents_url)
    response = requests.get(contents_url, headers=headers)
    contents = response.json()

    for file in contents:
        if file["type"] == "file":
            file_url = file["download_url"]
            #print(file_url)
            if file_url is not None:
                response = requests.get(file_url, headers=headers)
                #print(response)
                file_contents = response.text
                if keyword in file_contents:
                    #print(colored(f"Keyword found in {repo_name}/{file['path']}", "green"))
                    print(colored(f"Keyword found in {file['path']}", "green"))
            else:
                print("\033[91mInvalid URL or URL is None\033[0m")
```


Updated the token input to avoid hard coding

```bash
personal_access_token = os.environ.get("GITHUB_ACCESS_TOKEN")
```

Setting the environmental variable
```bash
set GITHUB_ACCESS_TOKEN=ghp_wcNBCXe3kNrWWwcNBCXe3kNrWWwcNBCXe3kNrWW
```



Trying the script
![[Python/Searching GitHub Script/5containers.gif]]