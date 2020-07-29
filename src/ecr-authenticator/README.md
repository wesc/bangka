# ECR Authenticator

Tool for authenticating against AWS Elastic Container Registries.

Docker Swarm has an unfortunate design flaw wrt authentication against
container registries that use expiring tokens. When you deploy into a
Swarm cluster, the Docker client authenticates and then passes tokens
into Swarm, which then distributes the tokens to the worker nodes. The
worker nodes typically use the tokens immediately to download the
necessary image, however there are two situations in which this
doesn't happen:

1. If Swarm moves a task from one node to another and the token has
   expired, the new node won't be able to download an image if it
   doesn't already exist in its cache.

2. A new worker node that joins the cluster after the token has
   expired will not be able to fetch any images.

AWS ECR tokens expire every 12 hours, so there's a decent chance this
will happen. See more notes here:

- http://issamben.com/docker-swarm-ecr-auto-login/

- https://github.com/moby/moby/issues/31063

We employ a similar strategy to that detailed in the first
link. ecr-authenticator runs a service on a manager node. Every 4
hours, it authenticates against ECR and then updates any services
using images from the ECR repo.


## Configuration

There are two ways to set up AWS ECR authentication: 1. instance role,
or 2. programmatic user credentials.


### Instance Role

If you configure your Docker nodes to be associated with an instance
role, you can [apply an access
policy](https://docs.aws.amazon.com/AmazonECR/latest/userguide/repository-policies.html)
to allow that instance to authenticate with ECR.

You then should deploy ecr-authenticator:

```
version: "3.7"
services:
  ecr-authenticator:
    image: bangka/ecr-authenticator:0.2
    command: ecr-auth
    deploy:
      placement:
        constraints:
          - "node.role==manager"
      restart_policy:
        condition: any
        delay: 4h
    environment:
      AWS_DEFAULT_REGION: us-east-1
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```

Note that you may have to set `AWS_DEFAULT_REGION` for awscli to be
happy.

The `docker.sock` volume mount allows Docker within the container to
manipulate the host Docker installation.


### User Credentials

You can also supply ecr-authenticator with AWS user credentials that
have ECR access. The credentials file should have contents that look
like a bash script:

```
export AWS_DEFAULT_REGION=us-east-1
export AWS_ACCESS_KEY_ID=XXX
export AWS_SECRET_ACCESS_KEY=XXX
```

The credentials file is sourced before ecr-authenticator attempts to
authenticate. You can specify the location of the credentials in the
service definition:

```
version: "3.7"
services:
  ecr-authenticator:
    image: bangka/ecr-authenticator:0.2
    command: ecr-auth -a /run/secrets/ecr_auth_creds
    deploy:
      placement:
        constraints:
          - "node.role==manager"
      restart_policy:
        condition: any
        delay: 4h
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    secrets:
      - ecr_auth_creds
secrets:
  ecr_auth_creds:
    external: true
```

In this example, ecr-authenticator will look at the path
`/run/secrets/ecr_auth_creds`, and if it exists it will source it
before attempting authentication.


## System-Wide Docker Config

You can set up ecr-authenticator to authenticate on behalf of any user
on a manager node, but it takes a few more steps.

```
version: "3.7"
services:
  ecr-authenticator:
    image: bangka/ecr-authenticator:0.2
    command: ecr-auth -d /var/bangka/docker -g
    deploy:
      placement:
        constraints:
          - "node.role==manager"
      restart_policy:
        condition: any
        delay: 4h
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/bangka:/var/bangka
```

The `-d` option uses a config directory that in this case is mounted
on the host machine. The `-g` option tells ecr-authenticator to make
the config file group readable after it's done authenticating.

On the host machine, you can now configure:

```
$ export DOCKER_CONFIG=/var/bangka/docker
$ docker pull XXX.dkr.ecr...
```

Any user that is able to read the config file will be able to
automatically pull from ECR.


## Service Labels

ecr-authenticator knows which services to refresh by looking for the
`bangka.ecr-authenticator` label and a corresponding value of 1. For
example, this will deploy a service that ecr-authenticator will
refresh:

```
version: '3.7'
services:
  api:
    image: XXX.dkr.ecr.us-east-1.amazonaws.com/repo/app:v1
    deploy:
      labels:
        bangka.ecr-authenticator: 1
```

The update is a no-op if a service definition has not changed, but it
does allow Swarm to acquire a fresh ECR token. Note that since this
runs every four hours, if you use a floating "latest" tag on your
image, your service will upgrade itself without your intervention,
which is likely not desired behavior. This is an anti-pattern anyway,
so the solution is to not use floating "latest" tags and either deploy
your services by hash or by version labels.

If you need to authenticate against multiple ECR repos, set up another
ecr-authentiactor service.


## Development

It's easiest to use Docker Compose to develop. You will need to create
a file in the `secrets/` directory named `aws_credentials`. Don't
worry, `.gitignore` is set up to ignore the contents of the `secrets/`
directory.
