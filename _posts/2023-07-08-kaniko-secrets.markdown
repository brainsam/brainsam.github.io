---
layout: post
title:  "Handle secrets in kaniko builds"
date:   2023-07-08 12:29:14 +0400
categories: docker
---
**Problem**: I need to build a docker image using the Kaniko project with a package installed from a private apt repository restricted by basic authentication, and to avoid credentials revealing.

From [apt_auth.conf](https://manpages.debian.org/testing/apt/apt_auth.conf.5.en.html) man page we can know how to setup apt to authenticate to the apt repository: to put file /etc/apt/auth.conf with the following content:
```
#/etc/apt/auth.conf
machine private.repo.com
login privatelogin
password privatepassword
```

So possible solution might be:
1. Write auth.conf file
2. Run the installation of the package and delete the file and clean the apt cache
in one layer.

Docker provides docker secrets feature, with Kaniko we can do the following:
```bash
read -s APT_LOGIN
read -s APT_PASSWORD
```

```bash
# write auth.conf
echo -e "machine deb-ng.keyr.net\nlogin ${APT_LOGIN}\npassword ${APT_PASSWORD}" \
> auth.conf

```

```bash
# run kaniko
docker run --rm \
-v `pwd`/auth.conf:/kaniko/auth.conf \ # providing auth.conf to kaniko
-v `pwd`:/workspace -v $HOME/.docker/config.json:/kaniko/.docker/config.json:ro \
gcr.io/kaniko-project/executor:v1.12.1-debug \
--dockerfile yabbuilder.dockerfile \
--destination "docker.ecp-share.com/yabbuilder:1.23.3" \
--context /workspace
```

And in Dockerfile we can run the following step
```Dockerfile
RUN --mount=type=secret,id=apt_auth cp /kaniko/auth.conf /etc/apt/auth.conf \
 && apt-get update && apt-get install private-package \
 && apt-get clean \
 && rm -f /etc/apt/auth.conf
```


GitLab job may look like:
```yaml
build:
  rules:
    - if: $CI_COMMIT_TAG
    - if: $CI_PIPELINE_SOURCE == 'merge_request_event'
      when: manual
  image:
    name: gcr.io/kaniko-project/executor:v1.12.1-debug
    entrypoint: [""]
  stage: build
  before_script:
    - mkdir -p /kaniko/.docker
    - |
      echo "{\"auths\":{\"docker.ecp-share.com\":{\"auth\":\"$(echo -n ${REGISTRY_USER}:${REGISTRY_PASSWORD} | base64)\"}}}" \
      > /kaniko/.docker/config.json
    - |
      echo -e "machine deb-ng.keyr.net\nlogin ${APT_LOGIN}\npassword ${APT_PASSWORD}" \
      > /kaniko/auth.conf
  script: >-
    /kaniko/executor
    --context $CI_PROJECT_DIR
    --dockerfile Dockerfile
    --destination "${image}:${tag}"
```

Thats it. Now after building, the resulting image and intermediate layers don't contain credentials.
