---
title: "Continuous Integration: Testing Containers and Publishing Images"
date: 2020-08-20T12:26:39-05:00
draft: false
tags:
  - "continuous integration"
  - "docker"
  - "python"
  - "AWS"
---

In this guide, we'll use Travis CI to test containers and push images to Amazon Elastic
Container Registry. Before getting started make sure you have the following installed on
you machine:

* [git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
* [Docker](https://docs.docker.com/get-docker/)

Although the docker installation should include Docker Compose, run the following command
to see if it was installed. The command should return the version.

```
$ docker-compose --version
docker-compose version 1.26.2, build eefe0d31
```

Make sure you've also signed up for a [GitHub account](https://github.com/), a
[Travis CI account](https://travis-ci.org/), and an [AWS account](https://aws.amazon.com/).

## The Source Code
Clone the git repository.
```
$ git clone https://github.com/rvlz/docker-ci-guide.git
```
Next enter the project directory and check out the *demo* branch.
```
$ cd docker-ci-guide
$ git checkout demo
```

### Project Structure
Let's take a quick look at the project directory.
```
.
├── README.md
├── api
│   ├── Dockerfile
│   ├── app
│   │   ├── __init__.py
│   │   ├── config.py
│   │   ├── ping.py
│   │   └── test
│   │       ├── conftest.py
│   │       ├── test_config.py
│   │       └── test_ping.py
│   ├── manage.py
│   ├── requirements.txt
│   └── wsgi.py
├── docker-compose.yml
└── nginx
    ├── Dockerfile
    └── dev.conf
```
The two directories—*api* and *nginx*—in the project root each have source code for
a single container. In *api*, we have a very simple flask API and, in *nginx*, a reverse
proxy that relays all requests with the base path */api* to the API, as the server
block shows in *dev.conf*:
```
...

  location /api {
    proxy_pass http://api:5000;
    proxy_http_version 1.1;
    proxy_redirect    default;
    proxy_set_header  Host $host;
    proxy_set_header  X-Real-IP $remote_addr;
    proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header  X-Forwarded-Host $server_name;
  }

...
```
Now let's build the containers.

## Running Container Tests
To build the containers run the following command:
```
$ docker-compose up -d --build
```
If you don't already have the container images, it'll probably take a few minutes to
complete. But once you've downloaded the images the subsequent Docker Compose commands
won't take long to run.

Check that the containers are running.
```
$ docker-compose ps
         Name                      Command            State         Ports       
--------------------------------------------------------------------------------
docker-ci-guide_api_1     gunicorn -b 0.0.0.0:5000    Up      5000/tcp          
                          w ...                                                 
docker-ci-guide_nginx_1   nginx -g daemon off;        Up      0.0.0.0:80->80/tcp
```

Now let's run the API tests.
```
$ docker-compose exec api python manage.py test
============================== test session starts ===============================
platform linux -- Python 3.8.2, pytest-6.0.1, py-1.9.0, pluggy-0.13.1
rootdir: /code
collected 2 items                                                                

app/test/test_config.py .                                                  [ 50%]
app/test/test_ping.py .                                                    [100%]

=============================== 2 passed in 0.04s ================================
```

### Test Script

Since we don't want to type out these commands every time we test the API, we should
write a script to speed things up a bit.

Open up a file called *test.sh* with your favorite text editor. At the top of the file
write:

```
#!/bin/sh
```

This will tell the operating system to use the Bourne shell to execute *test.sh*.

Next add a helper function that adds the name of a failed test to the variable
*failed_tests*. We'll use *failed_tests* to determine the exit code of *test.sh*.
```
failed_tests=""

inspect() {
  if [ $1 -ne 0 ]; then
    failed_tests="${failed_tests} $2"
  fi
}
```
Now write the following commands:
```
docker-compose up -d --build
docker-compose exec api python manage.py test
inspect $? api
docker-compose down
```

The helper function takes the exit code of the previously executed command and the name of
the test to add to *failed_tests* when the code is nonzero. Below the function invocation,
the final command simply stops and removes all running containers.

Finally, add an *if/else* statement to choose the exit code for our script and to output
the results of the tests.
```
if [[ -n ${failed_tests} ]]; then
  echo "TESTS FAILED: ${failed_tests}"
  exit 1
else
  echo "TESTS SUCCEEDED"
  exit 0
fi
```

The entire script should look like this:
```
#!/bin/sh

failed_tests=""

inspect() {
  if [ $1 -ne 0 ]; then
    failed_tests="${failed_tests} $2"
  fi
}

docker-compose up -d --build
docker-compose exec api python manage.py test
inspect $? api
docker-compose down

if [[ -n ${failed_tests} ]]; then
  echo "TESTS FAILED: ${failed_tests}"
  exit 1
else
  echo "TESTS SUCCEEDED"
  exit 0
fi
```

Let's run the test script.
```
$ bash test.sh
============================== test session starts ===============================
platform linux -- Python 3.8.2, pytest-6.0.1, py-1.9.0, pluggy-0.13.1
rootdir: /code
collected 2 items                                                                

app/test/test_config.py .                                                  [ 50%]
app/test/test_ping.py .                                                    [100%]

=============================== 2 passed in 0.04s ================================
TESTS SUCCEEDED
```

All looks good. Before we commit our new file, create and check out a new branch called development.
```
$ git checkout -b development
```
Now let's commit the new file.
```
$ git add test.sh
$ git commit -m 'add API test script'
```

Since we won't be using the branches *master* and *demo*, we'll delete them.
```
$ git branch -d master demo
```

## Setting Up a Remote Repository
To test your code on Travis CI, you need to give Travis CI permission to access your source
code on GitHub. So let's create a remote repository called *docker-ci-demo*. If you need help
creating a GitHub repository follow these
[directions](https://docs.github.com/en/enterprise/2.15/user/articles/create-a-repo).


After creating the repository, replace the old remote repository with the newly created GitHub
repository.
```
git remote set-url origin https://github.com/<your-github-username>/docker-ci-demo.git
```

Next push to the GitHub repository.
```
$ git push -u origin development
```

From now on, as long as you're on the development branch you just need to run the following
command to push your changes.
```
$ git push
```

## Setting up Travis CI
Now that you have your GitHub repository you can connect your GitHub account to Travis CI
[here](https://travis-ci.org/signin). It'll require you to give Travis CI permission to
read your GitHub repositories. After you do, go to your
[dashboard](https://travis-ci.org/account/repositories), find the repository, and activate
it.

That's it!

## Travis CI Configuration
Every time you push any changes to the repository, Travis CI will look for a dot file called
*.travis.yml* in the project root. If it finds the file, it'll use it to run a build.

In your project root create .travis.yml and add the following content to it.

```
sudo: required

env:
  DOCKER_COMPOSE_VERSION: 1.25.4

before_install:
  - sudo rm /usr/local/bin/docker-compose
  - curl -L https://github.com/docker/compose/releases/download/${DOCKER_COMPOSE_VERSION}/docker-compose-`uname -s`-`uname -m` > docker-compose
  - chmod +x docker-compose
  - sudo mv docker-compose /usr/local/bin

script:
  - bash test.sh
```

The first line tells Travis CI to allow the running of commands using elevated permissions by using
*sudo* before the command. The next key *env* holds an environment variable called
*DOCKER_COMPOSE_VESRION* whose value Travis CI will insert wherever you reference it in the file. The
key that comes next may look quite complicate, but all it does is specify a sequence of commands that
replaces the local Docker Compose version with your own—in this case version 1.25.4. The final tells
Travis CI to run the script *test.sh*.

If all tests pass in *test.sh*, Travis CI will highlight the build in green; however, if it fails,
Travis CI will use red. But how does Travis CI know whether a build succeeded or not? Remember the
exit codes in the *if/else* statement in *test.sh*? After running the script, Travis CI will take a look
at the environment variable *?* and use its value to determine whether the script "passed"—zero means
success while a nonzero number means failure. Test it out!

Time to commit and push.
```
$ git add .travis.yml
$ git commit -m 'add Travis CI configuration'
$ git push
```

Go to Travis CI to see the build status. It'll probably take a minute or two to complete the build.

# Pushing to AWS Elastic Container Registry
First create and check out a new branch called *staging* which you'll use to publish new container
images. Whenever you're ready to release a new image, you'll switch to this branch and merge in
changes from the development branch. Then you'll push those changes to GitHub.

```
$ git checkout -b staging
```

Then push.

```
$ git push -u origin staging
```

Switch back to the *development* branch.

```
$ git checkout development
```

### Docker Push Script
To automate image publishing, you'll need a script that runs only when a *staging* branch build
succeeds, since you don't want to incur costs from frequent *development* branch builds.

Create a file called *docker-push.sh*.
```
$ touch docker-push.sh
```

Just like in *test.sh*, at the very top write:
```
#!/bin/sh
```

Next add an *if* statement block.
```
if [ "${TRAVIS_BRANCH}" == "staging" ]; then

  # code goes here

fi
```

On every build Travis CI provides useful environment variables to customize your build. Here you
use *TRAVIS_BRANCH* to run the nested code only on *staging* branch builds.

But before you can push to Amazon Elastic Container Registry, you need to log in. Unfortunately,
to do this you need the AWS CLI to communicate with AWS, but the virtual machines on which the builds
run don't have it installed. So you'll have to install it yourself. Inside the *if* statement add the
lines:

```
curl "https://s3.amazonaws.com/aws-cli/awscli-bundle.zip" -o "awscli-bundle.zip"
unzip awscli-bundle.zip
./awscli-bundle/install -b ~/bin/aws
export PATH=~/bin:${PATH}
```

Don't worry too much about the details. Just know that these commands install AWS CLI and add it
to the PATH variable.

Next, add the following line.
```
aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_REGISTRY}
``` 

The line above uses two environment variables, but to run the command you need a few
more: AWS_ACCOUNT_ID, AWS_ACCESS_KEY_ID, and AWS_SECRET_ACCESS_KEY. Since these variables hold
sensitive information, **never** include them in source code.

So how are you supposed to log in? Just let Travis CI inject them for you. Go to your Travis CI
dashboard and select docker-ci-demo. Click the *more options* button and select *zsettings*.
The dashboard should look something like this.


![Travis CI Repository Settings](/images/travis-ci-settings.png)

In the *Environment Variables* section add your aws information. Make sure that the variable values
aren't included in the build logs and that they are only available in the staging branch—just to be
extra cautious.

![Travis CI Environment Variables](/images/travis-ci-env-vars.png)

>> Remember to create an IAM user with sufficient permissions. Avoid using the root user to create
>> resources; it should only be used for billing purposes.

Finally, inside the *if/else* block add the commands that build a local image and push it to Amazon
Elastic Container Registry.

```
docker build ./api -t ${AWS_REGISTRY}/docker-ci-demo-api:staging -f ./api/Dockerfile
docker push ${AWS_REGISTRY}/docker-ci-demo-api:staging
```

The script should look like this.

```
#!/bin/sh

if [ "${TRAVIS_BRANCH}" == "staging" ]
then
  # Install and set up awscli
  curl "https://s3.amazonaws.com/aws-cli/awscli-bundle.zip" -o "awscli-bundle.zip"
  unzip awscli-bundle.zip
  ./awscli-bundle/install -b ~/bin/aws
  export PATH=~/bin:${PATH}

  # Add AWS_ACCOUNT_ID, AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY
  aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_REGISTRY}

  # Build and push API image
  docker build ./api -t ${AWS_REGISTRY}/docker-ci-demo-api:staging -f ./api/Dockerfile
  docker push ${AWS_REGISTRY}/docker-ci-demo-api:staging
fi

```

## Updating .travis.yml
To have Travis CI execute *docker-push.sh*, you need to update *.travis.yml*.

First, define AWS_REGION and AWS_REGISTRY in the *env* section. Be sure to replace
*\<your-region\>* with the region where you'll host the image repository.

```
env:
  ...
  AWS_REGION: <your-region>
  AWS_REGISTRY: ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
```

Then, define a new key called *after_success* and add *docker-push.sh* to it.

```
after_success:
  - docker-push.sh
```

.travis.yml should now look like this.

```
sudo: required

env:
  DOCKER_COMPOSE_VERSION: 1.25.4
  AWS_REGION: <your-region>
  AWS_REGISTRY: ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

before_install:
  - sudo rm /usr/local/bin/docker-compose
  - curl -L https://github.com/docker/compose/releases/download/${DOCKER_COMPOSE_VERSION}/docker-compose-`uname -s`-`uname -m` > docker-compose
  - chmod +x docker-compose
  - sudo mv docker-compose /usr/local/bin

script:
  - bash test.sh

after_success:
  - bash docker-push.sh

```

Now commit and push all the changes. (make sure you're on the *development* branch)

```
$ git add .
$ git commit -m 'add docker push script for successful staging builds'
$ git push
```

## Final Steps
If you merge in the changes to the *staging* branch and you push, the build will succeed;
however, your image push will end in failure. Why? Because the *docker-ci-demo-api* repository
doesn't exist on Amazon Elastic Container Registry! Head back to your AWS Management Console
and go to the Amazon Elastic Container Registry Console. Click on the *Create repository*
button and create a repository named *docker-ci-demo-api*.

![Amazon ECR](/images/aws-ecr.png)

Now switch to the *staging* branch, merge it with the *development* branch, and push.

```
$ git checkout staging
$ git merge development
$ git push
```

Wait a minute; then check the repository on AWS. Voila!

![Amazon ECR Repository](/images/aws-ecr-repo.png)

### Workflow
To be systemic, follow the steps below.

1. Switch to the *development* branch
2. Edit the API source code
3. Commit and push the changes
4. Switch to the *staging* branch
5. Merge the *development* branch
6. Push!

That's it for the tutorial! Try making a change to the source code and push your changes
to the Amazon Elastic Container Registry.

If you have any questions, suggestions, or concerns, shoot me an email at ricard@rvlz.io.
