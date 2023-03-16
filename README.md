# Pipeline-to-build-resume-website-and-deploy-to-AWS
How to securely host a resume in the cloud using Gitlab CI, AWS S3 Bucket, AWS CloudFront Distribution. Cost of such project per month and per year.

Here are the different steps we will go through in this project:
Step 0: Buy an AWS domain name
Step 1: Having ready the manuscript of our resume
Step 2: Create and configure a S3 bucket
Step 3: Configure the CI/CD pipeline in GitLab to build and deploy our application to S3
Step 4: Create a CloudFront distribution for an S3 bucket: This will allow us to serve our resume securely over HTTPS.
Step 5: Request a certificate from ACM
Step 6: Add records to the hosted zone
Step 7: The cost of our project

Let’s start then!

## Step 0: Buy an AWS domain name:
1.	Log in to the AWS Management Console with your AWS account credentials.
2.	Navigate to Route 53 service from the list of available services.
3.	Click on the "Registered domains" link in the left-hand menu.
4.	Search for the domain name you want to buy using the search bar provided or propose yours.  let’s choose “name.resume.example.com” for our project.
5.	Review the details of the domain registration, including the registration period and the price.
6.	Click on the "Continue" button to proceed to the checkout page.
7.	Enter the contact information of the person who will be the owner of the domain name, including their name, address, and email address.
8.	Choose your payment method, either a credit card or an AWS account.
9.	Review your order details and click on the "Complete purchase" button to complete the domain registration.
Once the domain name is registered, the owner will receive an email from AWS with instructions on how to manage the domain. This may include configuring DNS settings, setting up email forwarding, and renewing the domain registration when it expires. It is important to ensure that the owner of the domain name is aware of their responsibilities and has access to the necessary AWS services to manage the domain.

## Step 1: Having ready the manuscript of our resume
1.	Having our resume written on a simple word format
2.	Editing an index.html file with the content of our word file (for this project we will just download and edit a free JavaScript resume template online with these elements: assets/img/, scripts/, js/, index.html, error.html)

## Step 2: Create and configure a S3 bucket
1.	Create an AWS account if we don't have one already.
2.	Create an S3 bucket and enable versioning on it. This will allow us to keep track of changes to our resume over time.
Steps to create an S3 bucket and enable versioning on it:
  •	Log in to our AWS account and go to the S3 console.
  •	Click on "Create bucket" to start creating a new S3 bucket.
  •	Choose a unique name for our bucket. This name must be globally unique across all AWS accounts.
  •	Select the region where we want to create our bucket.
  •	Choose the appropriate options for versioning, logging, and encryption. For versioning, select "Enable versioning" to keep track of changes to our files over time.
  •	Click on "Next" to configure the bucket's permissions.
  •	Choose the appropriate options for access control. We can choose to keep the bucket private or make it publicly accessible.
  •	Click on "Next" to review our bucket's settings.
  •	Review our bucket's settings and click on "Create bucket" to create our new S3 bucket.
Once our S3 bucket is created, we can enable versioning on it by following these steps:
  •	Go to the S3 console and select our bucket.
  •	Click on the "Properties" tab.
  •	Under "Advanced Settings", click on "Versioning".
  •	Click on "Enable versioning" to turn on versioning for your bucket.
  •	Click on "Save" to save your changes.
Now, any changes made to our files in the bucket will be automatically versioned and we can view and restore previous versions of your files as needed.
3.	Create an IAM user with appropriate permissions to deploy files to the S3 bucket. AWS CLI configured appropriately in Gitlab will use IAM user credentials to interact with the S3. While creating IAM user make sure to generate an access key and secret key for the user(For this project, choose “Attach existing policies directly” and then search for “AmazonS3FullAccess” as Policy name.)

## Step 3: Configure the CI/CD pipeline in GitLab to build and deploy our application to S3
1.	Configure GitLab CI to build our resume. We can use a template or create our own configuration file to do this. Here is an example configuration file (.gitlab-ci.yml) created in the root directory of your repository (To create that file, let’s open our repository in GitLab and navigate to the root directory. Here, we can create a new file called .gitlab-ci.yml and paste the following contents:)

stages:
    - build
build website:
    image: nginx:latest
    stage: build
    script:
        - cp -R . /usr/share/nginx/html/
        - rm -f /builds/<our-gitlab-group-name-for-project>/<our-project-folder-name>/.gitlab-ci.yml
        - rm -rf /builds/<our-gitlab-group-name-for-project>/<our-project-folder-name>/.git    
    artifacts:
        paths:
            - /builds/<our-gitlab-group-name-for-project>/<our-project-folder-name>

  
This configuration file builds the content of <our-gitlab-group-name-for-project>/<our-project-folder-name> and stores it in the /usr/share/nginx/html/ directory. By the same time, it removes the .gitlab-ci.yml and .git/ from the artifacts.

2.	Configure GitLab CI to upload our resume to the S3 bucket. We can use the AWS CLI to do this as indicated in the “deploy” stage of our configuration file (.gitlab-ci.yml). 

stages:
    - build
    - test
    - deploy
    - post deploy

variables:
    APP_BASE_URL: <link-from-our-static-website-hosting-under-s3-bucket-properties>

build website:
    image: nginx:latest
    stage: build
    script:
        - cp -R . /usr/share/nginx/html/
        - rm -f /builds/<our-gitlab-group-name-for-project>/<our-project-folder-name>
/.gitlab-ci.yml
        - rm -rf /builds/<our-gitlab-group-name-for-project>/<our-project-folder-name>/.git    
    artifacts:
        paths:
            - /builds/<our-gitlab-group-name-for-project>/<our-project-folder-name>

test website:
    image: alpine
    stage: test
    script:
        - test -f /builds/<our-gitlab-group-name-for-project>/<our-project-folder-name>
/index.html
    
deploy to s3:
    stage: deploy
    image: 
        name: amazon/aws-cli:2.4.11
        entrypoint: [""]
    rules:
        - if: $CI_COMMIT_REF_NAME == $CI_DEFAULT_BRANCH
    script:
        - aws --version
        - aws s3 sync /builds/<our-gitlab-group-name-for-project>/<our-project-folder-name>
s3://$AWS_S3_BUCKET --delete

production tests:
    stage: post deploy
    image: curlimages/curl
    rules:
        - if: $CI_COMMIT_REF_NAME == $CI_DEFAULT_BRANCH
    script:
        - curl $APP_BASE_URL | grep "<our-index-file-title>"


This configuration file (.gitlab-ci.yml) first builds your resume and stores it in the /usr/share/nginx/html/ directory. It tests to be sure that we have the index.html file in our artifacts (this test job is optional and can be skipped). It then uses the AWS CLI to upload the contents of the /usr/share/nginx/html/ directory to the S3 bucket we created earlier.
Navigate to your repository in GitLab and click on "Settings".
Click on "CI/CD" in the left sidebar.
Under "Runners", select the runner you want to use for your pipeline.
Under "Variables", add the following environment variables:
AWS_ACCESS_KEY_ID: The access key ID of the IAM user with permissions to deploy to S3.
AWS_SECRET_ACCESS_KEY: The secret access key of the IAM user.
AWS_DEFAULT_REGION: The AWS region where your S3 bucket is located.
S3_BUCKET_NAME: The name of your S3 bucket.
Save your changes.

Bucket Policies declaration in S3 for this project (Bucket/Permissions/Bucket Policy), then edit.

{
  "Version": "2012-10-17",
      "Statement": [{
          "Sid": "AllowGetObject",
          "Principal": {
              "AWS": "*"
          },
          "Effect": "Allow",
          "Action": "s3:GetObject",
          "Resource": "arn:aws:s3::: S3_BUCKET_NAME/*"
}

  
## Step 4: Create a CloudFront distribution for an S3 bucket: This will allow us to serve our resume securely over HTTPS.
  1. Open the AWS Management Console and navigate to the CloudFront service.
  2. Click on the "Create Distribution" button.
  3. Select the "Web" distribution type and click "Get Started".
  4. In the "Origin Domain Name" field, select the S3 bucket we want to use as the origin for our CloudFront distribution.
  5. Configure the other settings for our distribution, including the origin protocol policy, the viewer protocol policy, and the default cache behavior.
  6. Optionally, we can also configure additional cache behaviors or add custom error pages. In the "Viewer Protocol Policy" field, select "Redirect HTTP to HTTPS."
  7. Click on the "Create Distribution" button to create the CloudFront distribution.
  8. Once our CloudFront distribution is created, we can access our resume securely using the distribution URL. We can find the distribution URL in the AWS CloudFront console.

## Step 5: Request a certificate from ACM
  1. Open the AWS Management Console and navigate to the ACM service.
  2. Click on the "Request a certificate" button.
  3. In the "Add domain names" field, enter the domain name that we want to use for our CloudFront distribution (e.g. name.resume.example.com).
  4. Select the "DNS validation" method and click on the "Review and Request" button.
  5. Review our certificate request details and click on the "Confirm and request" button to submit our request.

## Step 6: Add records to the hosted zone
Once our certificate request is approved, we can add records to the hosted zone by following these steps:
  1. Navigate to the Amazon Route 53 service in the AWS Management Console.
  2. Select the hosted zone that we want to add records to.
  3. Click on the "Create Record Set" button.
  4. In the "Name" field, enter the domain name that we want to associate with our CloudFront distribution (e.g. name.resume.example.com).
  5. In the "Type" field, select "A - IPv4 address".
  6. In the "Alias" field, select "Yes".
  7. In the "Alias Target" field, select the CloudFront distribution that we created earlier.
  8. Click on the "Create" button to create the record set. Then check the status until it says “INSYNC”.
Note: It may take some time for the changes to propagate across the DNS system, so we may need to wait a while before the changes take effect.

## Step 7: The cost of our project
Now, let's calculate the cost of this project per month and per year. Here are the components that we will be charged for:
  •	S3 storage: $0.023 per GB-month (first 50 TB)
  •	S3 data transfer: $0.09 per GB (first 10 TB)
  •	CloudFront data transfer: $0.085 per GB (first 10 TB)
Assuming our resume is 1 MB in size and we get 100 visitors per month who view our resume once, here is the estimated cost per month:
  •	S3 storage: $0.023 (1 MB * 30 days) = $0.69
  •	S3 data transfer: $0.09 (1 MB * 100 visitors) = $9
  •	CloudFront data transfer: $0.085 (1 MB * 100 visitors) = $8.50
  •	Total: $18.19
Assuming we have the same traffic throughout the year, here is the estimated cost: $18.19 x 12 = $218.28



  
  
