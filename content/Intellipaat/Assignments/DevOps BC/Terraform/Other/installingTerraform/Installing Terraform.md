---
hardlinked: "True"
quartz: "True"
refactored: "False"
completed: "False"
---

### Windows

1. **Downloading Terraform**
   First, I head over to the official Terraform website to grab the latest version. In the downloads section, I select the `Windows_AMD64` option, since that matches my system's architecture. Clicking the link initiates the download of a `.zip` file.

2. **Extracting the Terraform Executable**
   Once the download finishes, I find the `.zip` file in my Downloads folder. I right-click on it and choose `Extract All...`, which reveals the `terraform.exe` within. I decide to keep it simple and leave `terraform.exe` right there in the Downloads folder for now.

3. **Setting Up the Environment Variable**
   ![[Add.Edit Environment Variables (Windows)]]
4. **Testing the Terraform Command**
   To check if everything's working, I open a new Command Prompt window and type `terraform --version`.
  > [!success]
  >    ![[terraform --version.png]]

#### Refresh the Variable

```powershell
$env:PATH = [System.Environment]::GetEnvironmentVariable("Path", "User") + ";" + [System.Environment]::GetEnvironmentVariable("Path", "Machine")
```
  
### Linux

Went to terraform.io
Mention how to launch and connect to EC2 to install terraform. Mention [[PuTTY]]

```bash
curl -O <URL to 64Bit Linux Download>
unzip terraform.zip
mv terraform /usr/local/bin
terraform --version
```
---

We had to install in [[PART1_PROJECT_16]]