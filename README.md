# EC2 Instance Deployment with CloudWatch: Setup and Optimization

## Overview

In previous deployments, we used AWS Elastic Beanstalk to handle provisioning infrastructure and deploying our application. Elastic Beanstalk is a PaaS (Platform as a Service) that abstracts away managing EC2 instances, auto-scaling, load balancing, etc.

However, for this deployment, we wanted more control over the server configuration and infrastructure. So we provisioned an EC2 instance directly, installed dependencies like Nginx, configured the networking, set up monitoring with CloudWatch, and deployed our application code.

Moving to EC2 allows us to customize the server and architecture as needed. The tradeoff is we now have to handle tasks like scaling and high availability instead of relying on Beanstalk services.

## Requirements

For this deployment, you'll need the following:

- AWS account
- EC2 instance
- Git
- Python
- Jenkins
- Nginx

## Steps

### Setup a virtual environment

Set up the virtual environment to run Jenkins:

1. Create an AWS account if you don't have one.

2. Create a new public subnet in AWS and launch EC2 instance. I used t2.medium to have sufficient computing resources.

3. Configure the security group to open necessary ports - 80, 8080, 8000, 22. This allows access to the various services.

4. Install [Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)

5. Install Python and Pip:

    ```bash
    sudo apt install python3.10-venv python3-pip -y
    ```

6. Install and set up [Jenkins](https://www.jenkins.io/doc/book/installing/linux/#debianubuntu) on the EC2 instance. Jenkins will be used to automate builds and deployments.

    6a. Install `Pipeline Keep Running Step` plugin to keep processes running between builds.

7. Install [Nginx](https://www.nginx.com/blog/setting-up-nginx/) web server and update the default config file `/etc/nginx/sites-enabled/default` as follows:

    ```nginx
    server {
        listen 5000 default_server;
        listen [::]:5000 default_server;
        # We changed the port from 80 to 5000

        # Also scroll down to where you see “location” and replace it with the text below:
        location / {
            proxy_pass http://127.0.0.1:8000;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }

    ```
This configures Nginx as a reverse proxy to forward requests to the application server running on port 8000.


### Configure Monitoring

Since my infrastructure is hosted in AWS, I decided to use [CloudWatch](https://aws.amazon.com/cloudwatch/) for monitoring and observability.

**Benefits:**

- CloudWatch is a native AWS service and integrates well with EC2.
- Server metrics are collected automatically without additional configuration.
- Cost-effective and easy to get started.

**Alternatives:** 
I also considered using [Datadog](https://www.datadoghq.com/) which offers advanced dashboards, integration with many services, and robust analytics. However, for this simple infrastructure, Datadog may have been a bit much. In the future, as the infrastructure grows, Datadog may be worth investigating.

To set up CloudWatch:

1. Install the [CloudWatch agent](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Install-CloudWatch-Agent.html) on the EC2 instance.

2. Run the configuration wizard to set up the agent.

![Screen Shot 2023-10-01 at 3 12 10 PM](https://github.com/belindadunu/deployment4/assets/139175163/dadc005a-862f-46fa-bd84-d9d5442b5810)

3. Start the CloudWatch agent and confirm it is collecting metrics.

    ```bash
    sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -s -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json
    sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -m ec2 -a status
    ```
![Screen Shot 2023-10-01 at 3 15 44 PM](https://github.com/belindadunu/deployment4/assets/139175163/0f297406-0d63-47c2-a9df-33de09bddaaf)

### Configure Jenkins Pipeline

1. Created a multibranch  pipeline job in Jenkins.

2. Configured GitHub webhook to trigger pipeline on commits.

3. Setup branch sources to scan for Jenkinsfiles.

4. Cloned the repository locally and created a new branch.

5. Made changes to the Jenkinsfile in the feature branch.

6. Committed the changes and pushed the feature branch to GitHub.

7. Created a pull request and merged the feature branch into the main branch.

8. Pushed the updated main branch to GitHub to trigger a build.

### Evaluate Server Performance
During the deployment and builds, I monitored the server performance using tools like htop and CloudWatch metrics:

- Check the application by accessing the EC2 public IP on port 8000 in a web browser:
  `<your-public-ip>:8000`

- Monitor the server's performance, including CPU usage, during build runs and application usage.

![Screen Shot 2023-10-01 at 4 37 31 PM](https://github.com/belindadunu/deployment4/assets/139175163/824eb3a5-27e6-447a-9283-b1addf5dd0e9)


- CPU usage stayed under 50% during builds.

- Application deployed successfully and CloudWatch metrics provided visibility into server resource utilization.

Based on this, the t2.medium instance seems able to handle the current load.

<img width="1416" alt="Screen Shot 2023-10-01 at 6 55 43 PM" src="https://github.com/belindadunu/deployment4/assets/139175163/9b0221e7-0251-4f56-bf0c-57baeb3fc76b">

### CPU Usage Alerts

I configured a CloudWatch alarm to notify me when CPU usage exceeds 15% for 5 minutes.

![Screen Shot 2023-10-01 at 5 16 10 PM](https://github.com/belindadunu/deployment4/assets/139175163/3287db7f-1f58-40ba-ab33-b01a88010a58)

When running parallel builds, I received an SNS email alert as the CPU spiked above this threshold to around 18%.
![Screen Shot 2023-10-01 at 5 02 11 PM](https://github.com/belindadunu/deployment4/assets/139175163/ef5a9fdc-936a-4ca0-bac2-4c78df293079)
![Screen Shot 2023-10-01 at 4 58 17 PM](https://github.com/belindadunu/deployment4/assets/139175163/feb8e851-c546-470a-a9fb-5ddc209fc7bc)

This shows the alert is functioning as expected to catch high CPU usage.

### T2 Micro Considerations

A t2.micro may struggle with the current deployment, as the limited CPU could cause:

- Long build times due to lack of resources.
- Performance issues under even moderate loads.
- Stability problems when multiple processes compete for CPU.

### Configure Email Notifications in Jenkins

To set notifications in Jenkins:

1. In "Manage Jenkins" > "Configure System", scroll to "Jenkins Location".
- Enter in email under "System Admin e-mail address.
- Scroll to the bottom, select "Apply" then "Save".

2. Add a post-build action in Jenkinsfile, or in the [Post-build Actions](https://plugins.jenkins.io/email-ext/#plugin-content-pipeline-step) section, click on Add post-build action and then select Editable Email Notification.

This sends email alerts when builds fail due to the post-build action.

## Issues

### Error installing Jenkins - "GPG key retrieval failed"
- Initially got "GPG key retrieval failed" error during Jenkins apt installation.
- Troubleshooting showed the Jenkins apt repo changed for a recent release.
- Fixed by purging old Jenkins cached packages, installing latest Java version, re-adding the Jenkins repo, and reinstalling.
- I resolved this by removing Jenkins cache, re-installing Java versions, and reinstalling the packages.

### Inaccessible application
- After deployment, attempts to access the application failed.
- Debugging revealed the EC2 security group was still blocking port 8000.
- Updated the security group rules to open ingress to port 8000.
- The application became accessible again after the security group change.

### CloudWatch agent setup
- CloudWatch metrics were not showing up in the console after installing the agent.
- Realized the agent didn't have the required IAM permissions to send metrics.
- Attached the CloudWatchAgentServerPolicy managed policy for the instance role.
- Agent started pushing metrics after several minutes.

![Screen Shot 2023-10-01 at 3 08 52 PM](https://github.com/belindadunu/deployment4/assets/139175163/2cb19fe6-a752-495c-9a0c-48a60daf9f79)
![Screen Shot 2023-10-01 at 3 09 57 PM](https://github.com/belindadunu/deployment4/assets/139175163/8af23e6e-ca87-4c3e-80b1-02e58d2e92a3)
![Screen Shot 2023-10-01 at 3 04 32 PM](https://github.com/belindadunu/deployment4/assets/139175163/d7440197-3e83-4ca3-b6fe-ed20c6f095ec)

### ChatGPT Usage
- I leveraged ChatGPT AI assistant to provide assistance in finding out if my CloudWatch configuration was properly created on my EC2. I was provided with the following explanation and breakdown:

`sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -s -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json`

- `sudo`: Run the command with superuser privileges.
- `/opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl`: Path to the CloudWatch agent control script.
- `a fetch-config`: Action to fetch the configuration.
- `m ec2`: Specifies the machine type, in this case, an Amazon EC2 instance.
- `s`: Indicates that the configuration should be stored locally on the instance.
- `c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json`: Specifies the path to the configuration file. In this case, the configuration is fetched from the provided JSON file located at /opt/aws/amazon-cloudwatch-agent/bin/config.json.

This command will grab the CloudWatch agent configuration specific to the EC2 instance from the specified JSON file and apply it to the local configuration of the agent. This allows the instance to start sending metrics and logs to Amazon CloudWatch for monitoring and analysis.

## System Diagram

![Dep_4 drawio](https://github.com/belindadunu/deployment4/assets/139175163/07316a43-e848-4e99-8918-7b065a2cbff3)

## Conclusion

In this deployment, we were able to walk through setting up infrastructure and deploying an application on EC2 instead of using Elastic Beanstalk. Key steps included:

- Creating an EC2 instance and configuring networking/security groups.
- Installing dependencies like Python, Git, Jenkins, and Nginx.
- Setting up Jenkins for CI/CD.
- Configuring CloudWatch monitoring and alerts.
- Deploying the application with Gunicorn and Nginx.
- Automating deployments through Jenkins pipelines.
- Moving from Beanstalk to EC2 provided more control and customization, at the cost of manual configuration. CloudWatch enabled monitoring of the server and application.

While this architecture is functional, there are further improvements that could be made:

- Using auto-scaling groups to handle fluctuations in traffic.
- Implementing infrastructure-as-code tools like Terraform.
- Integrating a Content Delivery Network (CDN) to serve static assets and reduce latency for users.
- Creating an AMI to launch pre-configured EC2 instances.
- Setting up DNS and TLS for secure remote access.

Overall, this project demonstrated core practices of automation, infrastructure provisioning, and monitoring. 
