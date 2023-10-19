# Setting up Mockaroo Enterprise

## License

[Mockaroo Enterprise License](/mockaroo/mockaroo-enterprise/blob/master/LICENSE.md)

## Getting Access

Mockaroo Enterprise is distributed as a docker repository on Amazon ECR. Please let us know the AWS account ID under which Mockaroo will be installed. We need to grant permission to that account specifically.  Multiple accounts are supported if needed.

## Installation Requirements

Mockaroo can be installed in your private cloud as a docker image.  [Contact support for pricing.](https://mockaroo.com/comments/new)

Mockaroo requires the following cloud services:

* Amazon S3

Mockaroo also requires the following 3rd party software:

* Redis v6
* Postgres v13
* An email service such as Amazon SES or Sendgrid

## Architecture

Here is what Mockaroo looks like when deployed on AWS:

![Mockaroo Architecture](https://docs.google.com/drawings/d/e/2PACX-1vSTzZOirq7k4rL0azIovUguL47u49wRL3yZFg7v7j3Eqpf5quBzi9gbKEMtMsSliFBD4bxKcmZ4c7Q5/pub?w=569&h=577)

Note that the only required component from AWS is S3. You can use any other cloud provider to host the Mockaroo database, docker containers, load balancer, and Redis.

## Docker Image

The Mockaroo docker image is distributed via Amazon's Elastic Container Eegistry (ECR).  The URI for the image is:

```
622045361486.dkr.ecr.us-west-2.amazonaws.com/mockaroo-enterprise
```

## Redis

Mockaroo uses Redis for caching content and queuing data generation jobs. 

### Docker

To run Redis as a docker image:

```
docker run -d --name redis -p 6379:6379 redis
```

### Amazon ElastiCache

If you're using AWS the easiest way to provide Redis to Mockaroo is to create a Redis cluster using Amazon ElasticCache.

* Be sure to create the cluster in the same VPC where the EC2 instances running Mockaroo will reside.
* You can use a very small instance type as Mockaroo does not send much traffic to Redis. For example, cache.t2.small.

As an alternative, you can also run Redis natively or using docker if you don't want to use ElastiCache.

### Google Memorystore

You can use Google Cloud Platform's redis-compatible Memorystore service to host redis:

https://cloud.google.com/memorystore

## Postgres

### Amazon RDS

To host Mockaroo's database on Amazon RDS, create a postgres database called "mockaroo".  Remember the username and password.  You'll need to configure those as environment variables later.

### Google Cloud SQL

To host Mockaroo's database on Google Cloud SQL, create a postgres database called "mockaroo".  Remember the username and password.  You'll need to configure those as environment variables later.

https://cloud.google.com/sql

## Amazon S3 Bucket

Amazon S3 is required to run Mockaroo. Create an Amazon S3 bucket.  You'll configure the name as an environment variable later.
In order for Mockaroo to upload files to this bucket, you can either configure AWS_ACCESS_KEY and AWS_SECRET_KEY environment variables (See "App Container" below), or assign an IAM role to the EC2 instance(s) on which Mockaroo run that can write to the S3 bucket.  Here a guide that describes how to do this: [Enable S3 access from EC2 by IAM role](https://cloud-gc.readthedocs.io/en/latest/chapter03_advanced-tutorial/iam-role.html)

## Email

Mockaroo sends emails when users need to reset their password or have a file ready to download. 

### Amazon SES

To use Amazon SES for sending emails:

1. Under Identity Management > Domains, add the domain on which Mockaroo will be hosted.  You will later set this as the MOCKAROO_DOMAIN environment variables.
2. Under Identity Management > Emails, add a "no-reply@(your domain)" email address.
3. Under Email Sending > SMTP Settings, create your SMTP credentials.  You will use these to set the MAIL_HOST, MAIL_USERNAME, and MAIL_PASSWORD environment variables.

```
MAIL_HOST=(your SES email host, typically something like "email-smtp.us-west-2.amazonaws.com")
MAIL_USERNAME=(your SES email username)
MAIL_PASSWORD=(your SES email password)
```

## Mockaroo App and Worker Instances

Mockaroo provides two types of services:

* app - The web front-end
* worker - Data generation workers - When we need to generate large volumes of data quickly, this is what we'll scale

### CPU and Memory Requirements

|Instance Type|CPUs|Memory per CPU|HD Storage Capacity|
|-------------|----|--------------|-------------------|
|app|8 or more|8GB per CPU|At least 50GB per VM|
|worker|8 or more|4GB per CPU|At least 50GB per VM|

We suggest running at least 2 separate docker containers: one for the app and one for workers.

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

### App Instance

To run the Mockaroo web app, the first step is to create an app.env file...

```
# You'll need to configure these with your own values:
REDIS_URL=redis://(your redis hostname):6379/
REDIS_WORKERS=(The number of data generation workers you are running.  Can be from 1 to 32)
DB_USERNAME=(database username)
DB_PASSWORD=(database password)
DB_HOSTNAME=(hostname of your amazon rds instance)
AWS_REGION=(The region where your S3 bucket exists, e.g. us-east-1)
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
MOCKAROO_WORKERS=web=1,default=1
```
... then, run following to initialize the database ...

```
docker run --env-file app.env mockaroo/mockaroo-enterprise rake db:create && rake db:schema:load && rake db:seed
```

Finally, run the following to start the mockaroo web app on port 3000 (or any port you like):

```
docker run -d --name mockaroo --env-file app.env -p 3000:3000 mockaroo/mockaroo-enterprise
```

### Worker Instance

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

### Domains

You can choose any domain name you like for hosting Mockaroo Enterprise. In addition to creating a DNS entry to map your chosen domain to Mockaroo, you should also create a DNS entry for `my.api.(your-mockaroo-domain)` and `api.(your-mockaroo-domain)`. Make sure the TLS certificate you assign to Mockaroo contains all three domains.

### AWS

1. Create a target group called "mockaroo".
2. Add your Mockaroo app instance(s) to the target group
3. Create listeners on port 80 and 443, forwarding to the mockaroo target group.
4. Create and assign a certificate for the HTTPs listener.
5. Create an Elastic IP and assign it to your load balancer.
6. Create an A record via Route53 pointing (your-mockaroo-domain) to your Elastic IP.
7. Create CNAME records in Route53 that point my.api.(your-mockaroo-domain) and api.(your-mockaroo-domain) to (your-mockaroo-domain).

### GCP

Set up a GCP [load balancer](https://cloud.google.com/load-balancing/docs) and [Google Managed SSL Cert](https://cloud.google.com/load-balancing/docs/ssl-certificates/google-managed-certs), fowarding all traffic on ports 80 and 443 to your Mockaroo app instance(s).

## Limiting Sign Ups to Certain Email Domains

To limit signups to certain email domains, add an environment variable called `MOCKAROO_VALID_ACCOUNT_DOMAINS` to your app.env file. The value should be a comma delimited list of email domains to accept (without spaces between the values). For example:

```
MOCKAROO_VALID_ACCOUNT_DOMAINS=domain.com,mydomain.com
```

# Admin Mode

## Granting Admin Rights

Admins can view in progress data generation jobs for all users, and if necessary, cancel in progress jobs. To grant a user admin rights, run the following SQL query:

```sql
update users
set admin=true
where email = '<a user's email address>'
```

To view in progress jobs, go to "Admin - Downloads" in the user menu when signed in as an admin user.

## Impersonating Users

Admins can impersonate users to help troubleshoot and fix problems. To impersonate a user, select "Impersonate User" from the user menu in the upper right. Enter the email address of the user that you want to impersonate and click "Impersonate". Just remember to click "Stop Impersonating" in the user menu when you are done!

## Logs

Within both app and worker instances, application logs can be found at /app/log.

You can mount these as local volumes using docker compose like so:

```yaml
  app:
    image: "mockaroo-enterprise"
    volumes:
      - /var/log/mockaroo/app:/app/log
    ...
  worker:
    image: "mockaroo-enterprise"
    volumes:
      - /var/log/mockaroo/worker:/app/log
    ...
```

## Upgrades

When an upgrade is available, [pull the latest mockaroo-enterprise docker image](https://github.com/mockaroo/mockaroo-enterprise#pulling-the-image-from-amazon-ecr), then run:

```
docker run --env-file app.env 622045361486.dkr.ecr.us-west-2.amazonaws.com/mockaroo-enterprise rails db:migrate
```

Then, redeploy your app and worker containers

