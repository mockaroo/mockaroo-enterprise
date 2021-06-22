# Setting up Mockaroo Enterprise

Mockaroo can be installed in your private cloud as a docker image.  [Contact support for pricing.](https://mockaroo.com/comments/new)

## Requirements

Mockaroo requires the following cloud services:

* Amazon S3

Mockaroo also requires the following 3rd party software:

* Redis
* Postgres
* An email provider

## Docker Image

The Mockaroo docker image is distributed via Amazon's Elastic Container Eegistry (ECR).  The URI for the image is:

```
622045361486.dkr.ecr.us-west-2.amazonaws.com/mockaroo-enterprise
```

## Setup

Mockaroo provides two types of services:

* app - The web front-end
* worker - Data generation workers - When we need to generate large volumes of data quickly, this is what we'll scale

I suggest running at least 2 separate docker containers: one for the app and one for workers.

### Pulling the image from Amazon ECR

Once the Mockaroo Enterprise repo has been shared with your AWS account, you'll need to ensure that the IAM user that you'll be using to pull the image has the required permissions.  These can be granted by applying the following IAM policy to a role to which the user has been assigned:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:DescribeImages"
      ],
      "Resource": "arn:aws:ecr:us-west-2:622045361486:repository/mockaroo-enterprise"
    }
  ]
 }
```

Once the user has been given the rights above, you can pull the Mockaroo Enterprise image using the following command:

```
aws ecr get-login-password --region us-west-2 | docker login --password-stdin --username AWS 622045361486.dkr.ecr.us-west-2.amazonaws.com/mockaroo-enterprise:latest
```

This will authenticate against the ECR repo with docker CLI by injecting an ECR token into docker from the output of the AWS CLI.  You should see the following output:

```
Login Succeeded
```

Then, pull the docker image:

```
docker pull 622045361486.dkr.ecr.us-west-2.amazonaws.com/mockaroo-enterprise:latest
```

### Redis

Mockaroo uses Redis for caching content and queuing data generation jobs. 

#### Docker

To run Redis as a docker image:

```
docker run -d --name redis -p 6379:6379 redis
```

#### Amazon ElastiCache

If you're using AWS the easiest way to provide Redis to Mockaroo is to create a Redis cluster using Amazon ElasticCache.

* Be sure to create the cluster in the same VPC where the EC2 instances running Mockaroo will reside.
* You can use a very small instance type as Mockaroo does not send much traffic to Redis. For example, cache.t2.small.

As an alternative, you can also run Redis natively or using docker if you don't want to use ElastiCache.

#### Google Memorystore

You can use Google Cloud Platform's redis-compatible Memorystore service to host redis:

https://cloud.google.com/memorystore

### Postgres

#### Amazon RDS

To host Mockaroo's database on Amazon RDS, create a postgres database called "mockaroo".  Remember the username and password.  You'll need to configure those as environment variables later.

#### Google Cloud SQL

To host Mockaroo's database on Google Cloud SQL, create a postgres database called "mockaroo".  Remember the username and password.  You'll need to configure those as environment variables later.

https://cloud.google.com/sql

### Amazon S3 Bucket

Amazon S3 is required to run Mockaroo. Create an Amazon S3 bucket.  You'll configure the name as an environment variable later.
In order for Mockaroo to upload files to this bucket, you can either configure AWS_ACCESS_KEY and AWS_SECRET_KEY environment variables (See "App Container" below), or assign an IAM role to the EC2 instance(s) on which Mockaroo run that can write to the S3 bucket.  Here a guide that describes how to do this: [Enable S3 access from EC2 by IAM role](https://cloud-gc.readthedocs.io/en/latest/chapter03_advanced-tutorial/iam-role.html)


### Email

Mockaroo sends emails when users need to reset their password or have a file ready to download. 

#### Amazon SES

To use Amazon SES for sending emails:

1. Under Identity Management > Domains, add the domain on which Mockaroo will be hosted.  You will later set this as the MOCKAROO_DOMAIN environment variables.
2. Under Identity Management > Emails, add a "no-reply@(your domain)" email address.
3. Under Email Sending > SMTP Settings, create your SMTP credentials.  You will use these to set the MAIL_HOST, MAIL_USERNAME, and MAIL_PASSWORD environment variables.

```
MAIL_HOST=(your SES email host, typically something like "email-smtp.us-west-2.amazonaws.com")
MAIL_USERNAME=(your SES email username)
MAIL_PASSWORD=(your SES email password)
```

### Mockaroo

#### App Container

To run the Mockaroo web app, the first step is to create an app.env file...

```
# You'll need to configure these with your own values:
REDIS_URL=redis://(your redis hostname):6379/
REDIS_WORKERS=(The number of data generation workers you are running.  Can be from 1 to 32)
DB_USERNAME=(database username)
DB_PASSWORD=(database password)
DB_HOSTNAME=(hostname of your amazon rds instance)
AWS_ACCESS_KEY=(your aws key - alternatively you can omit this and grant mockaroo access to your bucket via IAM)
AWS_SECRET_KEY=(your aws secret - alternatively you can omit this and grant mockaroo access to your bucket via IAM)
S3_BUCKET=(the name of the S3 bucket assigned to mockaroo)
MOCKAROO_ADMIN_EMAIL=(an email address where errors and daily reports should be sent)
MOCKAROO_DOMAIN=(the domain name on which your hosting mockaroo)  
MOCKAROO_MAIL_FROM=(the email address used when Mockaroo sends automated emails, defaults to "no-reply@{MOCKAROO_DOMAIN}")
MAIL_HOST=(your email host)
MAIL_USERNAME=(your email username)
MAIL_PASSWORD=(your email password)

# Optional configs
GOOGLE_AUTH_KEY=(optional, your google auth key if you'd like to allow users to log in with google)
GOOGLE_AUTH_SECRET=(optional, your google auth secret if you'd like to allow users to log in with google)

# In most cases you can leave these unchanged:
RAILS_ENV=production
RACK_ENV=production
DB_ADAPTER=postgresql
DB_NAME=mockaroo
DB_PORT=5432
MAIL_PORT=587
PORT=3001
MOCKAROO_QUICK_DOWNLOAD_LIMIT=10000
MOCKAROO_ENTERPRISE=true
MOCKAROO_ALLOW_ANONYMOUS_ACCESS=true
MOCKAROO_ALLOW_PASSWORD_AUTH=true
MOCKAROO_API_REQUEST_LIMIT=100000
MOCKAROO_API_RECORD_LIMIT=5000
MOCKAROO_DEFAULT_PLAN=Free
MOCKAROO_SERVE_STATIC_ASSETS=true
MOCKAROO_USE_SSL=true
MOCKAROO_DEFAULT_PLAN=Enterprise
REDIS_CLIENT_CONNECTIONS=16
REDIS_CONCURRENCY=20
REDIS_SERVER_CONNECTIONS=18
MOCKAROO_WORKERS=web=1
```
... then, run following to initialize the database ...

```
docker run --env-file app.env mockaroo/mockaroo-enterprise rake db:create && rake db:schema:load && rake db:seed
```

Finally, run the following to start the mockaroo web app on port 3000 (or any port you like):

```
docker run -d --name mockaroo --env-file app.env -p 3000:3000 mockaroo/mockaroo-enterprise
```

### Worker Container

To run the data generation workers, copy app.env to a new file called worker.env and replace this:

```
MOCKAROO_WORKERS=web=1
```

with this:

```
MOCKAROO_WORKERS=worker0=1,worker1=1,worker2=1,worker3=1,worker4=1,worker5=1,worker6=1,worker7=1
```

This will give you 8 concurrent data generation processes.  You can add more by adding additional workers `worker8=1, worker9=1`, etc... to `MOCKAROO_WORKERS` and increasing the following:

```
REDIS_CLIENT_CONNECTIONS=(2 x #workers)
REDIS_CONCURRENCY=(2 x #workers + 4)
REDIS_SERVER_CONNECTIONS=(2 x #workers + 2)
```

To start the worker container, run:

```
docker run -d --name worker --env-file worker.env mockaroo/mockaroo-enterprise
```

## Load Balancer

Even if you're running a single Mockaroo app instance, you'll need to put a load balancer in front of Mockaroo to provide TLS.

### AWS

1. Create a target group called "mockaroo".
2. Add your Mockaroo app instance(s) to the target group
3. Create listeners on port 80 and 443, forwarding to the mockaroo target group.
4. Create and assign a certificate for the HTTPs listener.
5. Create an Elastic IP and assign it to your load balancer.
6. Create an A record via Route53 pointing to your Elastic IP.

### GCP

Set up a GCP [load balancer](https://cloud.google.com/load-balancing/docs) and [Google Managed SSL Cert](https://cloud.google.com/load-balancing/docs/ssl-certificates/google-managed-certs), fowarding all traffic on ports 80 and 443 to your Mockaroo app instance(s).

## Limiting Sign Ups to Certain Email Domains

To limit signups to certain email domains, add an environment variable called `MOCKAROO_VALID_ACCOUNT_DOMAINS` to your app.env file. The value should be a comma delimited list of email domains to accept (without spaces between the values). For example:

```
MOCKAROO_VALID_ACCOUNT_DOMAINS=domain.com,mydomain.com
```

## Upgrades

When an upgrade is available, grab the latest mockaroo-enterprise docker image, then run:

```
docker run app.env mockaroo/mockaroo-enterprise:(version) rake db:migrate
```

Then, redeploy your app and worker containers

