# Jenkins pipelines in AWS

- Create a new user: `Jenkins`
- Create a new group: `Jenkins`
- Create a new policy: `EC2S3CloudwatchFull`

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "VisualEditor0",
      "Effect": "Allow",
      "Action": ["s3:*", "ec2:*", "cloudwatch:*"],
      "Resource": "*"
    }
  ]
}
```

- Add the user `Jenkins` to the group `Jenkins` and attach the new policy `EC2S3CloudwatchFull`

- Create S3 bucket for static website. Example: `jenkins-test-123456`
- Edit public access settings of the bucket: un-check Block all public access and save
- Go to Properties and set Static website hosting (Use this bucket to host a website)
- Set the following policy for your bucket
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": [
                "s3:GetObject"
            ],
            "Resource": [
                "arn:aws:s3:::jenkins-test-123456/*"
            ]
        }
    ]
}
```
- You should see: **This bucket has public access**
- Example Static Website URL: http://jenkins-test-123456.s3-website-us-east-1.amazonaws.com/

- Launch an Ubuntu (t2.micro) EC2 instance.
- In the Security Group, Inbound, allow...
  - Port 22 (SSH) for Your IP
  - Port 8080 (Jenkins) for Your Ip
- Install JAVA Development Kit and Jenkins

```shell
ssh -i "JenkinsKP.pem" ubuntu@YOUR-EC2-PUBLIC-IP-OR-DNS
sudo apt-get update
sudo apt install -y default-jdk
wget -q -O - https://pkg.jenkins.io/debian/jenkins.io.key | sudo apt-key add -
sudo sh -c 'echo deb https://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
sudo apt-get update
sudo apt-get install -y jenkins
```

- Go to YOUR-EC2-PUBLIC-IP-OR-DNS:8080/
- Paste the code you will obtain as the result of executing `/var/lib/jenkins/secrets/initialAdminPassword` in your EC2
- This is going to be the password to log into Jenkins. The user, `admin`
- Install the following plugins (... either through the `UI` or using `Jenkins CLI`)

  - Blue ocean
  - Common API for Blue Ocean
  - Config API for Blue Ocean
  - Dashboard for Blue Ocean
  - Events API for Blue OceAN
  - Git Pipeline for Blue Ocean
  - GitHub Pipeline for Blue Ocean
  - Pipeline implementation for Blue Ocean
  - Blue Ocean pipeline editor
  - Display URL for Blue Ocean
  - Blue Ocean Executor info

- Re-start Jenkins: `sudo systemctl restart jenkins`

- Create a github repository for your code, `JenkinsPipelineStaticSiteS3` (in this example the repository is `jenlkins`)
- Inside your repository...
  - Create a sample `index.html`

```html
<!DOCTYPE html>
<html>
  <head>
    <title>I'm a title!</title>
  </head>
  <body>
    <h1>Hello World!</h1>
  </body>
</html>
```

  - Create a Jenkins file: `Jenkinsfile`

```
pipeline {
  agent any
  stages {
    stage('Build') {
      steps {
        sh 'echo "Hello World"'
      }
    }

    stage('Lint HTML') {
      steps {
        sh 'tidy -q -e *.html'
      }
    }

    stage('Upload to AWS') {
      steps {
        withAWS(region: 'us-east-1', credentials: 'MyCredentials') {
          s3Upload(pathStyleAccessEnabled: true, payloadSigningEnabled: true, file: 'index.html', bucket: 'jenkins-test-123456')
        }

      }
    }

  }
}
```
Note: You can omit or delete the Build stage.

- Install `tidy` for linting: `sudo apt install tidy`
- Install `Pipeline: AWS Steps` Jenkins's plugin to interact with AWS API

- Re-start Jenkins: `sudo systemctl restart jenkins`

- Go to Blue Ocean > New pipeline > GitHub
- Go to https://github.com/settings/tokens and generate a token
- Paste the token
- Select the repository. Example: `JenkinsPipelineStaticSiteS3`
- Create Pipeline

At this point, build and lint stages should be successful, but... Upload to AWS will be red (aka, failing).

- Install ansible: `sudo apt install ansible`
- Install pip: 
```shell
curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
python get-pip.py
sudo apt install python-pip
```
- Install boto: `sudo pip install boto`
- We are going to install the Jenkins's plugin `CloudBees Credentials`
- Now we have to configure the our `AWS credentials`, the ones tied to our IAM user `Jenkins`. Go to Credentials > Global > Add credentials.
- Select AWS credentials
- And set an ID: MyCredentials
- Access Key ID: YOUR-AWS-KEY-ID
- Secret Access KEY: YOUR-AWS-SECRET-ACCESS-KEY
- Now go to you pipeline (example URL: http://54.81.200.78:8080/blue/organizations/jenkins/jenkins/activity) and run it again (top replay icon)
- Everything should be `green`

- To test the linter, remove `<!DOCTYPE html>` tag. Commit, push and in Blue Ocean re-run the pipeline.
- Click on edit button, then Save, add your message ('removing <!DOCTYPE html> to force linting error') and Save & run. 