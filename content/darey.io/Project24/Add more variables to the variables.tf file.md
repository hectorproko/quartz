
10. Add more variables to the variables.tf file# create some variables  

```bash
# create some variables  
variable "admin_users" {  
type        = list(string)  
description = "List of Kubernetes admins."  
}  
variable "developer_users" {  
type        = list(string)  
description = "List of Kubernetes developers."  
}  
variable "asg_instance_types" {  
type        = list(string)  
description = "List of EC2 instance machine types to be used in EKS."  
}  
variable "autoscaling_minimum_size_by_az" {  
type        = number  
description = "Minimum number of EC2 instances to autoscale our EKS cluster on each AZ."  
}  
variable "autoscaling_maximum_size_by_az" {  
type        = number  
description = "Maximum number of EC2 instances to autoscale our EKS cluster on each AZ."  
}  
variable "autoscaling_average_cpu" {  
type        = number  
description = "Average CPU threshold to autoscale EKS EC2 instances."  
}
```
