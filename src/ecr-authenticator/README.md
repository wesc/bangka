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

To set up AWS ECR authentication, insert into the Swarm cluster a
secret named `ecr_auth_creds` with contents:

```
export AWS_ECR_REPO=XXX.dkr.ecr.us-east-1.amazonaws.com
export AWS_DEFAULT_REGION=us-east-1
export AWS_ACCESS_KEY_ID=XXX
export AWS_SECRET_ACCESS_KEY=XXX
```

Where the access and secret keys are programmatic access keys
specifically for ECR authentication, thus with limited privileges.

Then, set up the authentication service for the Swarm cluster like so:

```
version: "3.7"
services:
  ecr-authenticator:
    image: bangka/ecr-authenticator
    command: ecr-auth ecr_auth_creds
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

If the secret exists, ecr-authenticator will source the contents of
the file (bash) and then attempt to log into ECR every four hours, or
whatever is specified in the restart policy. If the login is
successful, it will trigger a service update in the background for
services that use an image from `AWS_ECR_REPO`.

If you need to authenticate against multiple ECR repos, set up another
global service.

The update is a no-op if a service definition has not changed, but it
does allow Swarm to acquire a fresh ECR token. Note that since this
runs every four hours, if you use a floating "latest" tag on your
image, your service will upgrade itself without your intervention,
which is likely not desired behavior. The solution is to not use
floating "latest" tags and either deploy your services by hash or by
version labels.


## Development

It's easiest to use Docker Compose to develop. You will need to create
a file in the `secrets/` directory named `aws_credentials`. Don't
worry, `.gitignore` is set up to ignore the contents of the `secrets/`
directory.
