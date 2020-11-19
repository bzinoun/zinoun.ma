---
title: "[ 🇬🇧 ] -IBMCloud- K8S ready CI/CD toolchain"
date: 2019-04-01
publishdate: 2019-04-01
draft: false
---

As i am an Ibmcloud (Bluemix)user, i share with you a ready Devops toolchain that i use for my Kubernetes Deployment .

---

In the interest of efficiency and continuous deployment, the implementation of a ToolChain of continuous deployment remains a must. The latter is divided into 3 Steps :

### Step 1

- Validate docker file
- Build Docker image
- Push docker images to Registry

### Step2 : Non blocking Step

- Vulnerability scanning for the docker image
- Get the recommendations to fix them

### Step3

- validate deployment files
- deploy changes to Kubernetes Cluster

---

# 🛠 Toolchain

![](https://thepracticaldev.s3.amazonaws.com/i/jkgjvggy4xwpeyg1ch9f.png)

## Step 1 : Build

This step is responsible to verify docker file with a linter then build the image and push it to registry . 
Bellow we describe the shell scripte that do the stuff : 

### Verify Docker file

    #!/bin/bash
    echo "REGISTRY_URL=${REGISTRY_URL}"
    echo "REGISTRY_NAMESPACE=${REGISTRY_NAMESPACE}"
    echo "IMAGE_NAME=${IMAGE_NAME}"
    echo "ARCHIVE_DIR=${ARCHIVE_DIR}"
    echo "DOCKER_ROOT=${DOCKER_ROOT}"
    echo "DOCKER_FILE=${DOCKER_FILE}"
    
    # View build properties
    if [ -f build.properties ]; then 
      echo "build.properties:"
      cat build.properties
    else 
      echo "build.properties : not found"
    fi 
    
    echo "=========================================================="
    echo "Checking for Dockerfile at the repository root"
    if [ -z "${DOCKER_ROOT}" ]; then DOCKER_ROOT=. ; fi
    if [ -z "${DOCKER_FILE}" ]; then DOCKER_FILE=Dockerfile ; fi
    if [ -f ${DOCKER_ROOT}/${DOCKER_FILE} ]; then 
    echo -e "Dockerfile found at: ${DOCKER_FILE}"
    else
        echo "Dockerfile not found at: ${DOCKER_FILE}"
        exit 1
    fi
    echo "Linting Dockerfile"
    npm install -g dockerlint
    dockerlint -f ${DOCKER_ROOT}/${DOCKER_FILE}
    
    echo "=========================================================="
    echo "Checking registry current plan and quota"
    bx cr plan
    bx cr quota
    echo "If needed, discard older images using: bx cr image-rm"
    echo "Checking registry namespace: ${REGISTRY_NAMESPACE}"
    NS=$( bx cr namespaces | grep ${REGISTRY_NAMESPACE} ||: )
    if [ -z "${NS}" ]; then
        echo "Registry namespace ${REGISTRY_NAMESPACE} not found, creating it."
        bx cr namespace-add ${REGISTRY_NAMESPACE}
        echo "Registry namespace ${REGISTRY_NAMESPACE} created."
    else 
        echo "Registry namespace ${REGISTRY_NAMESPACE} found."
    fi
    echo -e "Existing images in registry"
    bx cr images --restrict ${REGISTRY_NAMESPACE}
    

### Build Docker Image

    #!/bin/bash
    echo "REGISTRY_URL=${REGISTRY_URL}"
    echo "REGISTRY_NAMESPACE=${REGISTRY_NAMESPACE}"
    echo "IMAGE_NAME=${IMAGE_NAME}"
    echo "BUILD_NUMBER=${BUILD_NUMBER}"
    echo "ARCHIVE_DIR=${ARCHIVE_DIR}"
    echo "GIT_BRANCH=${GIT_BRANCH}"
    echo "GIT_COMMIT=${GIT_COMMIT}"
    echo "DOCKER_ROOT=${DOCKER_ROOT}"
    echo "DOCKER_FILE=${DOCKER_FILE}"
    
    # View build properties
    if [ -f build.properties ]; then 
      echo "build.properties:"
      cat build.properties
    else 
      echo "build.properties : not found"
    fi 
    
    # To review or change build options use:
    echo -e "build --help"
    
     bx cr build --help
    
    echo -e "Existing images in registry"
    bx cr images
    
    # Minting image tag using format: BUILD_NUMBER--BRANCH-COMMIT_ID-TIMESTAMP
    # e.g. 3-master-50da6912-20181123114435
    # (use build number as first segment to allow image tag as a patch release name according to semantic versioning)
    
    TIMESTAMP=$( date -u "+%Y%m%d%H%M%S")
    IMAGE_TAG=${TIMESTAMP}
    if [ ! -z "${GIT_COMMIT}" ]; then
      GIT_COMMIT_SHORT=$( echo ${GIT_COMMIT} | head -c 8 ) 
      IMAGE_TAG=${GIT_COMMIT_SHORT}-${IMAGE_TAG}
    fi
    if [ ! -z "${GIT_BRANCH}" ]; then IMAGE_TAG=${GIT_BRANCH}-${IMAGE_TAG} ; fi
    IMAGE_TAG=${BUILD_NUMBER}-${IMAGE_TAG}
    echo "=========================================================="
    echo -e "BUILDING CONTAINER IMAGE: ${IMAGE_NAME}:${IMAGE_TAG}"
    if [ -z "${DOCKER_ROOT}" ]; then DOCKER_ROOT=. ; fi
    if [ -z "${DOCKER_FILE}" ]; then DOCKER_FILE=${DOCKER_ROOT}/Dockerfile ; fi
    
    
    #set -x
    #bx cr build -t ${REGISTRY_URL}/${REGISTRY_NAMESPACE}/${IMAGE_NAME}:${IMAGE_TAG} ${DOCKER_ROOT} -f ${DOCKER_FILE}
    #set +x
    
    bx cr build -t ${REGISTRY_URL}/${REGISTRY_NAMESPACE}/${IMAGE_NAME}:${TIMESTAMP} ${DOCKER_ROOT} -f ${DOCKER_FILE}
    
    
    bx cr image-inspect ${REGISTRY_URL}/${REGISTRY_NAMESPACE}/${IMAGE_NAME}:${TIMESTAMP} 
    
    # Set PIPELINE_IMAGE_URL for subsequent jobs in stage (e.g. Vulnerability Advisor)
    export PIPELINE_IMAGE_URL="$REGISTRY_URL/$REGISTRY_NAMESPACE/$IMAGE_NAME:$IMAGE_TAG"
    
    bx cr images --restrict ${REGISTRY_NAMESPACE}/${IMAGE_NAME}
    
    echo "=========================================================="
    echo "COPYING ARTIFACTS needed for deployment and testing (in particular build.properties)"
    
    echo "Checking archive dir presence"
    if [ -z "${ARCHIVE_DIR}" ]; then
      echo -e "Build archive directory contains entire working directory."
    else
      echo -e "Copying working dir into build archive directory: ${ARCHIVE_DIR} "
      mkdir -p ${ARCHIVE_DIR}
      find . -mindepth 1 -maxdepth 1 -not -path "./$ARCHIVE_DIR" -exec cp -R '{}' "${ARCHIVE_DIR}/" ';'
    fi
    
    # If already defined build.properties from prior build job, append to it.
    cp build.properties $ARCHIVE_DIR/ || :
    
    # IMAGE information from build.properties is used in Helm Chart deployment to set the release name
    echo "IMAGE_NAME=${IMAGE_NAME}" >> $ARCHIVE_DIR/build.properties
    echo "IMAGE_TAG=${IMAGE_TAG}" >> $ARCHIVE_DIR/build.properties
    # REGISTRY information from build.properties is used in Helm Chart deployment to generate cluster secret
    echo "REGISTRY_URL=${REGISTRY_URL}" >> $ARCHIVE_DIR/build.properties
    echo "REGISTRY_NAMESPACE=${REGISTRY_NAMESPACE}" >> $ARCHIVE_DIR/build.properties
    echo "GIT_BRANCH=${GIT_BRANCH}" >> $ARCHIVE_DIR/build.properties
    echo "TIMESTAMP=${TIMESTAMP}" >> $ARCHIVE_DIR/build.properties
     
    echo "File 'build.properties' created for passing env variables to subsequent pipeline jobs:"
    cat $ARCHIVE_DIR/build.properties

## Step 2 : Validate

## Security Check

    #!/bin/bash
    # uncomment to debug the script
    #set -x
    
    # View build properties
    if [ -f build.properties ]; then 
      echo "build.properties:"
      cat build.properties
    else 
      echo "build.properties : not found"
    fi 
    
    PIPELINE_IMAGE_URL=${REGISTRY_URL}/${REGISTRY_NAMESPACE}/${IMAGE_NAME}:${TIMESTAMP}
    
    echo "PIPELINE_IMAGE_URL=${PIPELINE_IMAGE_URL}"
    echo "REGISTRY_URL=${REGISTRY_URL}"
    echo "REGISTRY_NAMESPACE=${REGISTRY_NAMESPACE}"
    echo "IMAGE_NAME=${IMAGE_NAME}"
    echo "IMAGE_TAG=${IMAGE_TAG}"
    echo "TIMESTAMP=${TIMESTAMP}"
    
    bx cr images --restrict ${REGISTRY_NAMESPACE}/${IMAGE_NAME}
    echo -e "Checking vulnerabilities in image: ${PIPELINE_IMAGE_URL}"
    
    for ITER in {1..30}
    do
      set +e
      STATUS=$( bx cr va -e -o json ${PIPELINE_IMAGE_URL} | jq -r '.[0].status' )
      set -e
      # Possible status from Vulnerability Advisor: OK, UNSUPPORTED, INCOMPLETE, UNSCANNED, FAIL, WARN
      if [[ ${STATUS} != "INCOMPLETE" && ${STATUS} != "UNSCANNED" ]]; then
        break
      fi
      echo -e "${ITER} STATUS ${STATUS} : A vulnerability report was not found for the specified image."
      echo "Either the image doesn't exist or the scan hasn't completed yet. "
      echo "Waiting for scan to complete..."
      sleep 10
    done
    set +e
    bx cr va -e ${PIPELINE_IMAGE_URL}
    set -e
    STATUS=$( bx cr va -e -o json ${PIPELINE_IMAGE_URL} | jq -r '.[0].status' )
    [[ ${STATUS} == "OK" ]] || [[ ${STATUS} == "UNSUPPORTED" ]] || [[ ${STATUS} == "WARN" ]] || { echo "ERROR: The vulnerability scan was not successful, check the OUTPUT of the command and try again."; exit 1; }

## Step 3 : Deploy

## Verify K8S deployments file

    #!/bin/bash
    # uncomment to debug the script
    #set -x
    echo "IMAGE_NAME=${IMAGE_NAME}"
    echo "IMAGE_TAG=${IMAGE_TAG}"
    echo "REGISTRY_URL=${REGISTRY_URL}"
    echo "REGISTRY_NAMESPACE=${REGISTRY_NAMESPACE}"
    echo "DEPLOYMENT_FILE=${DEPLOYMENT_FILE}"
    
    # Fix : after getting error with IMAGE_TAG variable
    IMAGE_TAG=${TIMESTAMP}
    
    
    # Input env variables from pipeline job
    echo "PIPELINE_KUBERNETES_CLUSTER_NAME=${PIPELINE_KUBERNETES_CLUSTER_NAME}"
    if [ -z "${CLUSTER_NAMESPACE}" ]; then CLUSTER_NAMESPACE=default ; fi
    echo "CLUSTER_NAMESPACE=${CLUSTER_NAMESPACE}"
    
    echo "=========================================================="
    echo "DEPLOYING using manifest"
    echo -e "Updating ${DEPLOYMENT_FILE} with image name: ${REGISTRY_URL}/${REGISTRY_NAMESPACE}/${IMAGE_NAME}:${IMAGE_TAG}"
    if [ -z "${DEPLOYMENT_FILE}" ]; then DEPLOYMENT_FILE=deployment.yml ; fi
    if [ -f ${DEPLOYMENT_FILE} ]; then
        sed -i "s~^\([[:blank:]]*\)image:.*$~\1image: ${REGISTRY_URL}/${REGISTRY_NAMESPACE}/${IMAGE_NAME}:${IMAGE_TAG}~" ${DEPLOYMENT_FILE}
        cat ${DEPLOYMENT_FILE}
    else 
        echo -e "${red}Kubernetes deployment file '${DEPLOYMENT_FILE}' not found${no_color}"
        exit 1
    fi    
    set -x
    kubectl apply --namespace ${CLUSTER_NAMESPACE} -f ${DEPLOYMENT_FILE} 
    set +x
    
    echo ""
    echo "=========================================================="
    IMAGE_REPOSITORY=${REGISTRY_URL}/${REGISTRY_NAMESPACE}/${IMAGE_NAME}
    echo -e "CHECKING deployment status of ${IMAGE_REPOSITORY}:${IMAGE_TAG}"
    echo ""
    for ITERATION in {1..30}
    do
      DATA=$( kubectl get pods --namespace ${CLUSTER_NAMESPACE} -o json )
      NOT_READY=$( echo $DATA | jq '.items[].status | select(.containerStatuses!=null) | .containerStatuses[] | select(.image=="'"${IMAGE_REPOSITORY}:${IMAGE_TAG}"'") | select(.ready==false) ' )
      if [[ -z "$NOT_READY" ]]; then
        echo -e "All pods are ready:"
        echo $DATA | jq '.items[].status | select(.containerStatuses!=null) | .containerStatuses[] | select(.image=="'"${IMAGE_REPOSITORY}:${IMAGE_TAG}"'") | select(.ready==true) '
        break # deployment succeeded
      fi
      REASON=$(echo $DATA | jq '.items[].status | select(.containerStatuses!=null) | .containerStatuses[] | select(.image=="'"${IMAGE_REPOSITORY}:${IMAGE_TAG}"'") | .state.waiting.reason')
      echo -e "${ITERATION} : Deployment still pending..."
      echo -e "NOT_READY:${NOT_READY}"
      echo -e "REASON: ${REASON}"
      if [[ ${REASON} == *ErrImagePull* ]] || [[ ${REASON} == *ImagePullBackOff* ]]; then
        echo "Detected ErrImagePull or ImagePullBackOff failure. "
        echo "Please check proper authenticating to from cluster to image registry (e.g. image pull secret)"
        break; # no need to wait longer, error is fatal
      elif [[ ${REASON} == *CrashLoopBackOff* ]]; then
        echo "Detected CrashLoopBackOff failure. "
        echo "Application is unable to start, check the application startup logs"
        break; # no need to wait longer, error is fatal
      fi
      sleep 5
    done
    
    APP_NAME=$(kubectl get pods --namespace ${CLUSTER_NAMESPACE} -o json | jq -r '.items[].status | select(.containerStatuses!=null) | .containerStatuses[] | select(.image=="'"${IMAGE_REPOSITORY}:${IMAGE_TAG}"'") | .name' | head -n 1)
    echo -e "APP: ${APP_NAME}"
    echo "DEPLOYED PODS:"
    kubectl describe pods --selector app=${APP_NAME} --namespace ${CLUSTER_NAMESPACE}
    if [ ! -z "${APP_NAME}" ]; then
      APP_SERVICE=$(kubectl get services --namespace ${CLUSTER_NAMESPACE} -o json | jq -r ' .items[] | select (.spec.selector.app=="'"${APP_NAME}"'") | .metadata.name ')
      echo -e "SERVICE: ${APP_SERVICE}"
      echo "DEPLOYED SERVICES:"
      kubectl describe services ${APP_SERVICE} --namespace ${CLUSTER_NAMESPACE}
    fi
    #echo "Application Logs"
    #kubectl logs --selector app=${APP_NAME} --namespace ${CLUSTER_NAMESPACE}  
    echo ""
    if [[ ! -z "$NOT_READY" ]]; then
      echo ""
      echo "=========================================================="
      echo "DEPLOYMENT FAILED"
      exit 1
    fi
    
    echo ""
    echo "=========================================================="
    echo "DEPLOYMENT SUCCEEDED"
    if [ ! -z "${APP_SERVICE}" ]; then
      echo ""
      IP_ADDR=$(bx cs workers ${PIPELINE_KUBERNETES_CLUSTER_NAME} | grep normal | head -n 1 | awk '{ print $2 }')
      PORT=$( kubectl get services --namespace ${CLUSTER_NAMESPACE} | grep ${APP_SERVICE} | sed 's/.*:\([0-9]*\).*/\1/g' )
      echo ""
      echo -e "VIEW THE APPLICATION AT: http://${IP_ADDR}:${PORT}"
    fi

## Deploy to K8S

    #!/bin/bash
    # uncomment to debug the script
    #set -x

    echo "IMAGE_NAME=${IMAGE_NAME}"
    echo "IMAGE_TAG=${IMAGE_TAG}"
    echo "REGISTRY_URL=${REGISTRY_URL}"
    echo "REGISTRY_NAMESPACE=${REGISTRY_NAMESPACE}"
    echo "DEPLOYMENT_FILE=${DEPLOYMENT_FILE}"
    
    IMAGE_TAG=${TIMESTAMP}
       
    # Input env variables from pipeline job
    echo "PIPELINE_KUBERNETES_CLUSTER_NAME=${PIPELINE_KUBERNETES_CLUSTER_NAME}"
    if [ -z "${CLUSTER_NAMESPACE}" ]; then CLUSTER_NAMESPACE=default ; fi
    echo "CLUSTER_NAMESPACE=${CLUSTER_NAMESPACE}"
    
    echo "=========================================================="
    echo "DEPLOYING using manifest"
    echo -e "Updating ${DEPLOYMENT_FILE} with image name: ${REGISTRY_URL}/${REGISTRY_NAMESPACE}/${IMAGE_NAME}:${IMAGE_TAG}"
    if [ -z "${DEPLOYMENT_FILE}" ]; then DEPLOYMENT_FILE=deployment.yml ; fi
    if [ -f ${DEPLOYMENT_FILE} ]; then
        sed -i "s~^\([[:blank:]]*\)image:.*$~\1image: ${REGISTRY_URL}/${REGISTRY_NAMESPACE}/${IMAGE_NAME}:${IMAGE_TAG}~" ${DEPLOYMENT_FILE}
        cat ${DEPLOYMENT_FILE}
    else 
        echo -e "${red}Kubernetes deployment file '${DEPLOYMENT_FILE}' not found${no_color}"
        exit 1
    fi    
    set -x
    kubectl apply --namespace ${CLUSTER_NAMESPACE} -f ${DEPLOYMENT_FILE} 
    set +x
    
    echo ""
    echo "=========================================================="
    IMAGE_REPOSITORY=${REGISTRY_URL}/${REGISTRY_NAMESPACE}/${IMAGE_NAME}
    echo -e "CHECKING deployment status of ${IMAGE_REPOSITORY}:${IMAGE_TAG}"
    echo ""
    for ITERATION in {1..30}
    do
      DATA=$( kubectl get pods --namespace ${CLUSTER_NAMESPACE} -o json )
      NOT_READY=$( echo $DATA | jq '.items[].status | select(.containerStatuses!=null) | .containerStatuses[] | select(.image=="'"${IMAGE_REPOSITORY}:${IMAGE_TAG}"'") | select(.ready==false) ' )
      if [[ -z "$NOT_READY" ]]; then
        echo -e "All pods are ready:"
        echo $DATA | jq '.items[].status | select(.containerStatuses!=null) | .containerStatuses[] | select(.image=="'"${IMAGE_REPOSITORY}:${IMAGE_TAG}"'") | select(.ready==true) '
        break # deployment succeeded
      fi
      REASON=$(echo $DATA | jq '.items[].status | select(.containerStatuses!=null) | .containerStatuses[] | select(.image=="'"${IMAGE_REPOSITORY}:${IMAGE_TAG}"'") | .state.waiting.reason')
      echo -e "${ITERATION} : Deployment still pending..."
      echo -e "NOT_READY:${NOT_READY}"
      echo -e "REASON: ${REASON}"
      if [[ ${REASON} == *ErrImagePull* ]] || [[ ${REASON} == *ImagePullBackOff* ]]; then
        echo "Detected ErrImagePull or ImagePullBackOff failure. "
        echo "Please check proper authenticating to from cluster to image registry (e.g. image pull secret)"
        break; # no need to wait longer, error is fatal
      elif [[ ${REASON} == *CrashLoopBackOff* ]]; then
        echo "Detected CrashLoopBackOff failure. "
        echo "Application is unable to start, check the application startup logs"
        break; # no need to wait longer, error is fatal
      fi
      sleep 5
    done
    
    APP_NAME=$(kubectl get pods --namespace ${CLUSTER_NAMESPACE} -o json | jq -r '.items[].status | select(.containerStatuses!=null) | .containerStatuses[] | select(.image=="'"${IMAGE_REPOSITORY}:${IMAGE_TAG}"'") | .name' | head -n 1)
    echo -e "APP: ${APP_NAME}"
    echo "DEPLOYED PODS:"
    kubectl describe pods --selector app=${APP_NAME} --namespace ${CLUSTER_NAMESPACE}
    if [ ! -z "${APP_NAME}" ]; then
      APP_SERVICE=$(kubectl get services --namespace ${CLUSTER_NAMESPACE} -o json | jq -r ' .items[] | select (.spec.selector.app=="'"${APP_NAME}"'") | .metadata.name ')
      echo -e "SERVICE: ${APP_SERVICE}"
      echo "DEPLOYED SERVICES:"
      kubectl describe services ${APP_SERVICE} --namespace ${CLUSTER_NAMESPACE}
    fi
    #echo "Application Logs"
    #kubectl logs --selector app=${APP_NAME} --namespace ${CLUSTER_NAMESPACE}  
    echo ""
    if [[ ! -z "$NOT_READY" ]]; then
      echo ""
      echo "=========================================================="
      echo "DEPLOYMENT FAILED"
      exit 1
    fi
    
    echo ""
    echo "=========================================================="
    echo "DEPLOYMENT SUCCEEDED"
    if [ ! -z "${APP_SERVICE}" ]; then
      echo ""
      IP_ADDR=$(bx cs workers ${PIPELINE_KUBERNETES_CLUSTER_NAME} | grep normal | head -n 1 | awk '{ print $2 }')
      PORT=$( kubectl get services --namespace ${CLUSTER_NAMESPACE} | grep ${APP_SERVICE} | sed 's/.*:\([0-9]*\).*/\1/g' )
      echo ""
      echo -e "VIEW THE APPLICATION AT: http://${IP_ADDR}:${PORT}"
    fi

# Problems


### ⚠ Cluster kube always on "Pending"

After creating k8s cluster , the latter is still always in 'Pending' status

✅ Resolution : Update Kubernetes cluster to last proposed version 

⚠ Can't publish docker image on registry

On the BUILD step of the Toolchain FAILED , because of the name of the image 

    The supplied image name was invalid.
    Correct the image name, and try again.

✅  add a TIMESTAMP to docker image name . Replace the ${TAG_NAME} with ${TIMESTAMP}

---

That's all , thank you for reading and your feedback is highly appreciated 