---
tags:
  - Bash
---

> [!NOTE] Module 5 - Assignment
> **Problem Statement:**
> You work for xyz organization. Your organization uses a UNIX/Linux based server.
> 
> You have been asked to:
> 
> 1. Create 2 shell script files — `one.sh` and `two.sh`
> 2. In `one.sh`, create a code which involves looping statements, Conditional statements and 3 functions
> 3. In `two.sh`, create a Case..esac statement in which first option runs the first function, 2nd option runs 2nd function and 3rd option runs the third function

1. Creating script `one.sh` containing a loop, a conditional statement, and three functions.
```bash
nano one.sh
```
The contents:
```bash
#!/bin/bash

# Function to greet
greet() {
  echo "Hello, $1"
}

# Function to calculate square of a number
square() {
  echo "Square of $1 is $(($1 * $1))"
}

# Function to check if a number is even or odd
check_even_odd() {
  if [ $(($1 % 2)) -eq 0 ]; then
    echo "$1 is even"
  else
    echo "$1 is odd"
  fi
}

# Loop to call functions
for i in 1 2 3; do
  if [ $i -eq 1 ]; then
    greet "User"
  elif [ $i -eq 2 ]; then
    square 4
  elif [ $i -eq 3 ]; then
    check_even_odd 5
  fi
done
```

Making script executable:
```bash
chmod +x one.sh
```

Testing `one.sh`  
![[Pasted image 20230820014809.png]]

2. Creating script `two.sh` containing a `case..esac` statement to call the functions defined in `one.sh`.
```bash
nano two.sh
```
The contents:
```bash
#!/bin/bash

# Source the functions from one.sh
source ./one.sh

# Menu for case..esac
echo "Choose an option:"
echo "1. Greet User"
echo "2. Square a number"
echo "3. Check if a number is even or odd"

read choice

# Case..esac statement
case $choice in
  1)
    greet "User"
    ;;
  2)
    square 4
    ;;
  3)
    check_even_odd 5
    ;;
  *)
    echo "Invalid choice"
    ;;
esac
```

Making script executable:
```bash
chmod +x two.sh
```

Testing `two.sh`  
![[Pasted image 20230820015844.png]]
Executing `two.sh` also executes `one.sh` that's why we see its output at the beginning. Then we see the output of `two.sh` which prompts us to pick an option, I picked 2 which triggered function `square`


