---
"last-modified:": 2024-02-09
tags:
  - Python
---

> [!info]
> This Python script was born out of a practical need to maintain identical copies of files in different locations within a Windows environment, without the hassle of manually updating each copy whenever changes occur. By leveraging the power of hard links, this tool ensures that a single file can exist in multiple locations, with all copies being automatically updated when any one of them is modified. This approach not only saves time but also minimizes the risk of inconsistencies between file versions.
> 
> The script uses a graphical interface for selecting files and their destination directories. Utilizing the `tkinter` module for its user-friendly file and directory selection dialogs, and the `subprocess` module to execute Windows' native `mklink` command, the script simplifies what would otherwise be a complex process of creating hard links.

```python
import os
import subprocess
import tkinter as tk
from tkinter import filedialog

root = tk.Tk()  # Create a temporary root window
root.withdraw()  # Hide the root window

def create_hard_link(target_file, link_destination):
    """Creates a hard link to the target file in the specified link destination.

    Args:
        target_file (str): Full path to the target file.
        link_destination (str): Full path to the directory where the hard link
                               should be created, including the desired filename.
    """

    file_name = os.path.basename(target_file)  # Extract filename with extension
    link_path = os.path.join(link_destination, file_name)  # Construct link path with filename

    try:
        # Ensure command is run through cmd.exe, using /c to execute the command
        command = ['cmd.exe', '/c', 'mklink', '/h', link_path, target_file]
        #Error here
        subprocess.run(command, check=True, shell=True)
        print(f"\033[92mHard link created successfully: {link_path}\033[0m")  # Green text
    except subprocess.CalledProcessError as error:
        print(f"\033[91mFailed to create hard link: {error}\033[0m")  # Red text

# Example usage:
target_files = filedialog.askopenfilenames()  # Prompts for multiple files
link_destination = os.path.normpath(filedialog.askdirectory()) 


for target_file in target_files:  # Iterate over each selected file
    target_file = os.path.normpath(target_file)  # Normalize the path
    create_hard_link(target_file, link_destination)  # Create a hard link for each file

```

### Running it:
![[Pasted image 20240209222647.png]]