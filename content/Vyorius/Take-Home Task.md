
# NodeJS API
## Part 1 Launch and Deploy a Web App Image to AWS ECS Fargate

**Step 1: Requirements**
- An AWS account.
- A Node.js application
  ![[nodejs-api-git-repo.zip]]
- AWS CLI installed 
- Docker installed 

![[Pasted image 20241207081200.png]]

**Step 2: Building and Testing the Web App**

> [!NOTE]- Review Dockerfile
> ```dockerfile
> # Use Node.js 14 as the base image
> FROM node:14
> 
> # Set the working directory inside the container
> WORKDIR /usr/src/app
> 
> # Copy package.json and package-lock.json files
> COPY package*.json ./
> 
> # Install dependencies
> RUN npm install
> 
> # Copy the entire application code into the container
> COPY . .
> 
> # Expose port 3000
> EXPOSE 3000
> 
> # Command to run the application
> CMD ["node", "app.js"]
> ```

> [!NOTE]- Review app.js file
> ```javascript
> const express = require('express');
> const app = express();
> 
> // Endpoint to return server status and uptime
> app.get('/status', (req, res) => {
>     res.json({
>         status: 'running test 8',
>         uptime: process.uptime(), // Uptime in seconds
>     });
> });
> 
> // Start the server on a specific port
> const PORT = 3000;
> app.listen(PORT, '0.0.0.0', () => {
>     console.log(`Server is running on port ${PORT}`);
> });
> ```
> 

Building an image and running it to test it:
 - Build and tag the Docker image: `docker build -t node-status-api .`
- Test the container locally: `docker run -p 3000:3000 node-status-api`

> [!done] Works
> ![[Pasted image 20241211090237.png]]

**Step 3: Configuring AWS CLI**
- Created an AWS CLI user `vyorius-demo` and attach the `AmazonEC2ContainerRegistryFullAccess` policy.

- Generate and configure AWS CLI credentials for this user.
  ![[Pasted image 20241207082545.png]]

**Step 4: Creating and Pushing the Docker Image to AWS ECR**

Created a private AWS ECR repository. 

Inside the repository I'll click "View push commands" to authenticate Docker with ECR, tag the image, and push it to the repository.


`aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 838427752759.dkr.ecr.us-east-1.amazonaws.com`

`docker build -t nodejs-api-repository .`

`docker tag nodejs-api-repository:latest 838427752759.dkr.ecr.us-east-1.amazonaws.com/nodejs-api-repository:latest`

`docker push 838427752759.dkr.ecr.us-east-1.amazonaws.com/nodejs-api-repository:latest`

We are pushing the **Node14** image to our ECR repository specifically to update the `FROM` statement in the Dockerfile. This ensures the image is pulled from our private repository rather than over the internet from Docker Hub. This change improves efficiency and avoids rate limit issues with Docker Hub.

The updated Dockerfile will look like this:
```dockerfile
FROM 838427752759.dkr.ecr.us-east-1.amazonaws.com/nodejs-api-repository:Node14
```

`docker pull node:14`

`docker tag node:14 838427752759.dkr.ecr.us-east-1.amazonaws.com/nodejs-api-repository:Node14`

`docker push 838427752759.dkr.ecr.us-east-1.amazonaws.com/nodejs-api-repository:Node14`

**Step 5: Creating Security Groups**
Create two security groups:
 - **Application Load Balancer (ALB) Security Group**: Open port 80,443 (HTTP/S) to the world.  `nodejs-api-alb-sg`   
 - **ECS Service Security Group**: Open port 3000 (Node.js app)

**Step 6: Creating an ECS Fargate Cluster**
We created AWS Fargate (serverless) cluster `nodejs-api-cluster`

**Step 7: Creating a Task Definition**
Task definition family: `nodejs-api-task-definition`
CPU: `0.5 vCPU`
Memory: `1GB`
Task execution role: `ecsTaskExecutionRole`
Container Name: `nodejs-api-container` 
Image URI: `838427752759.dkr.ecr.us-east-1.amazonaws.com/nodejs-api-repository:latest`
Container port: 80 and 3000

**Step 8: Creating an ECS Service**
- Attach the task definition to the ECS cluster.
- Configure load balancer settings, including health checks and a target group *(the ECS containers)*.
  
> [!done] Testing Load Balancer
> ![[Pasted image 20241211164626.png]]
> ![[Pasted image 20241211164829.png]]

I set up the subdomain `api.hectorproko.com` to direct traffic to the load balancer

![[Pasted image 20241213223519.png]]

I created a wildcard certificate for the domain `*.hectorproko.com`. This allows secure connections for all subdomains under `hectorproko.com`, such as `www.hectorproko.com` or `api.hectorproko.com`.
![[Pasted image 20241213224114.png]]
## Part 2 Setting Up a CI/CD Pipeline with AWS Tools

**Step 1: Create a CodeCommit Repository**
Created CodeCommit `nodejs-api-git-repo` and move the code there

**Step 2: Configure AWS CodeBuild**
Made sure the role used `codebuild-nodejs-api-codebuild-service-role` has `AmazonEC2ContainerRegistryFullAccess`

> [!NOTE]- Create CodeBuild
> ![[Pasted image 20241211180813.png]]

Made sure i have `buildspec.yml`

> [!NOTE]- `buildspec.yml` code
> ```
> version: 0.2
> 
> phases:
>   pre_build:
>     commands:
>       - echo Logging in to Amazon ECR...
>       - aws --version
>       - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
>       - REPOSITORY_URI=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME
>   build:
>     commands:
>       - echo Build started on `date`
>       - echo Building the Docker image...
>       - docker build -t $REPOSITORY_URI:$IMAGE_TAG .
>       - docker tag $REPOSITORY_URI:$IMAGE_TAG $REPOSITORY_URI:$IMAGE_TAG
>   post_build:
>     commands:
>       - echo Build completed on `date`
>       - echo Pushing the Docker images...
>       - docker push $REPOSITORY_URI:$IMAGE_TAG
>       - echo Writing image definitions file...
>       - printf '[{"name":"%s","imageUri":"%s"}]' $CONTAINER_NAME $REPOSITORY_URI:$IMAGE_TAG > imagedefinitions.json
> 
> artifacts:
>     files: imagedefinitions.json
> ```

Variables that need values:
- AWS_DEFAULT_REGION
- AWS_ACCOUNT_ID
- IMAGE_REPO_NAME
- IMAGE_TAG
- REPOSITORY_URI
- CONTAINER_NAME

**Step 3: Set Up AWS CodePipeline**

> [!NOTE]- Choose pipeline settings:
> ![[Pasted image 20241211184500.png]]

> [!NOTE]- Add source stage:
> ![[Pasted image 20241211182406.png]]

> [!NOTE]- Add build stage:
> *I am assigning values to the variables I declared earlier.*
> ![[Pasted image 20241211194052.png]]

> [!NOTE]- Add deploy stage:
> ![[Pasted image 20241211182831.png]]

> [!NOTE]- Review
> ![[Pasted image 20241211184813.png]]



# S3 Static Site

**Create bucket `staticsite-44656056`**

Created **index.html** and **error.html**

**index.html**
```html
<!DOCTYPE html>
<html>
<head>
<title>Page Title</title>
</head>
<body>

<h1>Hello, DevOps!</h1>

</body>
</html>
```

**error.html**
```html
<!DOCTYPE html>
<html>
<head>
<title>Page Title</title>
</head>
<body>

<h1>Error Page</h1>

</body>
</html>
```

We select the Bucket and navigate Properties > Static website hosting
![[Pasted image 20241211212206.png]]
Make sure to enable web hosting
![[Pasted image 20241211212430.png|300]]

I upload the files and make sure they are made public using ACL
![[Pasted image 20241211213125.png]]

If navigate to the Static website hosting section once again I get an URL I can test my page with

```
http://staticsite-44656056.s3-website-us-east-1.amazonaws.com
```

I have create a subdomain `hello.hectorproko.com` that redirects to the static page
![[Pasted image 20241211212742.png]]

## CloudFormation
Created a CloudFormation distribution linked to the previously created Amazon ACM certificate to enable HTTPS."
![[Pasted image 20241213224114.png]]
![[Pasted image 20241213232331.png]]

```
https://d2sub371zvvrmc.cloudfront.net/
```

![[Pasted image 20241213231629.png]]

## Pipeline

Set up a CodeCommit repository to push changes to the pages, triggering a pipeline that updates the S3 bucket.

<iframe title="Recording 2024 12 13 230245" src="https://www.youtube.com/embed/MRQfpYa9ftY?feature=oembed" height="113" width="200" style="aspect-ratio: 1.76991 / 1; width: 100%; height: 100%;" allowfullscreen="" allow="fullscreen"></iframe>
