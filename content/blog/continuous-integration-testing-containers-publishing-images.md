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
* [docker](https://docs.docker.com/get-docker/)

Although the docker installation should include Docker Compose, run the following command
to see if it was installed. The command should return the version.

```
$ docker-compose --version
docker-compose version 1.26.2, build eefe0d31
```

Make sure you've also signed up for a [Github account](https://github.com/), a
[Travis CI account](https://travis-ci.org/), and an [AWS account](https://aws.amazon.com/).

## The Source Code
Clone the git repository.
```
$ git clone https://github.com/rvlz/docker-ci-guide.git
```
Next enter the project directory and checkout the *demo* branch.
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
the test to add to *failed_tests* when the code is nonzero. Below the function invocation, the final command simply stops and removes all running containers.

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

The entire script should look like:
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

Let's run test the script.
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
