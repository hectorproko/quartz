
> [!NOTE] Module 7 - Assignment
> **Problem Statement:**
> • Create a cron job that will run once a month, on the second data of the month
> • Create a cron job that will run thrice in an hour and this should repeat every hour
> 


#### 1. Cron Job Running Once a Month on the Second Day

```bash
0 0 2 * * /path/to/script.sh
```
Online cron tester "Cronitor" check:
![[Pasted image 20230820030259.png|300]]
#### 2. Cron Job Running Thrice an Hour, Every Hour
*(I'll pick minutes 0, 20, and 40)*
```bash
0,20,40 * * * * /path/to/script.sh
```
Online cron tester "Cronitor" check:
![[Pasted image 20230820030452.png|300]]

