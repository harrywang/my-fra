# Deploying a Flask and React Microservice to AWS ECS

- setup IAM users ("Attach existing policies directly" with "Administrator Access" and "Billing")
to view billing - two steps:
https://aws.amazon.com/blogs/security/dont-forget-to-enable-access-to-the-billing-console/
- configure AWS CLI

```
$ pip3 install awscli
$ aws --version
aws-cli/1.18.39 Python/3.7.7 Darwin/19.4.0 botocore/1.15.39

```

https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html

The AWS CLI uses two files to store the sensitive credential information (in ~/.aws/credentials) separated from the less sensitive configuration options (in ~/.aws/config).


default region: https://docs.aws.amazon.com/general/latest/gr/rande.html


- US East (N. Virginia), code: us-east-1, slightly cheaper than singapore
- Asia Pacific (Singapore), code `ap-southeast-1`

```
$ aws configure
AWS Access Key ID [None]: A6NxxxxxZDKKB
AWS Secret Access Key [None]: VLxxxx2fZWhr
Default region name [None]: us-east-1
Default output format [None]: json

$ ls ~/.aws
config		credentials

$ aws s3 ls
```

Navigate to Amazon ECS, click "Repositories", and then add two new repositories:

test-driven-users
test-driven-client

or use command line

`$ aws ecr create-repository --repository-name REPOSITORY_NAME --region us-west-1`

Build the images locally with tags (dev in this case) - change the ids from ECR:

<img width="936" alt="Screen Shot 2020-04-09 at 3 19 13 PM" src="https://user-images.githubusercontent.com/595772/78932638-e5008400-7a75-11ea-94ed-1219887cc473.png">



```
docker build \
  -f services/users/Dockerfile \
  -t <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/test-driven-users:dev \
  ./services/users

  docker build \
    -f services/client/Dockerfile \
    -t <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/test-driven-client:dev \
    ./services/client
```


authenticate the Docker CLI to use the ECR registry:

https://docs.aws.amazon.com/cli/latest/reference/ecr/get-login-password.html

directly pipe the password to the `docker login`:

```
$   aws ecr get-login-password \
>     --region us-east-1 \
> | docker login \
>     --username AWS \
>     --password-stdin <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com
Login Succeeded

```

`docker image ls` shows the images built above is about 400M - 500M in size. so the following two commands take a while to finish (a few minutes):

```
docker push <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/test-driven-users:dev
 docker push <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/test-driven-client:dev
 ```

 create/revise the following files:

 - services/users/Dockerfile.prod
  - services/client/Dockerfile.prod
 - services/client/conf/conf.d/default.conf
 - docker-compose.prod.yml
 - services/users/project/db/create.sql

 Then,

```
 $ docker-compose down
 $ export REACT_APP_USERS_SERVICE_URL=http://localhost:5001
  $ docker-compose -f docker-compose.prod.yml up -d --build
  $ docker-compose -f docker-compose.prod.yml exec users python manage.py recreate_db
  $ docker-compose -f docker-compose.prod.yml exec users python manage.py seed_db
```
test it out at http://localhost:3007/


Build and push the new images with the prod tag, making sure to use the --build-arg flag to pass in the appropriate arguments:

```
$ docker build \
  -f services/users/Dockerfile.prod \
  -t <AWS_ACCOUNT_ID>.dkr.ecr.us-west-1.amazonaws.com/test-driven-users:prod \
  ./services/users

$ docker build \
  -f services/client/Dockerfile.prod \
  -t <AWS_ACCOUNT_ID>.dkr.ecr.us-west-1.amazonaws.com/test-driven-client:prod \
  --build-arg NODE_ENV=production \
  --build-arg REACT_APP_USERS_SERVICE_URL=${REACT_APP_USERS_SERVICE_URL} \
  ./services/client

$ docker push <AWS_ACCOUNT_ID>.dkr.ecr.us-west-1.amazonaws.com/test-driven-users:prod

$ docker push <AWS_ACCOUNT_ID>.dkr.ecr.us-west-1.amazonaws.com/test-driven-client:prod
```

the production frontend image is significantly smaller!!

CodeBuild Setup project and


- a new IAM role is created `codebuild-my-fra-service-role`
- add buildspec.yml - don't forget the version information
-
