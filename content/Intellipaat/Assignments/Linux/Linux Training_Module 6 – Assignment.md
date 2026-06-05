
> [!NOTE] Module 6 - Assignment
> **Problem Statement:**
> In your primary system you have a folder consist that consists of the dataset that we need to perform task on. Mount the folder to the Linux operating system. The dataset is stored in zipped file. Extract the file using 'zip' command. You can find the dataset in the attachment. Once done mounting perform the tasks given below:
> 
> a.  Write an awk program to printout only the Name of the game and Year of release
> b.  Write an awk program to print only the Name column from the row 200 to row 210
> c. Write an awk program to print all the games released on the year 2005 between the rows 100 to 500
> 


I'll be using GitBash and I have the dataset `linux_mod6_dataset.csv` in my Downloads folder
#### a) Print the Name of the game and Year of release
```bash
awk -F ',' 'NR > 1 {print $2, $4}' linux_mod6_dataset.csv
```
![[Pasted image 20230820024146.png]]
*Used `head` here to make output smaller*

#### b) Print only the Name column from row 200 to row 210
```bash
awk -F ',' 'NR >= 200 && NR <= 210 {print $2}' linux_mod6_dataset.csv
```

Our target:
![[Pasted image 20230820024520.png]]
Running awk:  
![[Pasted image 20230820024624.png]]

#### c) Print all the games released in the year 2005 between rows 100 to 500
```bash
awk -F ',' 'NR >= 100 && NR <= 500 && $4 == 2005 {print $2}' linux_mod6_dataset.csv
```
![[Pasted image 20230820024818.png]]

