# Antennas - GITOPS repository

## Pre-requisites

* Create the required namespaces.

```sh
oc new-project antennas-dev
oc new-project antennas-test
oc new-project antennas-prod
```

* Install the OpenShift GitOps operator.

* Fix the ArgoCD ingress route in order to use the router default TLS certificate.

```sh
oc patch argocd openshift-gitops -n openshift-gitops -p '{"spec":{"server":{"insecure":true,"route":{"enabled": true,"tls":{"termination":"edge","insecureEdgeTerminationPolicy":"Redirect"}}}}}' --type=merge
```

* Get the Webhook URL of your OpenShift Gitops installation

```sh
oc get route -n openshift-gitops openshift-gitops-server -o jsonpath='https://{.spec.host}/api/webhook'
```

* Add a webhook to your GitHub/GitLab repo

  * Payload URL: *url above*
  * Content-Type: Application/json

* Label the `antennas-prod` namespace with argocd annotations

```sh
oc label namespace antennas-prod argocd.argoproj.io/managed-by=openshift-gitops
```

* Create the `antennas-prod` application.

```sh
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: antennas-prod
  namespace: openshift-gitops
spec:
  destination:
    name: ''
    namespace: antennas-prod
    server: 'https://kubernetes.default.svc'
  source:
    path: .
    repoURL: 'https://gitlab.com/nmasse-itix/antennas-gitops.git'
    targetRevision: HEAD
    helm:
      valueFiles:
      - values-prod.yaml
  project: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## Gitlab version

Install the Gitlab Runner operator.
And because there is a bug in the v1.10.0, you will have to install a beta version manually.

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: gitlab-runner-catalog
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: registry.gitlab.com/gitlab-org/gl-openshift/gitlab-runner-operator/gitlab-runner-operator-catalog-source:amd64-v0.0.1-53d8a4e6
  displayName: GitLab Runner Operators
  publisher: GitLab Community (Beta)
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: gitlab-runner-operator
  namespace: openshift-operators
spec:
  channel: stable
  name: gitlab-runner-operator
  source: gitlab-runner-catalog
  sourceNamespace: openshift-marketplace
```

Create a runner in the `antennas-dev` namespace.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-runner-secret
  namespace: antennas-dev
type: Opaque
stringData:
  runner-registration-token: REPLACE_ME # your project runner secret
---
apiVersion: apps.gitlab.com/v1beta2
kind: Runner
metadata:
  name: nmasse-itix
  namespace: antennas-dev
spec:
  gitlabUrl: https://gitlab.com
  token: gitlab-runner-secret
  tags: openshift, test
```

Go to your [Gitlab profile](https://gitlab.com/-/profile/personal_access_tokens) and generate a Personal Access Token with the **read_repository** and **write_repository**.

Go on [quay.io](https://quay.io/), click on your name, select **Account Settings** and generate an encrypted password.

On Gitlab, go in your repository's **Settings** > **CI/CD**.

Expand the **Variables** section.

Add two variables:

* `QUAY_USERNAME`: contains your Quay.io username
* `QUAY_PASSWORD`: contains your Quay.io encrypted password
* `GITLAB_ACCESS_TOKEN`: contains your GitLab Personal Access Token

Do not forget to add the **Masked** flag to the `QUAY_PASSWORD` and `GITLAB_ACCESS_TOKEN` variables!

Create two public repositories on quay.io

* antennas-front
* antennas-incident

Give the gitlab runner the right to manage the test namespace.

```sh
oc adm policy add-role-to-user admin system:serviceaccount:antennas-dev:default -n antennas-test
```

Give the gitlab runner the right to execute containers with any user.

```sh
oc adm policy add-scc-to-user anyuid -z default -n antennas-dev
```

Use the following `.gitlab-ci.yaml` file.

```yaml
# This file is a template, and might need editing before it works on your project.
# This is a sample GitLab CI/CD configuration file that should run without any modifications.
# It demonstrates a basic 3 stage CI/CD pipeline. Instead of real tests or scripts,
# it uses echo commands to simulate the pipeline execution.
#
# A pipeline is composed of independent jobs that run scripts, grouped into stages.
# Stages run in sequential order, but jobs within stages run in parallel.
#
# For more information, see: https://docs.gitlab.com/ee/ci/yaml/index.html#stages

stages:
  - build
  - test
  - deploy

default:
  tags:
    - "openshift"

#
# HEADS UP ! You will need to change those variables to match the location of
# your Quay.io repository and GitLab git repository.
#
variables:
  ANTENNAS_FRONT_IMAGE: quay.io/nmasse_itix/antennas-front
  ANTENNAS_GITOPS_REPOSITORY: gitlab.com/nmasse-itix/antennas-gitops.git

#
# Build the source code of antennas-front, using Maven.
#
maven-build:
  stage: build
  image: maven:3.8.6-jdk-11
  variables:
    # This will suppress any download for dependencies and plugins or upload messages which would clutter the console log.
    # `showDateTime` will show the passed time in milliseconds. You need to specify `--batch-mode` to make this work.
    MAVEN_OPTS: "-Dhttps.protocols=TLSv1.2 -Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository -Dorg.slf4j.simpleLogger.log.org.apache.maven.cli.transfer.Slf4jMavenTransferListener=WARN -Dorg.slf4j.simpleLogger.showDateTime=true -Djava.awt.headless=true"
    # As of Maven 3.3.0 instead of this you may define these options in `.mvn/maven.config` so the same config is used
    # when running from the command line.
    # `installAtEnd` and `deployAtEnd` are only effective with recent version of the corresponding plugins.
    MAVEN_CLI_OPTS: "--batch-mode --errors --fail-at-end --show-version -DinstallAtEnd=true -DdeployAtEnd=true"
  artifacts:
    paths:
      - target/
  # Cache downloaded dependencies and plugins between builds.
  # To keep cache across branches add 'key: "$CI_JOB_NAME"'
  cache:
    paths:
      - .m2/repository
  script:
    - mvn $MAVEN_CLI_OPTS clean package

#
# Clone the Git repository containing all the YAML manifests needed to deploy
# the complete application.
#
# Note that .git files do not fit very well with Gitlab CI artefact management,
# so we pack and unpack the git repo before and after each usage.
#
clone-gitops:
  stage: build
  image: docker.io/alpine/git:2.36.3
  artifacts:
    paths:
    - antennas-gitops.tgz
  script:
    - git clone https://ci:${GITLAB_ACCESS_TOKEN}@${ANTENNAS_GITOPS_REPOSITORY}
    - tar -zcf antennas-gitops.tgz antennas-gitops

#
# Build the container image of antennas-front using buildah.
#
# Note: the digest of the newly built image is written in "antennas-front.iid".
# this digest is then used to update the YAML files in the antennas-gitops
# repository.
#
buildah:
  stage: build
  variables:
    STORAGE_DRIVER: "vfs"
    BUILDAH_FORMAT: "docker"
  needs: [ "maven-build" ]
  image: quay.io/buildah/stable:v1.27
  artifacts:
    paths:
      - antennas-front.iid
  script:
    - buildah login -u "$QUAY_USERNAME" -p "$QUAY_PASSWORD" quay.io
    - buildah build -f src/main/docker/Dockerfile.jvm -t ${ANTENNAS_FRONT_IMAGE}:latest .
    - buildah push --tls-verify=false --digestfile antennas-front.iid ${ANTENNAS_FRONT_IMAGE}:latest

#
# Update the Helm values files to use the newly built image.
#
# Note: yq (https://mikefarah.gitbook.io/) is used to update values.yaml
#
helm-update:
  stage: build
  needs: [ "buildah", "clone-gitops" ]
  image: docker.io/mikefarah/yq:4.28.2
  artifacts:
    paths:
      - antennas-gitops.tgz
  script: |
    #!/bin/sh
    set -Eeuo pipefail

    # Get back the antennas-gitops repository from the Gitlab CI artifacts
    tar -zxf antennas-gitops.tgz

    # Update the Helm values files with the newly built image digest (and new name if not already done)
    ANTENNAS_FRONT_IMAGE_DIGEST="$(cat antennas-front.iid)"
    yq -i ".antennas-front.image.repository = \"${ANTENNAS_FRONT_IMAGE}\"" antennas-gitops/values-prod.yaml
    yq -i ".antennas-front.image.repository = \"${ANTENNAS_FRONT_IMAGE}\"" antennas-gitops/values-test.yaml
    yq -i ".antennas-front.image.digest = \"${ANTENNAS_FRONT_IMAGE_DIGEST}\"" antennas-gitops/values-prod.yaml
    yq -i ".antennas-front.image.digest = \"${ANTENNAS_FRONT_IMAGE_DIGEST}\"" antennas-gitops/values-test.yaml

    # Re-archive the antennas-gitops repository
    rm -f antennas-gitops.tgz
    tar -zcf antennas-gitops.tgz antennas-gitops

#
# Deploy the application in a test environment and wait for the deployment
# to complete.
#
deploy-test:
  stage: test
  needs: [ "helm-update" ]
  environment: test
  image: quay.io/openshift/origin-cli:4.11
  script: |
    #!/bin/sh
    set -Eeuo pipefail
    tar -zxf antennas-gitops.tgz

    # Download and install Helm
    curl -sf https://get.helm.sh/helm-v3.10.1-linux-amd64.tar.gz | tar -zx --strip-components 1 -C /usr/local/bin linux-amd64/helm

    # Generate the YAML manifests for the test environment
    helm dependency build antennas-gitops
    helm template antennas antennas-gitops --values antennas-gitops/values-test.yaml > test-manifests.yaml

    # Apply the YAML manifests
    oc apply -n antennas-test -f test-manifests.yaml

    # Wait for the deployment to complete
    oc rollout status deploy/antennas-front -n antennas-test --timeout=5m

#
# Run some integration tests in the test environment.
#
integration-test:
  stage: test
  needs: [ "deploy-test" ]
  image: registry.access.redhat.com/ubi8/ubi:8.6
  script:
    - echo "Running integration tests..."
    - curl -vf http://antennas-front.antennas-test.svc:8080/rest/incidents

#
# Commit the changes in the antennas-gitops repository so that ArgoCD can pick
# them up.
#
deploy-prod:
  stage: deploy
  needs: [ "helm-update", "integration-test" ]
  environment: production
  image: docker.io/alpine/git:2.36.3
  script: |
    #!/bin/sh
    set -Eeuo pipefail

    # Inject git credentials
    echo "https://ci:${GITLAB_ACCESS_TOKEN}@${ANTENNAS_GITOPS_REPOSITORY}" > ~/.git-credentials
    chmod 600 ~/.git-credentials

    tar -zxf antennas-gitops.tgz
    chown $(id -u):$(id -g) -R antennas-gitops
    cd antennas-gitops

    # Commit the changes to the antennas-gitops repository
    git config user.email "nmasse@redhat.com"
    git config user.name "Gitlab-CI Bot"
    git add values-prod.yaml
    git commit -m 'update production image digest'    
    git push
```

### Blue-green version

An updated chart is available in the `blue-green` branch of this repo.

And you can use the following `.gitlab-ci.yaml` file.

```yaml
# This file is a template, and might need editing before it works on your project.
# This is a sample GitLab CI/CD configuration file that should run without any modifications.
# It demonstrates a basic 3 stage CI/CD pipeline. Instead of real tests or scripts,
# it uses echo commands to simulate the pipeline execution.
#
# A pipeline is composed of independent jobs that run scripts, grouped into stages.
# Stages run in sequential order, but jobs within stages run in parallel.
#
# For more information, see: https://docs.gitlab.com/ee/ci/yaml/index.html#stages

stages:
  - build
  - test
  - deploy

default:
  tags:
    - "openshift"

#
# HEADS UP ! You will need to change those variables to match the location of
# your Quay.io repository and GitLab git repository.
#
variables:
  ANTENNAS_FRONT_IMAGE: quay.io/nmasse_itix/antennas-front
  ANTENNAS_GITOPS_REPOSITORY: gitlab.com/nmasse-itix/antennas-gitops.git

#
# Build the source code of antennas-front, using Maven.
#
maven-build:
  stage: build
  image: maven:3.8.6-jdk-11
  variables:
    # This will suppress any download for dependencies and plugins or upload messages which would clutter the console log.
    # `showDateTime` will show the passed time in milliseconds. You need to specify `--batch-mode` to make this work.
    MAVEN_OPTS: "-Dhttps.protocols=TLSv1.2 -Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository -Dorg.slf4j.simpleLogger.log.org.apache.maven.cli.transfer.Slf4jMavenTransferListener=WARN -Dorg.slf4j.simpleLogger.showDateTime=true -Djava.awt.headless=true"
    # As of Maven 3.3.0 instead of this you may define these options in `.mvn/maven.config` so the same config is used
    # when running from the command line.
    # `installAtEnd` and `deployAtEnd` are only effective with recent version of the corresponding plugins.
    MAVEN_CLI_OPTS: "--batch-mode --errors --fail-at-end --show-version -DinstallAtEnd=true -DdeployAtEnd=true"
  artifacts:
    paths:
      - target/
  # Cache downloaded dependencies and plugins between builds.
  # To keep cache across branches add 'key: "$CI_JOB_NAME"'
  cache:
    paths:
      - .m2/repository
  script:
    - mvn $MAVEN_CLI_OPTS clean package

#
# Clone the Git repository containing all the YAML manifests needed to deploy
# the complete application.
#
# Note that .git files do not fit very well with Gitlab CI artefact management,
# so we pack and unpack the git repo before and after each usage.
#
clone-gitops:
  stage: build
  image: docker.io/alpine/git:2.36.3
  artifacts:
    paths:
    - antennas-gitops.tgz
  script:
    - git clone https://ci:${GITLAB_ACCESS_TOKEN}@${ANTENNAS_GITOPS_REPOSITORY}
    - tar -zcf antennas-gitops.tgz antennas-gitops

#
# Build the container image of antennas-front using buildah.
#
# Note: the digest of the newly built image is written in "antennas-front.iid".
# this digest is then used to update the YAML files in the antennas-gitops
# repository.
#
buildah:
  stage: build
  variables:
    STORAGE_DRIVER: "vfs"
    BUILDAH_FORMAT: "docker"
  needs: [ "maven-build" ]
  image: quay.io/buildah/stable:v1.27
  artifacts:
    paths:
      - antennas-front.iid
  script:
    - buildah login -u "$QUAY_USERNAME" -p "$QUAY_PASSWORD" quay.io
    - buildah build -f src/main/docker/Dockerfile.jvm -t ${ANTENNAS_FRONT_IMAGE}:latest .
    - buildah push --tls-verify=false --digestfile antennas-front.iid ${ANTENNAS_FRONT_IMAGE}:latest

#
# Update the Helm values files to use the newly built image.
#
# Note: yq (https://mikefarah.gitbook.io/) is used to update values.yaml
#
helm-update:
  stage: build
  needs: [ "buildah", "clone-gitops" ]
  image: docker.io/mikefarah/yq:4.28.2
  artifacts:
    paths:
      - antennas-gitops.tgz
  script: |
    #!/bin/sh
    set -Eeuo pipefail

    # Get back the antennas-gitops repository from the Gitlab CI artifacts
    tar -zxf antennas-gitops.tgz

    # Find the current target (green or blue?)
    CURRENT_TARGET_PROD="$(yq .route.target antennas-gitops/values-prod.yaml)"
    CURRENT_TARGET_TEST="$(yq .route.target antennas-gitops/values-test.yaml)"

    # Pick the opposite target (prod)
    case "$CURRENT_TARGET_PROD" in
    blue)
      TARGET_PROD=green
      ;;
    green)
      TARGET_PROD=blue
      ;;
    *)
      echo "Error: unexpected value"
      ;;
    esac

    # Pick the opposite target (test)
    case "$CURRENT_TARGET_TEST" in
    blue)
      TARGET_TEST=green
      ;;
    green)
      TARGET_TEST=blue
      ;;
    *)
      echo "Error: unexpected value"
      ;;
    esac

    # Update the Helm values files with the newly built image digest (and new name if not already done)
    ANTENNAS_FRONT_IMAGE_DIGEST="$(cat antennas-front.iid)"
    yq -i ".antennas-front-${TARGET_PROD}.image.repository = \"${ANTENNAS_FRONT_IMAGE}\"" antennas-gitops/values-prod.yaml
    yq -i ".antennas-front-${TARGET_TEST}.image.repository = \"${ANTENNAS_FRONT_IMAGE}\"" antennas-gitops/values-test.yaml
    yq -i ".antennas-front-${TARGET_PROD}.image.digest = \"${ANTENNAS_FRONT_IMAGE_DIGEST}\"" antennas-gitops/values-prod.yaml
    yq -i ".antennas-front-${TARGET_TEST}.image.digest = \"${ANTENNAS_FRONT_IMAGE_DIGEST}\"" antennas-gitops/values-test.yaml

    # Re-archive the antennas-gitops repository
    rm -f antennas-gitops.tgz
    tar -zcf antennas-gitops.tgz antennas-gitops

#
# Deploy the application in a test environment and wait for the deployment
# to complete.
#
deploy-test:
  stage: test
  needs: [ "helm-update" ]
  environment: test
  image: quay.io/openshift/origin-cli:4.11
  script: |
    #!/bin/sh
    set -Eeuo pipefail
    tar -zxf antennas-gitops.tgz

    # Download and install Helm
    curl -sf https://get.helm.sh/helm-v3.10.1-linux-amd64.tar.gz | tar -zx --strip-components 1 -C /usr/local/bin linux-amd64/helm

    # Generate the YAML manifests for the test environment
    helm dependency build antennas-gitops
    helm template antennas antennas-gitops --values antennas-gitops/values-test.yaml > test-manifests.yaml

    # Apply the YAML manifests
    oc apply -n antennas-test -f test-manifests.yaml

    # Wait for the deployment to complete
    oc rollout status deploy/antennas-front -n antennas-test --timeout=5m

#
# Run some integration tests in the test environment.
#
# Note: the test environment is hardcoded to always deploy on the "blue" target
#
integration-test:
  stage: test
  needs: [ "deploy-test" ]
  image: registry.access.redhat.com/ubi8/ubi:8.6
  script:
    - echo "Running integration tests..."
    - curl -vf http://antennas-front-blue.antennas-test.svc:8080/rest/incidents

#
# Commit the changes in the antennas-gitops repository so that ArgoCD can pick
# them up.
#
deploy-prod:
  stage: deploy
  needs: [ "helm-update", "integration-test" ]
  environment: production
  image: docker.io/alpine/git:2.36.3
  script: |
    #!/bin/sh
    set -Eeuo pipefail

    # Inject git credentials
    echo "https://ci:${GITLAB_ACCESS_TOKEN}@${ANTENNAS_GITOPS_REPOSITORY}" > ~/.git-credentials
    chmod 600 ~/.git-credentials

    tar -zxf antennas-gitops.tgz
    chown $(id -u):$(id -g) -R antennas-gitops
    cd antennas-gitops

    # Commit the changes to the antennas-gitops repository
    git config user.email "nmasse@redhat.com"
    git config user.name "Gitlab-CI Bot"
    git add values-prod.yaml
    git commit -m 'update production image digest'    
    git push
```
