---
title: "[ 🇬🇧 ] Gitlab CI to Build and Push containers to registry"
date: 2019-12-01
publishdate: 2019-12-01
dr  aft: false
---

We all know that Gitlab CI build uses docker image to do the job, But have you ever tried to build a docker image inside gitlab CI build ?

As we know gitlab CI start on docker container. So when we want to build a docker image inside gitlab CI build, it's docker in docker (DinD)

![](https://media.giphy.com/media/m1UTexVjvh2WA/giphy.gif)

---

Without transition lets take a look at the `.gitla-ci.yml`  file : 

```
    image: docker:latest
    variables:
     DOCKER_DRIVER: overlay2
    stages:
      - build
      - push
    services:
      - docker:dind
    before_script:
      # docker login needs the password to be passed through stdin for security
      # we use $CI_JOB_TOKEN here which is a special token provided by GitLab
      - echo -n $CI_JOB_TOKEN | docker login -u gitlab-ci-token --password-stdin $CI_REGISTRY
      - docker version
      - docker info
    
    after_script:
      - docker logout registry.gitlab.com
    Build:
      stage: build
      script:
        - docker pull $CI_REGISTRY_IMAGE:latest || true
        - >
          docker build
          --pull
          --cache-from $CI_REGISTRY_IMAGE:latest
          --tag $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
          .
        - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    Push_When_tag:
      stage: push
      only:
        # We want this job to be ran on tags only.
        - tags
      script:
        - docker pull $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
        - docker tag $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA $CI_REGISTRY_IMAGE:$CI_COMMIT_REF_NAME
        - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_REF_NAME
```

### Step 1 - Images & services
```
    image: docker:latest
    variables:
     DOCKER_DRIVER: overlay2
    stages:
      - build
      - push
    services:
      - docker:dind
```

We start by defining the docker image that will be used by GitlabCI build. In our case and as example we used the latest docker image . 

    image: docker:latest

> ⚠️  In production for example , I don't recommend using `latest`or`stable` versions. For many reasons ... 
One of them is reproducibility, Another reason is we want our pipeline to work in 10 month or 10 years. If a new feature is needed , then an upgrade is planned .
```
    variables:
     DOCKER_DRIVER: overlay2
```
When using `docker:dind` , Docker uses the `vfs` storage driver which copies the filesystem on every run. This is a very disk-intensive operation which can be avoided if a different driver is used, for example `overlay2`

### Step 2 - before and after Script
```
    before_script:
      # docker login needs the password to be passed through stdin for security
      # we use $CI_JOB_TOKEN here which is a special token provided by GitLab
      - echo -n $CI_JOB_TOKEN | docker login -u gitlab-ci-token --password-stdin $CI_REGISTRY
      - docker version
      - docker info
    
    after_script:
      - docker logout registry.gitlab.com
```
Nothing special on this step  : 

- Connection to Gitlab Registry
- Check docker daemon and config
- Logout from docker registry

### Step 3 -  Build and Push
```
    Build:
      stage: build
      script:
        - docker pull $CI_REGISTRY_IMAGE:latest || true
        
        # notice the cache-from, which is going to use the image we just pulled locally
        # the built image is tagged locally with the commit SHA, and then pushed to 
        # the GitLab registry
        - >
          docker build
          --pull
          --cache-from $CI_REGISTRY_IMAGE:latest
          --tag $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
          .
        - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
```

We pull the last pushed image on the registry ; the `|| true` assure that the pipeline will not fail if no image was found . 

After pulling the last image , this one will be used for the cache when building a new image using the `--cache-from` . 

Then we push to registry with the image flagged with `$CI_COMMIT_SHA` that contains the commit SHA . 

### Step 4 - Tag management
```
    Push_When_tag:
      stage: push
      only:
        - tags
      script:
        - docker pull $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
        - docker tag $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA $CI_REGISTRY_IMAGE:$CI_COMMIT_REF_NAME
        - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_REF_NAME
```
We want to keep  our Git tags in sync with our Docker tags. That helps a lot when debugging and trying to reproduce specific version bugs . 

If you have not automated this, you probably have found yourself in the situation of wondering “which git tag is this image again?”.

⚠️ This stage is triggered only when a tag is created .