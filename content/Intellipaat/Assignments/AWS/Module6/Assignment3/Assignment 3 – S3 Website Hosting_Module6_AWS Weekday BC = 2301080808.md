---
tags:
  - AWS
title: Hosting Static Websites and Managing Lifecycle Policies in S3 Buckets on AWS
---
<!--
**Mini-Project: S3 Website Hosting Mastery!** In my latest endeavor, I dived into the realm of static website hosting using Amazon S3. This project involved setting up an S3 bucket to serve as a web host, configuring it to deliver a static website, and ensuring public accessibility through careful permission settings. I uploaded custom `index.html` and `error.html` pages, and implemented a lifecycle rule for efficient object management. This hands-on task allowed me to practically explore the seamless and scalable web hosting capabilities of S3, furthering my cloud expertise.
-->
> [!NOTE] Module 6: S3 Website Hosting Assignment 
> **Problem Statement:** 
> You work for XYZ Corporation. Their application requires a storage service that can store files and publicly share them if required. Implement S3 for the same.
> 
> **Tasks To Be Performed:** 
> 1. Use the created bucket in the previous task to host static websites, upload an index.html file and error.html page. 
> 2. Add a lifecycle rule for the bucket: 
>a. Transition from Standard to Standard-IA in 60 days 
>b. Expiration in 200 days


### **1. Set Up the S3 Bucket for Static Website Hosting:**

**Step 1:** I logged into the AWS Management Console and navigated to the S3 dashboard.

**Step 2:** Since I'm using the bucket I created in the previous task, I clicked on its name `my-test-bucket-unique-name` to access its settings.

**Step 3:** I headed over to the "Properties" tab.

**Step 4:** I scrolled down to find the "Static website hosting" box and clicked on Edit.

**Step 5:** I specified 'index.html' for the Index document and 'error.html' for the Error document. This means any visitor to my website's root URL will be served the 'index.html' file, and any errors will direct them to 'error.html'.
<br>![[Pasted image 20230927223731.png]]

**Step 6:** I noted down the endpoint URL provided by AWS, as this will be the URL for the hosted static website.
```bash
http://my-test-bucket-unique-name.s3-website-us-east-1.amazonaws.com
```

---

### **2. Upload index.html and error.html:**

**Step 1:** I clicked the “Upload” button inside the bucket.

**Step 2:** Using the "Add files" option, I selected and uploaded the `index.html` and `error.html` files from my computer.

**Index:**
```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta http-equiv="X-UA-Compatible" content="IE=edge">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>XYZ Corporation</title>
<style>
body {
font-family: Arial, sans-serif;
text-align: center;
padding: 50px;
}
</style>
</head>
<body>
<h1>Welcome to XYZ Corporation</h1>
<p>This is the main landing page for XYZ Corporation's static website hosted on Amazon S3.</p>
</body>
</html>
```

**error.html**
```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta http-equiv="X-UA-Compatible" content="IE=edge">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Error - XYZ Corporation</title>
<style>
body {
font-family: Arial, sans-serif;
text-align: center;
padding: 50px;
color: red;
}
</style>
</head>
<body>
<h1>Error!</h1>
<p>Sorry, the page you are looking for could not be found.</p>
<p><a href="index.html">Return to Home</a></p>
</body>
</html>
```

**Step 3:** After both files were added, I clicked the "Upload" button to finalize the upload process.

1. Access the Bucket URL: `http://my-test-bucket-unique-name.s3-website-us-east-1.amazonaws.com`
  <br>![[Pasted image 20230927223840.png]]
  *We get access denied*

2. Modify Public Access Settings:

  - Navigated to the bucket settings. Inside the bucket, moved to the "Permissions" tab.
  - Selected "Block public access (bucket settings)".
  - Clicked on "Edit".
  - Unchecked "Block all public access" to allow changes to individual object permissions.
    <br>![[Pasted image 20230928102228.png]]

3. Update the Bucket Policy:

  - While still in the "Permissions" tab of the bucket, selected "Bucket Policy".  
  - Inserted the following policy to grant public read access to the `index.html` and `error.html`:


```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": [
                "arn:aws:s3:::my-test-bucket-unique-name/index.html",
                "arn:aws:s3:::my-test-bucket-unique-name/error.html"
            ]
        }
    ]
}

```

4. Verify the Changes:
  - Access the website again using the provided URL.

> [!success]
>   <br>![[Pasted image 20230928102434.png]]

---

### **3. Setting Up Lifecycle Rule for the Bucket:**

**Step 1:** Still inside my bucket’s settings, I navigated to the "Management" tab.

**Step 2:** I clicked on the "Create lifecycle rule" button.

**Step 3:** I gave the lifecycle rule a name, "TransitionAndExpirationRule".

**Step 4**: I made sure to select "Apply to all objects in the bucket" for scope.

**Step 5:** Now, for the actions:

**a. Transition to Standard-IA Storage:**

1. I checked the option that read "Move current versions of objects between storage classes."
2. When prompted under "Choose storage class transitions", I set "days after object creation" to the desired value.
3. I clicked on "Add transition."


**b. Object Expiry after 200 Days:**

1. I spotted the "Expire current versions of objects" option and checked it.
2. I wanted to ensure objects were deleted after a set time, so I specified "200" days for their automatic deletion.
    
<br>![[Pasted image 20230928110726.png]]

**Step 6:** I took a moment to review everything. Satisfied, I clicked "Create rule".

<br>![[Pasted image 20230928111253.png]]

---

With these steps, I've converted my S3 bucket into a host for a static website and set a lifecycle rule to manage the storage and expiration of my objects. I can now access my static website using the endpoint URL provided by AWS.


