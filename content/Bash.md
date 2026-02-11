---
tags:
  - Bash
---
Some Scripts:
[Scripts](https://github.com/hectorproko/Work_Scripts/tree/master/Bash)  
[Auxillary-Project](https://github.com/hectorproko/Auxillary-Projects/blob/main/Steps.md)  

---

https://github.com/hectorproko/Work_Scripts/tree/master/Bash

****

> [!summary] Quick Summary:
> 1. **fullAutoBuild.sh** - Main script, calls everything in order
> 2. **Pull_Step_Full.sh** - Gets latest code from repos
> 3. **autoBuildV3_Full.bash** - Builds 7 packages, extracts artifact numbers from Jenkins console
> 4. **FileUpdatorV3.sh** - Updates version numbers in project files
> 5. **Push_StepV3_Full.sh** - Pushes updated code to GitHub
> 6. **buildExAll_Full.sh** - Triggers 60+ app builds in Jenkins
> 7. **cloneDevops.sh** - One-time setup to clone all repos

# Jenkins Build Scripts - Simple Overview

## Script Relationships

```
fullAutoBuild.sh (MAIN ORCHESTRATOR)
    ↓
    ├─→ Pull_Step_Full.sh          (Pull/clone all repos)
    ↓
    ├─→ autoBuildV3_Full.bash      (Build packages, extract artifacts)
    │   ├─→ Calls: FileUpdatorV3.sh (Updates version numbers in files)
    │   └─→ Creates: MegaArtifacts_${branch}_${date}.txt
    ↓
    ├─→ Push_StepV3_Full.sh        (Push updated repos to GitHub)
    │   └─→ Reads: MegaArtifacts file
    ↓
    └─→ buildExAll_Full.sh         (Trigger 60+ app builds in Jenkins)
```

---

## What Each Script Does

### **fullAutoBuild.sh** - Main Script

- Orchestrates the entire pipeline
- Calls other scripts in sequence
- Tracks total execution time

### **Pull_Step_Full.sh** - Get Latest Code

- Pulls latest code from all repos
- Clones repos if missing
- If pull fails: removes repo and re-clones

### **autoBuildV3_Full.bash** - Build & Extract

- Builds 7 packages in order:
    1. WorkabilityFramework
    2. WKABCommon
    3. Workability
    4. ScriptEngine
    5. WorkabilityCustomerPortal
    6. BENEFITENGINE
    7. WkabBulkClosure
- For each package:
    - Edits Jenkinsfile (SNAPSHOT/RELEASE)
    - Pushes to GitHub
    - Triggers Jenkins build
    - **Extracts artifact number from Jenkins console**
    - Saves to MegaArtifacts file
    - Updates all repos with new version

### **FileUpdatorV3.sh** - Update Version Numbers

- Updates package version numbers in:
    - packages.config
    - .csproj files
    - .vbproj files
- Handles SNAPSHOT vs RELEASE builds

### **Push_StepV3_Full.sh** - Push Updated Code

- Reads MegaArtifacts file
- Updates all app repos with package versions
- Commits and pushes to GitHub
- Moves MegaPackage repos to temp (to skip them)
- Has retry logic if push fails

### **buildExAll_Full.sh** - Trigger App Builds

- Triggers 60+ Jenkins jobs (WEB_SERVICES + WINDOWS_SERVICES)
- All triggered in parallel
- Uses curl POST to Jenkins API

### **cloneDevops.sh** - One-Time Setup

- Initial setup script
- Clones all repos listed in repo.txt
- Checks out specific branch

---

## Data Flow

```
1. autoBuildV3_Full.bash builds packages
   ↓
2. Extracts artifact numbers: "WorkabilityFramework 2021.1234.0"
   ↓
3. Saves to: MegaArtifacts_B.R.04042020_02-27-21.txt
   ↓
4. FileUpdatorV3.sh reads this file
   ↓
5. Updates version in all repo files
   ↓
6. Push_StepV3_Full.sh pushes changes
   ↓
7. buildExAll_Full.sh triggers app builds
```

---

## **Phase 1: BUILD** (First set of scripts)

- `autoBuildV3_Full.bash` + `buildExAll_Full.sh`
- **Builds** all packages and apps
- Extracts artifacts **in real-time while building**

## **Phase 2: EXTRACT** (New scripts you just shared)

- `Art_extract.sh`
- **Reads already-completed** Jenkins builds
- Extracts artifact versions from console output
- Creates: `ArtifactsVersions.txt`

## **Phase 3: DEPLOY** (singleclickdeploy.sh)

- Reads `version.txt` (artifact versions)
- **Generates deploy scripts** with artifact versions
- Executes deploys to X environment


**Here's a strong answer that explains the technical details clearly:**

---
<!--
## **Version 1 - Technical but Clear (Recommended)**

The 88% improvement came from automating a complex, multi-step build process that was extremely time-consuming when done manually. Our application ecosystem consisted of 7 core packages that had to be built in a specific dependency order, and those packages were then referenced by 60+ application repositories. The manual process required someone to build each package, wait for completion, extract the artifact version number from the Jenkins console output, manually update version numbers in all the application repositories' configuration files, commit and push those changes to GitHub, and then trigger each of the 60+ application builds individually. This took 60-65 minutes and was error-prone.

I designed a Bash script orchestration system that automated the entire workflow. The main intelligence was in how the scripts extracted data from Jenkins and used it to drive subsequent steps. Here's what made it work:

First, I created a script that built the 7 packages in the correct dependency order—WorkabilityFramework, WKABCommon, Workability, ScriptEngine, and so on. But the key innovation was that after triggering each package build through the Jenkins API, the script didn't just wait—it actively monitored the Jenkins console output in real-time using curl requests, parsed the build logs to extract the specific artifact version number (like "2021.1234.0"), and saved these to a central artifacts file. This eliminated the manual step of opening each Jenkins job and copying version numbers.

Second, I built intelligence into the version update process. The script read the extracted artifact versions and automatically updated package references across all 60+ application repositories—editing packages.config files, .csproj files, and .vbproj files with the correct SNAPSHOT or RELEASE version numbers based on the build type. This alone saved 20-30 minutes of manual file editing.

Third, I implemented self-correcting logic. If a git pull failed due to conflicts or network issues, the script automatically removed the local repository and re-cloned it fresh. If a git push failed, the script had retry logic with exponential backoff. This meant builds that would have failed and required manual intervention could now complete successfully without human involvement.

Finally, instead of triggering 60+ application builds one by one, I parallelized the process—the script used the Jenkins REST API to trigger all builds simultaneously via curl POST requests, reducing what was 15-20 minutes of clicking through Jenkins UI to about 30 seconds of automated API calls.

The result: a process that took 60 minutes of careful manual work was reduced to 7 minutes of fully automated execution. More importantly, the scripts eliminated the human error factor—no more typos in version numbers, no more forgotten repositories, no more builds triggered in the wrong order. The team could kick off the entire build process with a single command and trust it would complete correctly.

---

## **Version 2 - More Concise (If They Want Shorter)**

The manual process was incredibly time-consuming: build 7 packages in dependency order, extract artifact version numbers from Jenkins console output, manually update those versions in 60+ application repositories, push changes to GitHub, then trigger each application build individually—all of which took about an hour.

I automated this with a Bash script orchestration system that had three key innovations:

**Real-time artifact extraction:** After triggering each package build via Jenkins API, my script actively monitored the Jenkins console output using curl, parsed the build logs to extract the specific artifact version number, and saved it to a central file. This eliminated manually copying version numbers from 7 different Jenkins jobs.

**Intelligent version updating:** The script read the extracted versions and automatically updated package references across all 60+ repositories—editing packages.config, .csproj, and .vbproj files with correct SNAPSHOT or RELEASE versions. This saved 20-30 minutes of manual file editing.

**Self-correcting logic and parallelization:** If git operations failed, the script automatically recovered—removing and re-cloning repos on pull failures, retrying pushes with backoff logic. Instead of triggering 60+ builds manually, it parallelized them through Jenkins REST API, reducing 15-20 minutes of UI clicking to 30 seconds of automated calls.

The result was a 60-minute manual process reduced to 7 minutes of fully automated, error-free execution that could be kicked off with a single command.

---

## **Version 3 - Story Format (More Engaging)**

Let me walk you through what this process looked like before and after automation, because that really shows where the time savings came from.

Before automation, here's what someone had to do manually: We had 7 core packages that needed to be built in a specific order because they had dependencies—you couldn't build Workability until WorkabilityFramework was done, for example. So you'd trigger the first build in Jenkins, wait for it to complete, open the Jenkins console output, scroll through the logs to find the artifact version number—something like "2021.1234.0"—copy that number, then go into 60+ application repositories and manually update the package version references in their configuration files. After updating all those files, you'd commit and push the changes to GitHub, then go back to Jenkins and manually trigger each of the 60+ application builds one by one. This took over an hour, and if you made a typo in a version number or forgot to update a repository, you'd have inconsistent builds.

I built a Bash script orchestration system that automated all of this. The most critical piece was making the scripts "intelligent" enough to extract information from Jenkins and use it to drive the next steps. Here's how it worked:

The main script triggered the first package build through the Jenkins API, but instead of just waiting passively, it actively monitored the Jenkins console output in real-time using curl requests. It parsed the build logs looking for the artifact version number, extracted it automatically, and saved it to a file. Then it moved to the next package build and repeated the process. This meant I didn't need anyone manually copying version numbers from console output—the script did it in real-time as builds completed.

Once all packages were built, the next script read those extracted version numbers and automatically updated all 60+ application repositories. It knew exactly which files to edit—packages.config, .csproj files, .vbproj files—and it handled both SNAPSHOT and RELEASE version formats correctly. This eliminated 20-30 minutes of manual file editing and the risk of typos.

I also built in self-correction. If a git pull failed due to conflicts, the script automatically deleted the local repo and re-cloned it fresh. If a git push failed, it had retry logic with exponential backoff. This meant builds that would have failed and required someone to troubleshoot could now complete successfully on their own.

Finally, instead of manually triggering 60+ builds, the script used the Jenkins REST API to trigger all of them in parallel—a single curl POST command for each build, executed simultaneously. This turned 15-20 minutes of clicking through Jenkins into about 30 seconds of automated API calls.

The end result: what took 60 minutes of careful manual work became a 7-minute automated process that someone could kick off with a single command. More importantly, it was consistent and error-free—no more typos, no more forgotten repos, no more builds triggered out of order.

-->
