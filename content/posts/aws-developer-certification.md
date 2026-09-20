---
title: 'Aws Developer Certification'
date: 2024-04-16T09:50:09-04:00
draft: false
tags:
  - cloud
  - aws
categories: notes
keywords:
  - aws
  - certification
---

AWS Certified Developer - Associate showcases knowledge and understanding of core AWS services, uses, and basic AWS architecture best practices, and proficiency in developing, deploying, and debugging cloud-based applications by using AWS. Preparing for and attaining this certification gives certified individuals more confidence and credibility. Organizations with AWS Certified developers have the assurance of having the right talent to give them a competitive advantage and ensure stakeholder and customer satisfaction.

> To learn more about the "AWS Certified Developer - Associate" exam refer [https://aws.amazon.com/certification/certified-developer-associate/](https://aws.amazon.com/certification/certified-developer-associate/)

> To check if there is any upcoming changes in any AWS exam refer [https://aws.amazon.com/certification/coming-soon/](https://aws.amazon.com/certification/coming-soon/)

## Exam Overview

- **Level** Associate
- **Length** `130 minutes` to complete the exam.
- **Cost** `150 USD`
- **Format** `65 questions`, either multiple choice or multiple responses.
- **Delivery method** Pearson VUE testing center or online proctored exam.
- **Passing** `~72%` _\*\*could be changed._

> [Download Exam Guide](https://d1.awsstatic.com/training-and-certification/docs-dev-associate/AWS-Certified-Developer-Associate_Exam-Guide.pdf) | [Download Sample Questions](https://d1.awsstatic.com/training-and-certification/docs-dev-associate/AWS-Certified-Developer-Associate_Sample-Questions.pdf)

## Who should take this exam?

AWS Certified Developer - Associate is a great starting point on the AWS Certification path for individuals who may have any of the following:

- Experience working in a developer role with in-depth knowledge of at least one high-level programming language.
- Experience in AWS technology.
- Strong on-premises IT experience and understanding of mapping on-premises to the cloud.
- Experience working in other cloud services.

## Personal Notes

- **Code** `DVA-C01` / `DVA-C02 (latest)`, both codes have the same syllabus and only the module distribution has changed.
- **Average Time** `2 mins / question`
- **Types of Questions** Multiple Choice and Multiple Responses

### Exam Breakdown

The whole exam is broken down into domains and sub-domains.

- **Deployment** `~24%` roughly `15-16 Questions`
- **Security** `~26%` roughly `16-17 Questions`
- **Development** `~32%` roughly `20-21 Questions`
- **Monitoring and Troubleshooting** `~18%` roughly `11-12 Questions`

### Recommended Whitepaper

- [Architecting for the Cloud: AWS Best Practices](https://aws.amazon.com/blogs/aws/new-whitepaper-architecting-for-the-cloud-best-practices/)
- [Practicing Continuous Integration and Continuous Delivery on AWS Accelerating Software Delivery with DevOps](https://docs.aws.amazon.com/whitepapers/latest/practicing-continuous-integration-continuous-delivery/welcome.html)

## Introduction to Elastic Beanstalk

![Elastic Beanstalk Icon](./img/elastic-beanstalk-icon.png)

It is a PaaS that allows you to quickly deploy and manage web apps on AWS **without worrying about the underlying infrastructure**.

### What is Platform as a Service? (PaaS)

A platform allowing customers to develop, run, and manage applications without the complexity of building and maintaining the infrastructure typically associated with developing and launching an app.

PaaS is like renting a pre-built cloud workspace for developing and running applications. It saves time and money and allows for easy scaling.

---

It is not recommended for _"Production"_ applications. Talking about enterprise, large companies.

Powered by a CloudFormation template setup for you:

- Elastic Load Balancer
- Autoscaling Groups
- RDS Database
- EC2 Instance pre-configured (or custom) platforms
- Monitoring (CloudWatch, SNS)
- In-place and Blue/Green deployment methodologies
- Security (Rotates Passwords)
- Can run **Dockerize** environments

### Supported Languages

- Ruby
- Python
- PHP
- Tomcat
- NodeJS

### Web vs. Worker Environment

#### Web Environment

- Has two variants
  - **Load Balanced Environment**
    - Uses ASG and set to scale
    - Use an ELB
    - Designed to Scale
    - Variable cost associated based on load
  - **Single-Instance Env**
    - Still uses an AGS, but Desired Capacity is set to 1 to ensure the server is always running.
    - No ELB to save on cost.
    - The Public IP Address has to be used to route traffic to the server.
- Creates an ASG (Auto Scaling Group)
- Creates an ELB (Elastic Load Balance) - optional

![web environment](./img/elastic-beanstalk-web-environment-types.png)

#### Worker Environment

- For backend jobs
- Creates and ASG
- Creates and SQS Queue
- Installs the SQSD daemon on the EC2 Instances
- Create CloudWatch Alarm to dynamically scale instances based on health.

![worker environment](./img/elastic-beanstalk-worker-environment.png)

### Deployment Policies

These are the deployment policies available with the Elastic Beanstalk

| Deployment Policy             | Load Balanced Env | Single Instance Env |
| ----------------------------- | ----------------- | ------------------- |
| All at Once                   | 👍                | 👍                  |
| Rolling                       | 👍                | ❌                  |
| Rolling with additional batch | 👍                | ❌                  |
| Immutable                     | 👍                | 👍                  |

#### All at Once

![All at Once Deployment](./img/elastic-beanstalk-all-at-once-deplyment.png)

- Deploy the new app version to all instances at the same time.
- Takes all instances **out of service** during the deployment process.
- Servers become available again

> The **fastest** but also the most **dangerous** deployment method.
> *In case of Failur*e, you need to roll back the changes by re-deploying the original version again to all instances.

#### Rolling

![Rolling Deployment](./img/elastic-beanstalk-rolling-deplyment.png)

- Deploys the new app version to a batch of instances at a time.
- Takes batch instances **out of service** while the deployment processes.
- Reattaches updated instances.
- Goes onto the next batch, taking them out of service.
- Reattaches those instances (rinse and repeat)

> _In Case of Failure_, You need to perform an additional rolling update in order to roll back the changes.

#### Rolling with Additional Batch

Rolling with additional batch ensure our capacity is never reduced. This is important for applications where a reduction in capacity could cause availability issues for users.

- Launch a new instance that will be used to replace a batch.
- Deploy update app version to new batch.
- Attach the new batch and terminate the existing batch.

> _In case of failure_, you need to perform an additional rolling update to roll back the changes.

#### Immutable

![Immutable](./img/elastic-beanstalk-immutable-deplyment.png)

- Create a new ASG with EC2 instances.
- Deploy the updated version of the app on the new EC2 instances.
- Point the ELB to the new ASG and delete the old ASG, which will terminate the old EC2 instances.

> The **safest way** to deploy for critical applications. _In case of failure_ just terminate the new instances since the existing instances still remain.

### EB - Deployment Methods

| Method                            | Impact of failed deployment                                                                        | Deploy time | No downtime | No DNS change | Rollback progress | code deployed to Instances |
| --------------------------------- | -------------------------------------------------------------------------------------------------- | ----------- | ----------- | ------------- | ----------------- | -------------------------- |
| **All at once**                   | Downtime                                                                                           | ⏰          | ❌          | 👍            | Manual            | Existing                   |
| **Rolling**                       | Single batch out of service; any successful batches before failure running new application version | ⏰⏰ \*     | 👍          | 👍            | Manual            | Existing                   |
| **Rolling with additional batch** | Minimal if first batch fails; otherwise, similar to Rolling                                        | ⏰⏰⏰ \*   | 👍          | 👍            | Manual            | New and Existing           |
| **Immutable**                     | Minimal                                                                                            | ⏰⏰⏰⏰    | 👍          | 👍            | Terminate New     | New                        |
| **Blue/Green**                    | Minimal                                                                                            | ⏰⏰⏰⏰    | 👍          | ❌            | Swap URL          | New                        |

\* Time may vary

### EB - In Place vs. Blue/Green Deployment

Elastic Beanstalk, by default, performs in-place updates.

> 👋🏼 In-Place and Blue/Green Deployment are not definitive in definition and the context can change the scope of what they mean.
