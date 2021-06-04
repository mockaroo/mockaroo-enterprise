# Setting up Mockaroo Enterprise on Docker

Mockaroo can be installed in your private AWS cloud as a docker image.  [Contact support for pricing.](https://mockaroo.com/comments/new)

## Requirements

Mockaroo requires the following cloud services:

* Amazon RDS (Postgres)
* Amazon S3
* Amazone SES (optional)

Mockaroo also requires redis, which can be installed via the redis docker image.

## Docker Image

The Mockaroo docker image is distributed via Amazon's Elastic Container Eegistry (ECR).  The URI for the image is:

```
622045361486.dkr.ecr.us-west-2.amazonaws.com/mockaroo-enterprise
```

## Setup

Mockaroo provides three types of services:

* app - The web front-end
* api - The REST api
* worker - Data generation workers - When we need to generate large volumes of data quickly, this is what we'll scale

I suggest running at least 2 separate docker containers: one for app and api, and one for workers.

### Amazon RDS

Create a postgres database on Amazon RDS called "mockaroo".  Remember the username and password.  You'll need to configure those as environment variables later.

### Amazon S3 Bucket

Create an Amazon S3 bucket.  You'll configure the name as an environment variable later.
In order for Mockaroo to upload files to this bucket, you can either configure AWS_ACCESS_KEY and AWS_SECRET_KEY environment variables (See "App Container" below), or assign an IAM role to the EC2 instance(s) on which Mockaroo run that can write to the S3 bucket.  Here a guide that describes how to do this: [Enable S3 access from EC2 by IAM role](https://cloud-gc.readthedocs.io/en/latest/chapter03_advanced-tutorial/iam-role.html)

### Amazon SES for Email

Mockaroo sends emails when users need to reset their password or have a file ready to download. We recommend you use Amazon SES to send emails. To set up SES:

1. Under Identity Management > Domains, add the domain on which Mockaroo will be hosted.  You will later set this as the MOCKAROO_DOMAIN environment variables.
2. Under Identity Management > Emails, add a "no-reply@(your domain)" email address.
3. Under Email Sending > SMTP Settings, create your SMTP credentials.  You will use these to set the MAIL_HOST, MAIL_USERNAME, and MAIL_PASSWORD environment variables.

### Redis Docker Image

Run redis as a docker image:

```
docker run -d --name redis -p 6379:6379 redis
```

### App Container

To run the app and api services, the first step is to create an app.env file...

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
MAIL_HOST=(your SES email host, typically something like "email-smtp.us-west-2.amazonaws.com")
MAIL_USERNAME=(your SES email username)
MAIL_PASSWORD=(your SES email password)

# In most cases you can leave these unchanged:
RAILS_ENV=production
RACK_ENV=production
DB_ADAPTER=postgresql
DB_NAME=mockaroo
DB_PORT=5432
MAIL_PORT=587
PORT=80
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
docker run --env-file app.env mockaroo/mockaroo-enterprise:2.0.0 rake db:reset && rake db:seed
```

Finally, run the following to start the mockaroo web app on port 8080 (or any port you like):

```
docker run -d --name mockaroo --env-file app.env -p 8080:80 mockaroo/mockaroo-enterprise:2.0.0
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
docker run -d --name worker --env-file worker.env mockaroo/mockaroo-enterprise:2.0.0
```

## NGINX

You'll most likely want to put a webserver like NGINX in front of mockaroo so that users can connect securely over SSL.  If you use nginx, here are some options that you should enable:

```
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header Host $http_host;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_redirect off;
proxy_send_timeout 1000s;   # disable timeout - protects against complicated schemas timing out before flushing
proxy_read_timeout 1000s;   # disable timeout - protects against complicated schemas timing out before flushing
proxy_buffering off;        # enabled response streaming

client_max_body_size 20M;
```

## Upgrades

When an upgrade is available, grab the latest mockaroo-enterprise docker image, then run:

```
docker run app.env mockaroo/mockaroo-enterprise:(version) rake db:migrate
```

Then, redeploy your app and worker containers

