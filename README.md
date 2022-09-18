# aws-ecs-workshop-copilot

Deploying ECS workshop microservices (UI and services) application using copilot.
The deployment consists of multi containers.

This tutorial is based on the [AWS ECS Workshop](Source: https://ecsworkshop.com/)
but revised to use the local Linux environment instead of the Cloud9 workspace
used in the workshop.

![Microservices application](/images/architecture.jpg)

**NOTE:** the above picture shows three Availability Zones (AZ) but by default
the `copilot init` creates only two private subnets (i.e., one in each AZs)
and two private also one in each AZs. Therefore, the UI will show the traffic
arrows moving between the two subnets only. 

## Prerequisite

- Setup AWS CLI v2; follow the instructions:
  https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html

- Configure the AWS CLI default profile, as follow. The access key ID and secret
  should be from a user with AdministratorAccess permission.
```
$ aws configure
AWS Access Key ID [None]: <replace-with-access-key-id>
AWS Secret Access Key [None]: <replace-with-access-key>
Default region name [None]: us-west-2
Default output format [None]: json
```
  The configurations are stored in:
  - `~/.aws/config` file:
```
[default]
region=us-west-2
```  
  
  - `~/.aws/credential` file
```
[default]
aws_access_key_id = <replace-with-access-key-id>
aws_secret_access_key = <replace-with-access-key>
```


## SETUP for this workshop

- Setup the local linux "Environment variables" in the `~/.bashrc` file as follow.
  Please adjust the preferred default region. 
```
$ echo "export AWS_DEFAULT_REGION='us-west-2' >> ~/.bashrc
$ echo "export AWS_REGION=\$AWS_DEFAULT_REGION" >> ~/.bashrc
$ echo "export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)" >> ~/.bashrc
$ source ~/.bashrc
```

- Validate the aws region has been setup
``` 
$ test -n "$AWS_REGION" && echo AWS_REGION is "$AWS_REGION" || echo AWS_REGION is not set
```

- Setup the local linux `~/.bash_profile` file as follow.
```
$ echo "export AWS_ACCOUNT_ID=${AWS_ACCOUNT_ID}" | tee -a ~/.bash_profile
$ echo "export AWS_REGION=${AWS_REGION}" | tee -a ~/.bash_profile
$ aws configure get default.region
```

- SKIP NOTE: unlike for the Cloud9 environment, we do not need to create the role,
  because our setup is using the local linux `aws cli` that has been configured
  to use the user who has the AdministratorRole, as described above.
  Source: [Create the IAM Role and attach it to the Cloud9 instance](https://ecsworkshop.com/start_the_workshop/workspace/#create-the-iam-role-and-attach-it-to-the-cloud9-instance)

- SKIP NOTE: unlike for he Cloud9 environment, we do not need to setup the
  the `~/.aws/config` - `credential_source = Ec2InstanceMetadata` as instructed
  in the workshop
  Source: [Using credentials for Amazon EC2 instance metadata](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-metadata.html)

- Clone the repo; 
  Source: [CLONE THE SERVICE REPOS](https://ecsworkshop.com/microservices/clone/)
  NOTE: the service repos have been cloned and included with this repo in the
  corresponding folders. If a newer versions are available, delete the folders
  and reclone them.   
```
$ cd <project_folder>/aws-ecs-workshop
$ git clone https://github.com/aws-containers/ecsdemo-platform
$ git clone https://github.com/aws-containers/ecsdemo-frontend
$ git clone https://github.com/aws-containers/ecsdemo-nodejs
$ git clone https://github.com/aws-containers/ecsdemo-crystal
```

- Platform - Build the Platform; we are using copilot, so the platform resources
  will be created automatically by copilot. This will be done during deployment
  of the `ecsdemo-frontend` below when we issue `copilot init`  

- Platform - BUILD THE ENVIRONMENTS. Ensure service linked roles exist for 
  Load Balancers as follow. Alternatively, check using the aws console by 
  go to "IAM > Roles" page, filter the role "AWSServiceRoleForElasticLoadBalancing"
```
$ aws iam get-role --role-name "AWSServiceRoleForElasticLoadBalancing" 
```
  - If the role does not exist, create it:
  ```
  $ aws iam create-service-linked-role --aws-service-name "elasticloadbalancing.amazonaws.com"
  ```
  - Note, if the role already exists and you run the command to create it again
    an error will occur
  ```
  $ aws iam create-service-linked-role --aws-service-name "elasticloadbalancing.amazonaws.com"

  An error occurred (InvalidInput) when calling the CreateServiceLinkedRole operation: Service role name AWSServiceRoleForElasticLoadBalancing has been taken in this account, please try a different suffix.
  ```

- Platform - BUILD THE ENVIRONMENTS. Ensure service linked roles exist for ECS:
```
$ aws iam get-role --role-name "AWSServiceRoleForECS"
```
  - otherwise, create it
  ```
  $ aws iam create-service-linked-role --aws-service-name "ecs.amazonaws.com"
  ```  

- Platform - BUILD THE ENVIRONMENTS. Install the session manager plugin;
```
# For RHEL OS
curl "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/linux_64bit/session-manager-plugin.rpm" -o "session-manager-plugin.rpm"
sudo yum install -y session-manager-plugin.rpm
session-manager-plugin

# For Ubuntu
# https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html#install-plugin-linux
$ curl "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/ubuntu_64bit/session-manager-plugin.deb" -o "session-manager-plugin.deb"
$ sudo dpkg -i session-manager-plugin.deb
```
  - Verify the session-manager-plugin was installed.
```
$ session-manager-plugin
The Session Manager plugin is installed successfully. Use the AWS CLI to start a session.
$
```

## DEPLOY

### DEPLOY FRONTEND RAILS APP

Since we are deploying the first container, copilot will create the 
"copilot application", and the corresponding platform resources.

**Note:** we are not using any artifacts from the "ecsdemo-platform" repo.
The repo contains "aws cdk" codes for building platform resources that is 
generated by the copilot automatically during `copilot init` performed below.

- **Initialize** copilot application
```
$ cd <project_folder>/aws-ecs-workshop/ecsdemo-frontend
$ copilot init
Welcome to the Copilot CLI! We're going to walk you through some questions
to help you get set up with a containerized application on AWS. An application is a collection of
containerized services that operate together.

Application name: ecsworkshop
Workload type: Load Balanced Web Service
Service name: ecsdemo-frontend
Dockerfile: ./Dockerfile
Ok great, we'll set up a Load Balanced Web Service named ecsdemo-frontend in application ecsworkshop listening on port 3000.

✔ Proposing infrastructure changes for stack ecsworkshop-infrastructure-roles
- Creating the infrastructure for stack ecsworkshop-infrastructure-roles                        [create complete]  [52.9s]
  - A StackSet admin role assumed by CloudFormation to manage regional stacks                   [create complete]  [22.0s]
  - An IAM role assumed by the admin role to create ECR repositories, KMS keys, and S3 buckets  [create complete]  [24.7s]
✔ The directory copilot will hold service manifests for application ecsworkshop.

✔ Wrote the manifest for service ecsdemo-frontend at copilot/ecsdemo-frontend/manifest.yml
Your manifest contains configurations like your container size and port (:3000).

- Update regional resources with stack set "ecsworkshop-infrastructure"  [succeeded]  [0.0s]
All right, you're all set for local development.
Deploy: Yes

✔ Wrote the manifest for environment test at copilot/environments/test/manifest.yml
- Update regional resources with stack set "ecsworkshop-infrastructure"  [succeeded]  [0.0s]
- Update regional resources with stack set "ecsworkshop-infrastructure"  [succeeded]          [132.0s]
  - Update resources in region "us-west-2"                               [create complete]    [124.6s]
    - ECR container image repository for "ecsdemo-frontend"              [create complete]    [2.4s]
    - KMS key to encrypt pipeline artifacts between stages               [create complete]    [122.7s]
    - S3 Bucket to store local artifacts                                 [create in progress]  [106.3s]
✔ Proposing infrastructure changes for the ecsworkshop-test environment.
- Creating the infrastructure for the ecsworkshop-test environment.  [create complete]  [56.3s]
  - An IAM Role for AWS CloudFormation to manage resources           [create complete]  [20.2s]
  - An IAM Role to describe resources in your environment            [create complete]  [19.7s]
✔ Provisioned bootstrap resources for environment test in region us-west-2 under application ecsworkshop.
✔ Provisioned bootstrap resources for environment test.
✔ Proposing infrastructure changes for the ecsworkshop-test environment.
- Creating the infrastructure for the ecsworkshop-test environment.           [update complete]  [80.4s]
  - An ECS cluster to group your services                                     [create complete]  [6.7s]
  - A security group to allow your containers to talk to each other           [create complete]  [0.0s]
  - An Internet Gateway to connect to the public internet                     [create complete]  [18.1s]
  - Private subnet 1 for resources with no internet access                    [create complete]  [5.6s]
  - Private subnet 2 for resources with no internet access                    [create complete]  [5.6s]
  - A custom route table that directs network traffic for the public subnets  [create complete]  [12.0s]
  - Public subnet 1 for resources that can access the internet                [create complete]  [3.9s]
  - Public subnet 2 for resources that can access the internet                [create complete]  [5.6s]
  - A private DNS namespace for discovering services within the environment   [create complete]  [44.7s]
  - A Virtual Private Cloud to control networking of your AWS resources       [create complete]  [11.4s]
Sending build context to Docker daemon  1.933MB
Step 1/9 : FROM public.ecr.aws/bitnami/ruby:2.5
2.5: Pulling from bitnami/ruby
3cc1ae8ac7e0: Pull complete 
a8ad656a84c0: Pull complete 
48991004b324: Pull complete 
c9967730b138: Pull complete 
6e9279bed9c1: Pull complete 
b14c5e200b02: Pull complete 
Digest: sha256:fb00ab79006927ea50bfc1a8ed08803ec7bf8dd54e3bbeeaed7a1871c275c288
Status: Downloaded newer image for public.ecr.aws/bitnami/ruby:2.5
 ---> 5172fb39c740
Step 2/9 : COPY Gemfile Gemfile.lock /usr/src/app/
 ---> 2dfae76b7167
Step 3/9 : WORKDIR /usr/src/app
 ---> Running in 1723df3fcfba
Removing intermediate container 1723df3fcfba
 ---> cd5a62a59514
Step 4/9 : RUN apt-get update && apt-get -y install iproute2 curl jq libgmp3-dev ruby-dev build-essential sqlite libsqlite3-dev python3 python3-pip &&     gem install bundler:1.17.3 &&     bundle install &&     pip3 install awscli netaddr &&     apt-get autoremove -y --purge &&     apt-get remove -y --auto-remove --purge ruby-dev libgmp3-dev build-essential libsqlite3-dev &&     apt-get clean &&     rm -rvf /root/* /root/.gem* /var/cache/*
 ---> Running in 5ab06269ccca
Get:1 http://deb.debian.org/debian buster InRelease [122 kB]
Get:2 http://security.debian.org buster/updates InRelease [34.8 kB]
Get:3 http://deb.debian.org/debian buster/main amd64 Packages [10.7 MB]
Get:4 http://security.debian.org buster/updates/main amd64 Packages [443 kB]
Fetched 11.3 MB in 2s (5683 kB/s)
Reading package lists...
Reading package lists...
Building dependency tree...
Reading state information...
build-essential is already the newest version (12.6).
libsqlite3-dev is already the newest version (3.27.2-3+deb10u1).
The following additional packages will be installed:
  dbus dh-python file fonts-lato gir1.2-glib-2.0 javascript-common
  libapparmor1 libatm1 libcap2 libcap2-bin libcurl4 libelf1 libexpat1

[...]

Fetching uglifier 4.1.20
Installing uglifier 4.1.20
Fetching web-console 2.3.0
Installing web-console 2.3.0
Bundle complete! 15 Gemfile dependencies, 66 gems now installed.
Use `bundle info [gemname]` to see where a bundled gem is installed.
Post-install message from sass:

Ruby Sass has reached end-of-life and should no longer be used.

* If you use Sass as a command-line tool, we recommend using Dart Sass, the new
  primary implementation: https://sass-lang.com/install

* If you use Sass as a plug-in for a Ruby web framework, we recommend using the
  sassc gem: https://github.com/sass/sassc-ruby#readme

* For more details, please refer to the Sass blog:
  https://sass-lang.com/blog/posts/7828841

Collecting awscli
  Downloading https://files.pythonhosted.org/packages/06/63/b604df48563c4d2008fe0972c1873d57a9e8fcc7c5bf98531cacae18eb0f/awscli-1.25.71-py3-none-any.whl (3.9MB)
Collecting netaddr
  Downloading https://files.pythonhosted.org/packages/ff/cd/9cdfea8fc45c56680b798db6a55fa60a22e2d3d3ccf54fc729d083b50ce4/netaddr-0.8.0-py2.py3-none-any.whl (1.9MB)
Collecting docutils<0.17,>=0.10 (from awscli)
  Downloading https://files.pythonhosted.org/packages/81/44/8a15e45ffa96e6cf82956dd8d7af9e666357e16b0d93b253903475ee947f/docutils-0.16-py2.py3-none-any.whl (548kB)
Collecting s3transfer<0.7.0,>=0.6.0 (from awscli)
  Downloading https://files.pythonhosted.org/packages/5e/c6/af903b5fab3f9b5b1e883f49a770066314c6dcceb589cf938d48c89556c1/s3transfer-0.6.0-py3-none-any.whl (79kB)
Collecting rsa<4.8,>=3.1.2 (from awscli)
  Downloading https://files.pythonhosted.org/packages/e9/93/0c0f002031f18b53af7a6166103c02b9c0667be528944137cc954ec921b3/rsa-4.7.2-py3-none-any.whl
Collecting colorama<0.4.5,>=0.2.5 (from awscli)
  Downloading https://files.pythonhosted.org/packages/44/98/5b86278fbbf250d239ae0ecb724f8572af1c91f4a11edf4d36a206189440/colorama-0.4.4-py2.py3-none-any.whl
Collecting PyYAML<5.5,>=3.10 (from awscli)
  Downloading https://files.pythonhosted.org/packages/7a/a5/393c087efdc78091afa2af9f1378762f9821c9c1d7a22c5753fb5ac5f97a/PyYAML-5.4.1-cp37-cp37m-manylinux1_x86_64.whl (636kB)
Collecting botocore==1.27.70 (from awscli)
  Downloading https://files.pythonhosted.org/packages/58/11/4de2aa56633264a0c040b1b7158e5c39390029e6f6054f51b32f9375e7ba/botocore-1.27.70-py3-none-any.whl (9.1MB)
Collecting pyasn1>=0.1.3 (from rsa<4.8,>=3.1.2->awscli)
  Downloading https://files.pythonhosted.org/packages/62/1e/a94a8d635fa3ce4cfc7f506003548d0a2447ae76fd5ca53932970fe3053f/pyasn1-0.4.8-py2.py3-none-any.whl (77kB)
Collecting jmespath<2.0.0,>=0.7.1 (from botocore==1.27.70->awscli)
  Downloading https://files.pythonhosted.org/packages/31/b4/b9b800c45527aadd64d5b442f9b932b00648617eb5d63d2c7a6587b7cafc/jmespath-1.0.1-py3-none-any.whl
Collecting python-dateutil<3.0.0,>=2.1 (from botocore==1.27.70->awscli)
  Downloading https://files.pythonhosted.org/packages/36/7a/87837f39d0296e723bb9b62bbb257d0355c7f6128853c78955f57342a56d/python_dateutil-2.8.2-py2.py3-none-any.whl (247kB)
Collecting urllib3<1.27,>=1.25.4 (from botocore==1.27.70->awscli)
  Downloading https://files.pythonhosted.org/packages/6f/de/5be2e3eed8426f871b170663333a0f627fc2924cc386cd41be065e7ea870/urllib3-1.26.12-py2.py3-none-any.whl (140kB)
Requirement already satisfied: six>=1.5 in /usr/lib/python3/dist-packages (from python-dateutil<3.0.0,>=2.1->botocore==1.27.70->awscli) (1.12.0)
Installing collected packages: docutils, jmespath, python-dateutil, urllib3, botocore, s3transfer, pyasn1, rsa, colorama, PyYAML, awscli, netaddr
Successfully installed PyYAML-5.4.1 awscli-1.25.71 botocore-1.27.70 colorama-0.4.4 docutils-0.16 jmespath-1.0.1 netaddr-0.8.0 pyasn1-0.4.8 python-dateutil-2.8.2 rsa-4.7.2 s3transfer-0.6.0 urllib3-1.26.12
Reading package lists...
Building dependency tree...
Reading state information...
0 upgraded, 0 newly installed, 0 to remove and 57 not upgraded.
Reading package lists...
Building dependency tree...
Reading state information...
The following packages will be REMOVED:
  binutils* binutils-common* binutils-x86-64-linux-gnu* build-essential* cpp*
  cpp-8* dpkg-dev* fonts-lato* g++* g++-8* gcc* gcc-8* javascript-common*
  libasan5* libatomic1* libbinutils* libcc1-0* libgcc-8-dev* libgmp3-dev*
  libisl19* libitm1* libjs-jquery* liblsan0* libmpc3* libmpfr6* libmpx2*
  libquadmath0* libruby2.5* libsqlite3-dev* libstdc++-8-dev* libtsan0*
  libubsan1* libyaml-0-2* make* rake* ruby* ruby-dev* ruby-did-you-mean*
  ruby-minitest* ruby-net-telnet* ruby-power-assert* ruby-test-unit*
  ruby-xmlrpc* ruby2.5* ruby2.5-dev* ruby2.5-doc* rubygems-integration* zip*
0 upgraded, 0 newly installed, 48 to remove and 56 not upgraded.
After this operation, 205 MB disk space will be freed.
(Reading database ... 34873 files and directories currently installed.)
Removing build-essential (12.6) ...
Removing dpkg-dev (1.19.7) ...

[...]

removed '/var/cache/ldconfig/aux-cache'
removed directory '/var/cache/ldconfig'
Removing intermediate container 5ab06269ccca
 ---> 64cdf71408f5
Step 5/9 : COPY . /usr/src/app
 ---> 0c6b87837b94
Step 6/9 : RUN chmod +x /usr/src/app/startup-cdk.sh
 ---> Running in 3d95bf4719cd
Removing intermediate container 3d95bf4719cd
 ---> 5b3118471d04
Step 7/9 : HEALTHCHECK --interval=10s --timeout=3s   CMD curl -f -s http://localhost:3000/health/ || exit 1
 ---> Running in 6485a943b473
Removing intermediate container 6485a943b473
 ---> b232415bd650
Step 8/9 : EXPOSE 3000
 ---> Running in 248af3af00b6
Removing intermediate container 248af3af00b6
 ---> cbcf4b642014
Step 9/9 : ENTRYPOINT ["bash","/usr/src/app/startup-cdk.sh"]
 ---> Running in f87ac06105be
Removing intermediate container f87ac06105be
 ---> c2f2d7e385c6
Successfully built c2f2d7e385c6
Successfully tagged 349327579537.dkr.ecr.us-west-2.amazonaws.com/ecsworkshop/ecsdemo-frontend:latest
WARNING! Your password will be stored unencrypted in /home/gabe/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
Using default tag: latest
The push refers to repository [349327579537.dkr.ecr.us-west-2.amazonaws.com/ecsworkshop/ecsdemo-frontend]
06eb06e89776: Pushed 
d956d0d2c843: Pushed 
1b16dcc74d15: Pushed 
ec036a1d110f: Pushed 
097184cb5649: Pushed 
ca58e20a1e0a: Pushed 
b8f1b3c1c4cb: Pushed 
958773801523: Pushed 
caa232770d56: Pushed 
9da9c7071a76: Pushed 
latest: digest: sha256:d7aee3ac50759cda77b3f2385d65d7a2b8c663e8b91342c9f14e66299a95f255 size: 2413
✔ Proposing infrastructure changes for stack ecsworkshop-test-ecsdemo-frontend
- Creating the infrastructure for stack ecsworkshop-test-ecsdemo-frontend         [create complete]  [354.1s]
  - An Addons CloudFormation Stack for your additional AWS resources              [create complete]  [227.7s]
  - Service discovery for your services to communicate within the VPC             [create complete]  [0.0s]
  - Update your environment's shared resources                                    [update complete]  [123.1s]
    - A security group for your load balancer allowing HTTP traffic               [create complete]  [8.3s]
    - An Application Load Balancer to distribute public traffic to your services  [create complete]  [91.4s]
    - A load balancer listener to route HTTP traffic                              [create complete]  [2.0s]
  - An IAM role to update your environment stack                                  [create complete]  [21.4s]
  - An IAM Role for the Fargate agent to make AWS API calls on your behalf        [create complete]  [18.7s]
  - A HTTP listener rule for forwarding HTTP traffic                              [create complete]  [4.3s]
  - A custom resource assigning priority for HTTP listener rules                  [create complete]  [0.0s]
  - A CloudWatch log group to hold your service logs                              [create complete]  [0.0s]
  - An IAM Role to describe load balancer rules for assigning a priority          [create complete]  [18.7s]
  - An ECS service to run and maintain your tasks in the environment cluster      [create complete]  [90.3s]
    Deployments                                                                                       
               Revision  Rollout      Desired  Running  Failed  Pending                                       
      PRIMARY  1         [completed]  1        1        0       0                                             
  - A target group to connect the load balancer to your service                   [create complete]  [0.0s]
  - An ECS task definition to group your containers and run them on ECS           [create complete]  [3.2s]
  - An IAM role to control permissions for the containers in your tasks           [create complete]  [23.4s]
✔ Deployed service ecsdemo-frontend.
Recommended follow-up action:
  - You can access your service at http://ecswo-Publi-I78Y270HWMBY-626812307.us-west-2.elb.amazonaws.com over the internet.
- Be a part of the Copilot ✨community✨!
  Ask or answer a question, submit a feature request...
  Visit 👉 https://aws.github.io/copilot-cli/community/get-involved/ to see how!
$ 
```

- Next, the application presents the git hash to show the version of the 
  application that is deployed.
```
$ git rev-parse --short=7 HEAD > code_hash.txt
```

- We’re now ready to deploy our environment; the repo contains precreated
  folder & manifest. 
```
$ copilot env init --name test --profile default --default-config
It seems like you are trying to init an environment that already exists.
To generate a manifest for the environment:
1. `mkdir -p copilot/environments/test`
2. `copilot env show -n test --manifest > copilot/environments/test/manifest.yml`

Alternatively, to recreate the environment:
1. `copilot env delete --name test`
2. And then `copilot env init --name test`
✘ environment test already exists
```

- **DEPLOY** our service
```
$ copilot svc deploy
Only found one service, defaulting to: ecsdemo-frontend
Only found one environment, defaulting to: test
Sending build context to Docker daemon  1.933MB
Step 1/9 : FROM public.ecr.aws/bitnami/ruby:2.5
 ---> 5172fb39c740
Step 2/9 : COPY Gemfile Gemfile.lock /usr/src/app/
 ---> Using cache
 ---> 2dfae76b7167
Step 3/9 : WORKDIR /usr/src/app
 ---> Using cache
 ---> cd5a62a59514
Step 4/9 : RUN apt-get update && apt-get -y install iproute2 curl jq libgmp3-dev ruby-dev build-essential sqlite libsqlite3-dev python3 python3-pip &&     gem install bundler:1.17.3 &&     bundle install &&     pip3 install awscli netaddr &&     apt-get autoremove -y --purge &&     apt-get remove -y --auto-remove --purge ruby-dev libgmp3-dev build-essential libsqlite3-dev &&     apt-get clean &&     rm -rvf /root/* /root/.gem* /var/cache/*
 ---> Using cache
 ---> 64cdf71408f5
Step 5/9 : COPY . /usr/src/app
 ---> 4c10245c8896
Step 6/9 : RUN chmod +x /usr/src/app/startup-cdk.sh
 ---> Running in f9c4fc029696
Removing intermediate container f9c4fc029696
 ---> d685821793ae
Step 7/9 : HEALTHCHECK --interval=10s --timeout=3s   CMD curl -f -s http://localhost:3000/health/ || exit 1
 ---> Running in b6817879ec46
Removing intermediate container b6817879ec46
 ---> 39771defca2e
Step 8/9 : EXPOSE 3000
 ---> Running in 964d4a3a62fc
Removing intermediate container 964d4a3a62fc
 ---> a19945418004
Step 9/9 : ENTRYPOINT ["bash","/usr/src/app/startup-cdk.sh"]
 ---> Running in efa20bc870fe
Removing intermediate container efa20bc870fe
 ---> 844332fb6c36
Successfully built 844332fb6c36
Successfully tagged 349327579537.dkr.ecr.us-west-2.amazonaws.com/ecsworkshop/ecsdemo-frontend:latest
WARNING! Your password will be stored unencrypted in /home/gabe/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
Using default tag: latest
The push refers to repository [349327579537.dkr.ecr.us-west-2.amazonaws.com/ecsworkshop/ecsdemo-frontend]
2a724fa59917: Pushed 
f1bee8d60c82: Pushed 
1b16dcc74d15: Layer already exists 
ec036a1d110f: Layer already exists 
097184cb5649: Layer already exists 
ca58e20a1e0a: Layer already exists 
b8f1b3c1c4cb: Layer already exists 
958773801523: Layer already exists 
caa232770d56: Layer already exists 
9da9c7071a76: Layer already exists 
latest: digest: sha256:0f9570bdce6c372fc48b149df6f305406e42fe2dcd7787b55844bbfff3d54df3 size: 2413
✔ Proposing infrastructure changes for stack ecsworkshop-test-ecsdemo-frontend
- Updating the infrastructure for stack ecsworkshop-test-ecsdemo-frontend     [update complete]  [285.2s]
  - An ECS service to run and maintain your tasks in the environment cluster  [update complete]  [259.6s]
    Deployments                                                                                   
               Revision  Rollout      Desired  Running  Failed  Pending                                   
      PRIMARY  2         [completed]  1        1        0       0                                         
  - An ECS task definition to group your containers and run them on ECS       [delete complete]  [2.0s]
✔ Deployed service ecsdemo-frontend.
Recommended follow-up action:
  - You can access your service at http://ecswo-Publi-I78Y270HWMBY-626812307.us-west-2.elb.amazonaws.com over the internet.
```

- Grab the load balancer url from below, and paste it into your browser.
```
$ copilot svc show -n ecsdemo-frontend --json
{"service":"ecsdemo-frontend","type":"Load Balanced Web Service","application":"ecsworkshop","configurations":[{"environment":"test","port":"3000","cpu":"256","memory":"512","platform":"LINUX/X86_64","tasks":"1"}],"routes":[{"environment":"test","url":"http://ecswo-Publi-I78Y270HWMBY-626812307.us-west-2.elb.amazonaws.com"}],"serviceDiscovery":[{"environment":["test"],"namespace":"ecsdemo-frontend.test.ecsworkshop.local:3000"}],"variables":[{"environment":"test","name":"COPILOT_SERVICE_DISCOVERY_ENDPOINT","value":"test.ecsworkshop.local","container":"ecsdemo-frontend"},{"environment":"test","name":"COPILOT_LB_DNS","value":"ecswo-Publi-I78Y270HWMBY-626812307.us-west-2.elb.amazonaws.com","container":"ecsdemo-frontend"},{"environment":"test","name":"COPILOT_APPLICATION_NAME","value":"ecsworkshop","container":"ecsdemo-frontend"},{"environment":"test","name":"COPILOT_ENVIRONMENT_NAME","value":"test","container":"ecsdemo-frontend"},{"environment":"test","name":"CRYSTAL_URL","value":"http://ecsdemo-crystal.test.ecsworkshop.local:3000/crystal","container":"ecsdemo-frontend"},{"environment":"test","name":"NODEJS_URL","value":"http://ecsdemo-nodejs.test.ecsworkshop.local:3000","container":"ecsdemo-frontend"},{"environment":"test","name":"COPILOT_SERVICE_NAME","value":"ecsdemo-frontend","container":"ecsdemo-frontend"}]}

$ copilot svc show -n ecsdemo-frontend
About

  Application  ecsworkshop
  Name         ecsdemo-frontend
  Type         Load Balanced Web Service

Configurations

  Environment  Tasks     CPU (vCPU)  Memory (MiB)  Platform      Port
  -----------  -----     ----------  ------------  --------      ----
  test         1         0.25        512           LINUX/X86_64  3000

Routes

  Environment  URL
  -----------  ---
  test         http://ecswo-Publi-I78Y270HWMBY-626812307.us-west-2.elb.amazonaws.com

Service Discovery

  Environment  Namespace
  -----------  ---------
  test         ecsdemo-frontend.test.ecsworkshop.local:3000

Variables

  Name                                Container         Environment  Value
  ----                                ---------         -----------  -----
  COPILOT_APPLICATION_NAME            ecsdemo-frontend  test         ecsworkshop
  COPILOT_ENVIRONMENT_NAME              "                 "          test
  COPILOT_LB_DNS                        "                 "          ecswo-Publi-I78Y270HWMBY-626812307.us-west-2.elb.amazonaws.com
  COPILOT_SERVICE_DISCOVERY_ENDPOINT    "                 "          test.ecsworkshop.local
  COPILOT_SERVICE_NAME                  "                 "          ecsdemo-frontend
  CRYSTAL_URL                           "                 "          http://ecsdemo-crystal.test.ecsworkshop.local:3000/crystal
  NODEJS_URL                            "                 "          http://ecsdemo-nodejs.test.ecsworkshop.local:3000
```

- **INTERACTING** with the copilot *app*
```
$ copilot app
Commands for applications.
Applications are a collection of services and environments.

Usage
  copilot app [command]

Available Commands
  init        Creates a new empty application.
  ls          Lists all the applications in your account.
  show        Shows info about an application.
  delete      Delete all resources associated with the application.
  upgrade     Upgrades the template of an application to the latest version.

Flags
  -h, --help   help for app

#####################################################################
  
$ copilot app ls
ecsworkshop

#####################################################################

$ copilot app show ecsworkshop
About

  Name     ecsworkshop
  Version  v1.0.2 
  URI      

Environments

  Name    AccountID     Region
  ----    ---------     ------
  test    349327579537  us-west-2

Workloads

  Name              Type                       Environments
  ----              ----                       ------------
  ecsdemo-frontend  Load Balanced Web Service  test

Pipelines

  Name
  ----

# The output shows the environments and services deployed under the application.
# We would want to deploy a production environment that is completely isolated 
# from test. Ideally that would be in another account as well.
#####################################################################
```

- **INTERACTING** with the copilot app *environment*
```
copilot env
Commands for environments.
Environments are deployment stages shared between services.

Usage
  copilot env [command]

Available Commands
  init        Creates a new environment in your application.
  ls          Lists all the environments in an application.
  delete      Deletes an environment from your application.
  show        Shows info about a deployed environment.
  deploy      Deploys an environment to an application.
  package     Print the AWS CloudFormation template of an environment.

Flags
  -h, --help   help for env

#####################################################################

$ copilot env ls
test

#####################################################################

$ copilot env show -n test
About

  Name        test
  Region      us-west-2
  Account ID  349327579537

Workloads

  Name              Type
  ----              ----
  ecsdemo-frontend  Load Balanced Web Service

Tags

  Key                  Value
  ---                  -----
  copilot-application  ecsworkshop
  copilot-environment  test

#####################################################################
```

- **INTERACTING** with the copilot app *service*
```
$ copilot svc
Commands for services.
Services are long-running ECS or App Runner services.

Usage
  copilot svc [command]

Available Commands
  init        Creates a new service in an application.
  ls          Lists all the services in an application.
  package     Print the AWS CloudFormation template of a service.
  deploy      Deploys a service to an environment.
  delete      Deletes a service from an application.
  show        Shows info about a deployed service per environment.
  status      Shows status of a deployed service.
  logs        Displays logs of a deployed service.
  exec        Execute a command in a running container part of a service.
  pause       Pause running App Runner service.
  resume      Resumes a paused service.

Flags
  -h, --help   help for svc

#####################################################################

$ copilot svc status -n ecsdemo-frontend
SERVICE ecsdemo-frontend found in environment test
Task Summary

  Running   ██████████  1/1 desired tasks are running
  Health    ██████████  1/1 passes HTTP health checks

Tasks

  ID        Status      Revision    Started At      HTTP Health
  --        ------      --------    ----------      -----------
  628946e4  RUNNING     2           17 minutes ago  HEALTHY

#####################################################################  
```

- **SCALE OUT** the number of service containers; open the manifest file 
  (`./copilot/ecsdemo-frontend/manifest.yml`), and replace the value of the 
  count key from 1 to 3.
```
# Number of tasks that should be running in your service.
count: 3
```

- **SCALE OUT** - **Redeploy the service**; copilot does the following:
  - Build your image locally
  - Push to your service’s ECR repository
  - Convert your manifest file to CloudFormation
  - Package any additional infrastructure into CloudFormation
  - Deploy your updated service and resources to CloudFormation
```
$ copilot svc deploy
Only found one service, defaulting to: ecsdemo-frontend
Only found one environment, defaulting to: test
Sending build context to Docker daemon  1.933MB
Step 1/9 : FROM public.ecr.aws/bitnami/ruby:2.5
 ---> 5172fb39c740
Step 2/9 : COPY Gemfile Gemfile.lock /usr/src/app/
 ---> Using cache
 ---> 2dfae76b7167
Step 3/9 : WORKDIR /usr/src/app
 ---> Using cache
 ---> cd5a62a59514
Step 4/9 : RUN apt-get update && apt-get -y install iproute2 curl jq libgmp3-dev ruby-dev build-essential sqlite libsqlite3-dev python3 python3-pip &&     gem install bundler:1.17.3 &&     bundle install &&     pip3 install awscli netaddr &&     apt-get autoremove -y --purge &&     apt-get remove -y --auto-remove --purge ruby-dev libgmp3-dev build-essential libsqlite3-dev &&     apt-get clean &&     rm -rvf /root/* /root/.gem* /var/cache/*
 ---> Using cache
 ---> 64cdf71408f5
Step 5/9 : COPY . /usr/src/app
 ---> 22e322cbac06
Step 6/9 : RUN chmod +x /usr/src/app/startup-cdk.sh
 ---> Running in 53db0f9a5898
Removing intermediate container 53db0f9a5898
 ---> faa95f38510c
Step 7/9 : HEALTHCHECK --interval=10s --timeout=3s   CMD curl -f -s http://localhost:3000/health/ || exit 1
 ---> Running in 65f618905623
Removing intermediate container 65f618905623
 ---> a39f4b67f4f9
Step 8/9 : EXPOSE 3000
 ---> Running in dbc120d2abb4
Removing intermediate container dbc120d2abb4
 ---> 2fafb22db25b
Step 9/9 : ENTRYPOINT ["bash","/usr/src/app/startup-cdk.sh"]
 ---> Running in 247429f47468
Removing intermediate container 247429f47468
 ---> 36b1408cfba7
Successfully built 36b1408cfba7
Successfully tagged 349327579537.dkr.ecr.us-west-2.amazonaws.com/ecsworkshop/ecsdemo-frontend:latest
WARNING! Your password will be stored unencrypted in /home/gabe/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
Using default tag: latest
The push refers to repository [349327579537.dkr.ecr.us-west-2.amazonaws.com/ecsworkshop/ecsdemo-frontend]
40afa70be36b: Pushed 
101261a0c6f8: Pushed 
1b16dcc74d15: Layer already exists 
ec036a1d110f: Layer already exists 
097184cb5649: Layer already exists 
ca58e20a1e0a: Layer already exists 
b8f1b3c1c4cb: Layer already exists 
958773801523: Layer already exists 
caa232770d56: Layer already exists 
9da9c7071a76: Layer already exists 
latest: digest: sha256:c8bf57a3d988b55b40443bb749b98ae3a3b3d600a34cd4021a0792d9f9f53ffd size: 2413
✔ Proposing infrastructure changes for stack ecsworkshop-test-ecsdemo-frontend
- Updating the infrastructure for stack ecsworkshop-test-ecsdemo-frontend     [update complete]  [293.5s]
  - An ECS service to run and maintain your tasks in the environment cluster  [update complete]  [268.7s]
    Deployments                                                                                   
               Revision  Rollout      Desired  Running  Failed  Pending                                   
      PRIMARY  3         [completed]  3        3        0       0                                         
  - An ECS task definition to group your containers and run them on ECS       [delete complete]  [4.0s]
✔ Deployed service ecsdemo-frontend.
Recommended follow-up action:
  - You can access your service at http://ecswo-Publi-I78Y270HWMBY-626812307.us-west-2.elb.amazonaws.com over the internet.
```

- **SCALE OUT** - check the number of service container
```
$ copilot svc status -n ecsdemo-frontend
SERVICE ecsdemo-frontend found in environment test
Task Summary

  Running   ██████████  3/3 desired tasks are running
  Health    ██████████  3/3 passes HTTP health checks

Tasks

  ID        Status      Revision    Started At     HTTP Health
  --        ------      --------    ----------     -----------
  06625937  RUNNING     3           6 minutes ago  HEALTHY
  558639b8  RUNNING     3           6 minutes ago  HEALTHY
  5e7d8ee3  RUNNING     3           6 minutes ago  HEALTHY
$
```

- **LOG** - review the service log; pass the -e flag with the environment name
```
$ copilot svc logs -a ecsworkshop -n ecsdemo-frontend --follow
SERVICE ecsdemo-frontend found in environment test
copilot/ecsdemo-frontend/ E, [2022-09-09T21:54:34.127470 #1] ERROR -- : DNS result has no information for ecsdemo-crystal.test.ecsworkshop.local
copilot/ecsdemo-frontend/ I, [2022-09-09T21:54:34.127515 #1]  INFO -- : expanded http://ecsdemo-crystal.test.ecsworkshop.local:3000/crystal to http://ecsdemo-crystal.test.ecsworkshop.local:3000/crystal
copilot/ecsdemo-frontend/ E, [2022-09-09T21:54:34.129879 #1] ERROR -- : DNS result has no information for ecsdemo-crystal.test.ecsworkshop.local
copilot/ecsdemo-frontend/ I, [2022-09-09T21:54:34.129922 #1]  INFO -- : expanded http://ecsdemo-crystal.test.ecsworkshop.local:3000/crystal to http://ecsdemo-crystal.test.ecsworkshop.local:3000/crystal
copilot/ecsdemo-frontend/ E, [2022-09-09T21:54:34.131196 #1] ERROR -- : Failed to open TCP connection to ecsdemo-crystal.test.ecsworkshop.local:3000 (getaddrinfo: Name or service not known)
copilot/ecsdemo-frontend/ I, [2022-09-09T21:54:34.131976 #1]  INFO -- :   Rendered application/index.html.erb within layouts/application (0.3ms)
copilot/ecsdemo-frontend/ I, [2022-09-09T21:54:34.132256 #1]  INFO -- : Completed 200 OK in 26ms (Views: 0.8ms | ActiveRecord: 0.0ms)
copilot/ecsdemo-frontend/ I, [2022-09-09T21:55:00.366804 #1]  INFO -- : Started GET "/" for 10.0.1.34 at 2022-09-09 21:55:00 +0000
copilot/ecsdemo-frontend/ I, [2022-09-09T21:55:00.367741 #1]  INFO -- : Processing by ApplicationController#index as HTML

[...]
copilot/ecsdemo-frontend/ I, [2022-09-09T21:55:30.459518 #1]  INFO -- : expanded http://ecsdemo-nodejs.test.ecsworkshop.local:3000 to http://ecsdemo-nodejs.test.ecsworkshop.local:3000
copilot/ecsdemo-frontend/ E, [2022-09-09T21:55:30.462187 #1] ERROR -- : DNS result has no information for ecsdemo-nodejs.test.ecsworkshop.local
```

### Deploy the Nodejs backend service

We are now deploying the Nodejs backend service to the same platform 
infrastructure, or "copilot application" as the "ecsdemo-frontend".

- **Initialize** copilot application, but using the same application created 
  above during deployment of "ecsdemo-frontend": 
```
$ cd <project_folder>/aws-ecs-workshop/ecsdemo-nodejs
$ git rev-parse --short=7 HEAD
05f1957
$ git rev-parse --short=7 HEAD > code_hash.txt
$ copilot init
Welcome to the Copilot CLI! We're going to walk you through some questions
to help you get set up with a containerized application on AWS. An application is a collection of
containerized services that operate together.

Use existing application: Yes
Application name: ecsworkshop
Workload type: Backend Service
Service name: ecsdemo-nodejs
Dockerfile: ./Dockerfile
Ok great, we'll set up a Backend Service named ecsdemo-nodejs in application ecsworkshop listening on port 3000.

✔ Proposing infrastructure changes for stack ecsworkshop-infrastructure-roles
✔ The directory copilot will hold service manifests for application ecsworkshop.

✔ Wrote the manifest for service ecsdemo-nodejs at copilot/ecsdemo-nodejs/manifest.yml
Your manifest contains configurations like your container size and port (:3000).

- Update regional resources with stack set "ecsworkshop-infrastructure"  [succeeded]        [22.4s]
  - Update resources in region "us-west-2"                               [update complete]  [11.7s]
    - ECR container image repository for "ecsdemo-nodejs"                [create complete]  [2.6s]
All right, you're all set for local development.
Deploy: Yes

✔ Wrote the manifest for environment test at copilot/environments/test/manifest.yml
- Update regional resources with stack set "ecsworkshop-infrastructure"  [succeeded]  [9.4s]
✔ Proposing infrastructure changes for the ecsworkshop-test environment.
✔ Provisioned bootstrap resources for environment test in region us-west-2 under application ecsworkshop.
✔ Provisioned bootstrap resources for environment test.
✔ Proposing infrastructure changes for the ecsworkshop-test environment.
✘ Your update does not introduce immediate resource changes. 
This may be because the resources are not created until they are deemed 
necessary by a service deployment.

In this case, you can run `copilot env deploy --force` to push a modified template, even if there are no immediate changes.
Sending build context to Docker daemon  111.1kB
Step 1/10 : FROM public.ecr.aws/docker/library/ubuntu:18.04
18.04: Pulling from docker/library/ubuntu
726b8a513d66: Pull complete 
Digest: sha256:6fec50623d6d37b7f3c14c5b6fc36c73fd04aa8173d59d54dba00da0e7ac50ee
Status: Downloaded newer image for public.ecr.aws/docker/library/ubuntu:18.04
 ---> 35b3f4f76a24
Step 2/10 : ARG NODE=production
 ---> Running in 46bff7b94b40
Removing intermediate container 46bff7b94b40
 ---> 9e947039055b
Step 3/10 : ENV NODE_ENV ${NODE}
 ---> Running in 2833a8ab9f97
Removing intermediate container 2833a8ab9f97
 ---> 0f1524974744
Step 4/10 : WORKDIR /usr/src/app
 ---> Running in 18903032c302
Removing intermediate container 18903032c302
 ---> a13deaa4d570
Step 5/10 : COPY package*.json ./
 ---> 47e8529bb805
Step 6/10 : RUN apt-get update &&    apt install -y curl jq bash nodejs npm python3 python3-pip &&     pip3 install awscli netaddr &&     npm install &&    apt-get purge -y npm &&     apt clean
 ---> Running in b275ce4d0582
Get:1 http://archive.ubuntu.com/ubuntu bionic InRelease [242 kB]
Get:2 http://security.ubuntu.com/ubuntu bionic-security InRelease [88.7 kB]
Get:3 http://archive.ubuntu.com/ubuntu bionic-updates InRelease [88.7 kB]
Get:4 http://archive.ubuntu.com/ubuntu bionic-backports InRelease [74.6 kB]
Get:5 http://archive.ubuntu.com/ubuntu bionic/universe amd64 Packages [11.3 MB]
Get:6 http://security.ubuntu.com/ubuntu bionic-security/main amd64 Packages [2965 kB]
Get:7 http://archive.ubuntu.com/ubuntu bionic/restricted amd64 Packages [13.5 kB]
Get:8 http://archive.ubuntu.com/ubuntu bionic/multiverse amd64 Packages [186 kB]
Get:9 http://archive.ubuntu.com/ubuntu bionic/main amd64 Packages [1344 kB]
Get:10 http://archive.ubuntu.com/ubuntu bionic-updates/multiverse amd64 Packages [29.9 kB]
Get:11 http://archive.ubuntu.com/ubuntu bionic-updates/main amd64 Packages [3396 kB]
Get:12 http://archive.ubuntu.com/ubuntu bionic-updates/restricted amd64 Packages [1172 kB]
Get:13 http://archive.ubuntu.com/ubuntu bionic-updates/universe amd64 Packages [2318 kB]
Get:14 http://archive.ubuntu.com/ubuntu bionic-backports/main amd64 Packages [12.2 kB]
Get:15 http://archive.ubuntu.com/ubuntu bionic-backports/universe amd64 Packages [12.9 kB]
Get:16 http://security.ubuntu.com/ubuntu bionic-security/universe amd64 Packages [1540 kB]
Get:17 http://security.ubuntu.com/ubuntu bionic-security/multiverse amd64 Packages [22.8 kB]
Get:18 http://security.ubuntu.com/ubuntu bionic-security/restricted amd64 Packages [1131 kB]
Fetched 26.0 MB in 4s (6900 kB/s)
Reading package lists...

WARNING: apt does not have a stable CLI interface. Use with caution in scripts.

Reading package lists...
Building dependency tree...
Reading state information...
bash is already the newest version (4.4.18-2ubuntu1.3).
The following additional packages will be installed:
  binutils binutils-common binutils-x86-64-linux-gnu build-essential
  ca-certificates cpp cpp-7 dbus dh-python dirmngr dpkg-dev fakeroot file g++

[...]

After this operation, 10.7 MB disk space will be freed.
(Reading database ... 19925 files and directories currently installed.)
Removing npm (3.5.2-0ubuntu4) ...

WARNING: apt does not have a stable CLI interface. Use with caution in scripts.

Removing intermediate container b275ce4d0582
 ---> 0f34ef437c24
Step 7/10 : COPY . .
 ---> b7be03656a26
Step 8/10 : HEALTHCHECK --interval=10s --timeout=3s   CMD curl -f -s http://localhost:3000/health/ || exit 1
 ---> Running in 2681bfe3eb60
Removing intermediate container 2681bfe3eb60
 ---> 37500f4e2083
Step 9/10 : EXPOSE 3000
 ---> Running in 6fc74d8f28ec
Removing intermediate container 6fc74d8f28ec
 ---> ffc03dc32005
Step 10/10 : ENTRYPOINT ["bash","/usr/src/app/startup.sh"]
 ---> Running in 0de28665cd06
Removing intermediate container 0de28665cd06
 ---> 979586ecd7c8
Successfully built 979586ecd7c8
Successfully tagged 349327579537.dkr.ecr.us-west-2.amazonaws.com/ecsworkshop/ecsdemo-nodejs:latest
WARNING! Your password will be stored unencrypted in /home/gabe/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
Using default tag: latest
The push refers to repository [349327579537.dkr.ecr.us-west-2.amazonaws.com/ecsworkshop/ecsdemo-nodejs]
48b187a71533: Pushed 
0841d1a6ab5a: Pushed 
7ec566b4b715: Pushed 
a76cfd270838: Pushed 
4a641e21953d: Pushed 
latest: digest: sha256:f8ccfa2eba752983acf22f48473a55d77cc500fb50bd6ac15e846d11ecb465d6 size: 1366
✔ Proposing infrastructure changes for stack ecsworkshop-test-ecsdemo-nodejs
- Creating the infrastructure for stack ecsworkshop-test-ecsdemo-nodejs       [create complete]  [113.0s]
  - Service discovery for your services to communicate within the VPC         [create complete]  [0.0s]
  - Update your environment's shared resources                                [create complete]  [2.9s]
  - An IAM role to update your environment stack                              [create complete]  [20.2s]
  - An IAM Role for the Fargate agent to make AWS API calls on your behalf    [create complete]  [19.9s]
  - A CloudWatch log group to hold your service logs                          [create complete]  [0.0s]
  - An ECS service to run and maintain your tasks in the environment cluster  [create complete]  [63.1s]
    Deployments                                                                                   
               Revision  Rollout      Desired  Running  Failed  Pending                                   
      PRIMARY  1         [completed]  1        1        0       0                                         
  - An ECS task definition to group your containers and run them on ECS       [create complete]  [3.9s]
  - An IAM role to control permissions for the containers in your tasks       [create complete]  [20.2s]
✔ Deployed service ecsdemo-nodejs.
Recommended follow-up action:
  - You can access your service at ecsdemo-nodejs.test.ecsworkshop.local:3000 with service discovery.
- Be a part of the Copilot ✨community✨!
  Ask or answer a question, submit a feature request...
  Visit 👉 https://aws.github.io/copilot-cli/community/get-involved/ to see how!
$ 
```
  **Note:** the responses to the copilot prompted series of questions:
  - Would you like to use one of your existing applications? “Y”
  - Which existing application do you want to add a new service to? Select “ecsworkshop”, hit enter
  - Which service type best represents your service’s architecture? Select “Backend Service”, hit enter
  - What do you want to name this Backend Service: ecsdemo-nodejs
  - Dockerfile: ./Dockerfile  

- **INTERACTING** with the copilot app *environment*
```
$ copilot env show -n test
About

  Name        test
  Region      us-west-2
  Account ID  349327579537

Workloads

  Name              Type
  ----              ----
  ecsdemo-frontend  Load Balanced Web Service
  ecsdemo-nodejs    Backend Service

Tags

  Key                  Value
  ---                  -----
  copilot-application  ecsworkshop
  copilot-environment  test
```

- **INTERACTING** with the copilot app *service* - nodejs backend service
```
$ copilot svc status -n ecsdemo-nodejs
SERVICE ecsdemo-nodejs found in environment test
Task Summary

  Running   ██████████  1/1 desired tasks are running
  Health    ██████████  1/1 passes container health checks

Tasks

  ID        Status      Revision    Started At      Cont. Health
  --        ------      --------    ----------      ------------
  97f41bc8  RUNNING     1           19 minutes ago  HEALTHY
```

- **INTERACTING** with the copilot app service
```
$ copilot svc status -n ecsdemo-frontend
SERVICE ecsdemo-frontend found in environment test
Task Summary

  Running   ██████████  3/3 desired tasks are running
  Health    ██████████  3/3 passes HTTP health checks

Tasks

  ID        Status      Revision    Started At  HTTP Health
  --        ------      --------    ----------  -----------
  06625937  RUNNING     3           1 hour ago  HEALTHY
  558639b8  RUNNING     3           1 hour ago  HEALTHY
  5e7d8ee3  RUNNING     3           1 hour ago  HEALTHY
```

- **SCALE OUT** the number of service containers; open the manifest file 
  (`./copilot/ecsdemo-nodejs/manifest.yml`), and replace the value of the 
  count key from 1 to 3.
```
# Number of tasks that should be running in your service.
count: 3
```

- **SCALE OUT** - *Redeploy* the service
```
$ copilot svc deploy
Only found one service, defaulting to: ecsdemo-nodejs
Only found one environment, defaulting to: test
Sending build context to Docker daemon  111.1kB
Step 1/10 : FROM public.ecr.aws/docker/library/ubuntu:18.04
 ---> 35b3f4f76a24
Step 2/10 : ARG NODE=production
 ---> Using cache
 ---> 9e947039055b
Step 3/10 : ENV NODE_ENV ${NODE}
 ---> Using cache
 ---> 0f1524974744
Step 4/10 : WORKDIR /usr/src/app
 ---> Using cache
 ---> a13deaa4d570
Step 5/10 : COPY package*.json ./
 ---> Using cache
 ---> 47e8529bb805
Step 6/10 : RUN apt-get update &&    apt install -y curl jq bash nodejs npm python3 python3-pip &&     pip3 install awscli netaddr &&     npm install &&    apt-get purge -y npm &&     apt clean
 ---> Using cache
 ---> 0f34ef437c24
Step 7/10 : COPY . .
 ---> bc54b81853b0
Step 8/10 : HEALTHCHECK --interval=10s --timeout=3s   CMD curl -f -s http://localhost:3000/health/ || exit 1
 ---> Running in 3ef7a76831f2
Removing intermediate container 3ef7a76831f2
 ---> c11814d2857a
Step 9/10 : EXPOSE 3000
 ---> Running in f1e6134560e0
Removing intermediate container f1e6134560e0
 ---> a5df95e41345
Step 10/10 : ENTRYPOINT ["bash","/usr/src/app/startup.sh"]
 ---> Running in 70c5fa4a646b
Removing intermediate container 70c5fa4a646b
 ---> fbf5daa16bd4
Successfully built fbf5daa16bd4
Successfully tagged 349327579537.dkr.ecr.us-west-2.amazonaws.com/ecsworkshop/ecsdemo-nodejs:latest
WARNING! Your password will be stored unencrypted in /home/gabe/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
Using default tag: latest
The push refers to repository [349327579537.dkr.ecr.us-west-2.amazonaws.com/ecsworkshop/ecsdemo-nodejs]
dabf728256d8: Pushed 
0841d1a6ab5a: Layer already exists 
7ec566b4b715: Layer already exists 
a76cfd270838: Layer already exists 
4a641e21953d: Layer already exists 
latest: digest: sha256:2a7c50f867c91c28c6da06ef6ae054ed668653ea777cdb7545bc873604f2d51c size: 1366
✔ Proposing infrastructure changes for stack ecsworkshop-test-ecsdemo-nodejs
- Updating the infrastructure for stack ecsworkshop-test-ecsdemo-nodejs       [update complete]  [188.8s]
  - An ECS service to run and maintain your tasks in the environment cluster  [update complete]  [161.1s]
    Deployments                                                                                   
               Revision  Rollout      Desired  Running  Failed  Pending                                   
      PRIMARY  2         [completed]  3        3        0       0                                         
  - An ECS task definition to group your containers and run them on ECS       [delete complete]  [3.9s]
✔ Deployed service ecsdemo-nodejs.
Recommended follow-up action:
  - You can access your service at ecsdemo-nodejs.test.ecsworkshop.local:3000 with service discovery.
```

- **SCALE OUT** - check our service details
```
$ copilot svc status -n ecsdemo-nodejs
SERVICE ecsdemo-nodejs found in environment test
Task Summary

  Running   ██████████  3/3 desired tasks are running
  Health    ██████████  3/3 passes container health checks

Tasks

  ID        Status      Revision    Started At     Cont. Health
  --        ------      --------    ----------     ------------
  5886fd12  RUNNING     2           2 minutes ago  HEALTHY
  863afa63  RUNNING     2           2 minutes ago  HEALTHY
  ba5bf980  RUNNING     2           2 minutes ago  HEALTHY
```
  You should now see three tasks running!
  Now go back to the load balancer url, and you should see the diagram
  alternate between the three nodejs tasks.

- Grab the load balancer url and paste it into your browser.
```
$ copilot svc show -n ecsdemo-frontend
About

  Application  ecsworkshop
  Name         ecsdemo-frontend
  Type         Load Balanced Web Service

Configurations

  Environment  Tasks     CPU (vCPU)  Memory (MiB)  Platform      Port
  -----------  -----     ----------  ------------  --------      ----
  test         3         0.25        512           LINUX/X86_64  3000

Routes

  Environment  URL
  -----------  ---
  test         http://ecswo-Publi-I78Y270HWMBY-626812307.us-west-2.elb.amazonaws.com

Service Discovery

  Environment  Namespace
  -----------  ---------
  test         ecsdemo-frontend.test.ecsworkshop.local:3000

Variables

  Name                                Container         Environment  Value
  ----                                ---------         -----------  -----
  COPILOT_APPLICATION_NAME            ecsdemo-frontend  test         ecsworkshop
  COPILOT_ENVIRONMENT_NAME              "                 "          test
  COPILOT_LB_DNS                        "                 "          ecswo-Publi-I78Y270HWMBY-626812307.us-west-2.elb.amazonaws.com
  COPILOT_SERVICE_DISCOVERY_ENDPOINT    "                 "          test.ecsworkshop.local
  COPILOT_SERVICE_NAME                  "                 "          ecsdemo-frontend
  CRYSTAL_URL                           "                 "          http://ecsdemo-crystal.test.ecsworkshop.local:3000/crystal
  NODEJS_URL                            "                 "          http://ecsdemo-nodejs.test.ecsworkshop.local:3000
```

- **LOG** - review the service log; 
```
$ copilot svc logs -a ecsworkshop -n ecsdemo-nodejs
copilot/ecsdemo-nodejs/58 ::ffff:10.0.0.131 - - [09/Sep/2022:22:57:03 +0000] "GET http://ba5bf9808c9f430bb040c815a0f17bf9.ecsdemo-nodejs.test.ecsworkshop.local:3000 HTTP/1.1" 200 62 "-" "Ruby"
copilot/ecsdemo-nodejs/ba ::ffff:10.0.0.62 - - [09/Sep/2022:22:57:03 +0000] "GET http://863afa63507e4de38c329a23bf24be3b.ecsdemo-nodejs.test.ecsworkshop.local:3000 HTTP/1.1" 200 62 "-" "Ruby"
copilot/ecsdemo-nodejs/ba ::ffff:127.0.0.1 - - [09/Sep/2022:22:57:05 +0000] "GET /health/ HTTP/1.1" 200 18 "-" "curl/7.58.0"
copilot/ecsdemo-nodejs/86 ::ffff:127.0.0.1 - - [09/Sep/2022:22:57:07 +0000] "GET /health/ HTTP/1.1" 200 18 "-" "curl/7.58.0"
copilot/ecsdemo-nodejs/58 ::ffff:127.0.0.1 - - [09/Sep/2022:22:57:09 +0000] "GET /health/ HTTP/1.1" 200 18 "-" "curl/7.58.0"
copilot/ecsdemo-nodejs/ba ::ffff:127.0.0.1 - - [09/Sep/2022:22:57:15 +0000] "GET /health/ HTTP/1.1" 200 18 "-" "curl/7.58.0"
copilot/ecsdemo-nodejs/86 ::ffff:127.0.0.1 - - [09/Sep/2022:22:57:17 +0000] "GET /health/ HTTP/1.1" 200 18 "-" "curl/7.58.0"
copilot/ecsdemo-nodejs/58 ::ffff:127.0.0.1 - - [09/Sep/2022:22:57:19 +0000] "GET /health/ HTTP/1.1" 200 18 "-" "curl/7.58.0"
copilot/ecsdemo-nodejs/ba ::ffff:127.0.0.1 - - [09/Sep/2022:22:57:25 +0000] "GET /health/ HTTP/1.1" 200 18 "-" "curl/7.58.0"
copilot/ecsdemo-nodejs/86 ::ffff:127.0.0.1 - - [09/Sep/2022:22:57:27 +0000] "GET /health/ HTTP/1.1" 200 18 "-" "curl/7.58.0"
$ 
```

### Deploy the crystal backend service

- **Initialize** copilot application, but using the same application infrastructure,
  or "copilot application" as the "ecsdemo-frontend". 
```
$ cd <project_folder>/aws-ecs-workshop/ecsdemo-crystal
$ git rev-parse --short=7 HEAD
b6db623
$ git rev-parse --short=7 HEAD > code_hash.txt
$ copilot init
Welcome to the Copilot CLI! We're going to walk you through some questions
to help you get set up with a containerized application on AWS. An application is a collection of
containerized services that operate together.

Use existing application: Yes
Application name: ecsworkshop
Workload type: Backend Service
Service name: ecsdemo-crystal
Dockerfile: ./Dockerfile
Ok great, we'll set up a Backend Service named ecsdemo-crystal in application ecsworkshop listening on port 3000.

✔ Proposing infrastructure changes for stack ecsworkshop-infrastructure-roles
✔ The directory copilot will hold service manifests for application ecsworkshop.

✔ Wrote the manifest for service ecsdemo-crystal at copilot/ecsdemo-crystal/manifest.yml
Your manifest contains configurations like your container size and port (:3000).

- Update regional resources with stack set "ecsworkshop-infrastructure"  [succeeded]        [22.3s]
  - Update resources in region "us-west-2"                               [update complete]  [11.6s]
    - ECR container image repository for "ecsdemo-crystal"               [create complete]  [2.5s]
All right, you're all set for local development.
Deploy: Yes

✔ Wrote the manifest for environment test at copilot/environments/test/manifest.yml
- Update regional resources with stack set "ecsworkshop-infrastructure"  [succeeded]  [9.2s]
✔ Proposing infrastructure changes for the ecsworkshop-test environment.
✔ Provisioned bootstrap resources for environment test in region us-west-2 under application ecsworkshop.
✔ Provisioned bootstrap resources for environment test.
✔ Proposing infrastructure changes for the ecsworkshop-test environment.
✘ Your update does not introduce immediate resource changes. 
This may be because the resources are not created until they are deemed 
necessary by a service deployment.

[...]

Successfully built 640b8c0df9c9
Successfully tagged 349327579537.dkr.ecr.us-west-2.amazonaws.com/ecsworkshop/ecsdemo-crystal:latest
WARNING! Your password will be stored unencrypted in /home/gabe/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
Using default tag: latest
The push refers to repository [349327579537.dkr.ecr.us-west-2.amazonaws.com/ecsworkshop/ecsdemo-crystal]
b91380e780e3: Pushed 
774418a47f40: Pushed 
649488bd736d: Pushed 
8f673575e934: Pushed 
4a641e21953d: Pushed 
latest: digest: sha256:250237e2c7e22a9d10fb0a8262d2e2264c60c0d861fdcb63aeb57e7a6df63521 size: 1367
✔ Proposing infrastructure changes for stack ecsworkshop-test-ecsdemo-crystal
- Creating the infrastructure for stack ecsworkshop-test-ecsdemo-crystal      [create complete]  [105.4s]
  - Service discovery for your services to communicate within the VPC         [create complete]  [0.0s]
  - Update your environment's shared resources                                [create complete]  [2.7s]
  - An IAM role to update your environment stack                              [create complete]  [20.7s]
  - An IAM Role for the Fargate agent to make AWS API calls on your behalf    [create complete]  [18.5s]
  - A CloudWatch log group to hold your service logs                          [create complete]  [0.0s]
  - An ECS service to run and maintain your tasks in the environment cluster  [create complete]  [56.0s]
    Deployments                                                                                   
               Revision  Rollout      Desired  Running  Failed  Pending                                   
      PRIMARY  1         [completed]  1        1        0       0                                         
  - An ECS task definition to group your containers and run them on ECS       [create complete]  [0.0s]
  - An IAM role to control permissions for the containers in your tasks       [create complete]  [20.7s]
✔ Deployed service ecsdemo-crystal.
Recommended follow-up action:
  - You can access your service at ecsdemo-crystal.test.ecsworkshop.local:3000 with service discovery.
- Be a part of the Copilot ✨community✨!
  Ask or answer a question, submit a feature request...
  Visit 👉 https://aws.github.io/copilot-cli/community/get-involved/ to see how!
```
  **NOTE**, the responses to the copilot prompted series of questions:
  - Would you like to use one of your existing applications? “Y”
  - Which existing application do you want to add a new service to? Select “ecsworkshop”, hit enter
  - Which service type best represents your service’s architecture? Select “Backend Service”, hit enter
  - What do you want to name this Backend Service: ecsdemo-crystal
  - Dockerfile: ./Dockerfile  

- **INTERACTING** with the copilot *app*
```
$ copilot app show ecsworkshop
About

  Name     ecsworkshop
  Version  v1.0.2 
  URI      

Environments

  Name    AccountID     Region
  ----    ---------     ------
  test    349327579537  us-west-2

Workloads

  Name              Type                       Environments
  ----              ----                       ------------
  ecsdemo-crystal   Backend Service            test
  ecsdemo-frontend  Load Balanced Web Service  test
  ecsdemo-nodejs    Backend Service            test

Pipelines

  Name
  ----
```

- **INTERACTING** with the copilot app *environment*
```
$ copilot env show -n test
About

  Name        test
  Region      us-west-2
  Account ID  349327579537

Workloads

  Name              Type
  ----              ----
  ecsdemo-crystal   Backend Service
  ecsdemo-frontend  Load Balanced Web Service
  ecsdemo-nodejs    Backend Service

Tags

  Key                  Value
  ---                  -----
  copilot-application  ecsworkshop
  copilot-environment  test
```

- **INTERACTING** with the copilot app *service*
```
$ copilot svc status -n ecsdemo-crystal
SERVICE ecsdemo-crystal found in environment test
Task Summary

  Running   ██████████  1/1 desired tasks are running
  Health    ██████████  1/1 passes container health checks

Tasks

  ID        Status      Revision    Started At     Cont. Health
  --        ------      --------    ----------     ------------
  70640a70  RUNNING     1           3 minutes ago  HEALTHY
```

- **SCALE OUT** the number of service containers; open the manifest file 
  (`./copilot/ecsdemo-crystal/manifest.yml`), and replace the value of the 
  count key from 1 to 3.
```
# Number of tasks that should be running in your service.
count: 3
```

- **SCALE OUT** - Redeploy the service
```
$ copilot svc deploy
Only found one service, defaulting to: ecsdemo-crystal
Only found one environment, defaulting to: test
Sending build context to Docker daemon  79.36kB
Step 1/13 : FROM public.ecr.aws/aws-containers/crystal-dependency-image:0.26.1
 ---> 6a525b5b505a
Step 2/13 : WORKDIR /src/
 ---> Using cache
 ---> df1a1517ff51
Step 3/13 : COPY . .
 ---> 2004516fdd58
Step 4/13 : RUN shards install
 ---> Running in 97f32cf654f6

[...]

Login Succeeded
Using default tag: latest
The push refers to repository [349327579537.dkr.ecr.us-west-2.amazonaws.com/ecsworkshop/ecsdemo-crystal]
b91380e780e3: Layer already exists 
774418a47f40: Layer already exists 
649488bd736d: Layer already exists 
8f673575e934: Layer already exists 
4a641e21953d: Layer already exists 
latest: digest: sha256:250237e2c7e22a9d10fb0a8262d2e2264c60c0d861fdcb63aeb57e7a6df63521 size: 1367
✔ Proposing infrastructure changes for stack ecsworkshop-test-ecsdemo-crystal
- Updating the infrastructure for stack ecsworkshop-test-ecsdemo-crystal      [update complete]  [75.6s]
  - An ECS service to run and maintain your tasks in the environment cluster  [update complete]  [62.7s]
    Deployments                                                                                   
               Revision  Rollout      Desired  Running  Failed  Pending                                   
      PRIMARY  1         [completed]  3        3        0       0                                         
✔ Deployed service ecsdemo-crystal.
Recommended follow-up action:
  - You can access your service at ecsdemo-crystal.test.ecsworkshop.local:3000 with service discovery.
```

- **SCALE OUT** - check our service details
```
$ copilot svc status -n ecsdemo-crystal
SERVICE ecsdemo-crystal found in environment test
Task Summary

  Running   ██████████  3/3 desired tasks are running
  Health    ██████████  3/3 passes container health checks

Tasks

  ID        Status      Revision    Started At     Cont. Health
  --        ------      --------    ----------     ------------
  4e3622f8  RUNNING     1           1 minute ago   HEALTHY
  70640a70  RUNNING     1           8 minutes ago  HEALTHY
  ae0b54b4  RUNNING     1           1 minute ago   HEALTHY
```
  You should now see three tasks running!
  Now go back to the load balancer url, and you should see the diagram
  alternate between the three nodejs tasks.

- Grab the load balancer url and paste it into your browser.
```
$ copilot svc show -n ecsdemo-frontend
About

  Application  ecsworkshop
  Name         ecsdemo-frontend
  Type         Load Balanced Web Service

Configurations

  Environment  Tasks     CPU (vCPU)  Memory (MiB)  Platform      Port
  -----------  -----     ----------  ------------  --------      ----
  test         3         0.25        512           LINUX/X86_64  3000

Routes

  Environment  URL
  -----------  ---
  test         http://ecswo-Publi-I78Y270HWMBY-626812307.us-west-2.elb.amazonaws.com

Service Discovery

  Environment  Namespace
  -----------  ---------
  test         ecsdemo-frontend.test.ecsworkshop.local:3000

Variables

  Name                                Container         Environment  Value
  ----                                ---------         -----------  -----
  COPILOT_APPLICATION_NAME            ecsdemo-frontend  test         ecsworkshop
  COPILOT_ENVIRONMENT_NAME              "                 "          test
  COPILOT_LB_DNS                        "                 "          ecswo-Publi-I78Y270HWMBY-626812307.us-west-2.elb.amazonaws.com
  COPILOT_SERVICE_DISCOVERY_ENDPOINT    "                 "          test.ecsworkshop.local
  COPILOT_SERVICE_NAME                  "                 "          ecsdemo-frontend
  CRYSTAL_URL                           "                 "          http://ecsdemo-crystal.test.ecsworkshop.local:3000/crystal
  NODEJS_URL                            "                 "          http://ecsdemo-nodejs.test.ecsworkshop.local:3000
```

- **LOG** - review the service log; 
```
$ copilot svc logs -a ecsworkshop -n ecsdemo-crystal
SERVICE ecsdemo-crystal found in environment test
copilot/ecsdemo-crystal/a GET /health - 200 (3.79µs)
copilot/ecsdemo-crystal/4 GET /health - 200 (3.57µs)
copilot/ecsdemo-crystal/7 GET /health - 200 (4.2µs)
copilot/ecsdemo-crystal/a GET /health - 200 (3.66µs)
copilot/ecsdemo-crystal/4 GET /crystal - 200 (122.65µs)
copilot/ecsdemo-crystal/4 GET /crystal - 200 (63.2µs)
copilot/ecsdemo-crystal/7 GET /crystal - 200 (74.33µs)
copilot/ecsdemo-crystal/7 GET /crystal - 200 (150.53µs)
copilot/ecsdemo-crystal/a GET /crystal - 200 (174.17µs)
copilot/ecsdemo-crystal/7 GET /crystal - 200 (60.94µs)
```


## CLEANUP

- Copilot
```
$ cd <project_folder>/aws-ecs-workshop/ecsdemo-frontend
$ copilot app delete
```

