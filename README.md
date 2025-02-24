# on-premise-gitlab-ci-portfolio

![Image](https://github.com/user-attachments/assets/63f421ca-bfcf-4062-be40-346908443391)

## .gitlab-ci

```yaml
image: docker:23.0.1

variables:
  RUNNER_TAG: "Backend Group Runner"
  DOCKER_DRIVER: overlay2
  DOCKER_REGISTRY: "docker-pri.maum.ai:443"
  DOCKER_IMAGE: "backend/llm-onpremise-management-system"
  DOCKER_COMPOSE_PATH: "/DATA/scripts/llm-serving"
  REPOSITORY_IMAGE: "$DOCKER_REGISTRY/$DOCKER_IMAGE:$CI_COMMIT_REF_NAME"

stages:
  - init
  - build
  - test
  - package
  - deploy

# Initialization stage for CI/CD pipeline
init:
  stage: init
  tags:
    - $RUNNER_TAG
  script:
    - |
      echo "Initializing CI/CD pipeline..."
      echo "Branch: ${CI_COMMIT_BRANCH}"
      echo "CI/CD initialization complete."
  rules:
    - if: '$CI_COMMIT_BRANCH == "cicd"'

# Build Stage
build:
  stage: build
  tags:
    - $RUNNER_TAG
  image: gradle:7.4-jdk11
  before_script:
    - chmod +x ./gradlew
  cache:
    key: gradle-cache-$CI_COMMIT_REF_SLUG
    paths:
      - .gradle/ # Cache Gradle artifacts
  script:
    - ./gradlew clean generateProto build -x test
  artifacts:
    paths:
      - build/
    expire_in: 7 days
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop" || 
          $CI_COMMIT_TAG || 
          $CI_PIPELINE_SOURCE == "merge_request_event"'

# Test Stage
test:
  stage: test
  tags:
    - $RUNNER_TAG
  image: gradle:7.4-jdk11
  before_script:
    - chmod +x ./gradlew
  cache:
    key: gradle-cache-$CI_COMMIT_REF_SLUG
    paths:
      - .gradle/ # Reuse Gradle cache
  script:
    - ./gradlew test
  dependencies:
    - build # build 단계에서의 artifacts를 이용하기 위해
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop" || 
          $CI_PIPELINE_SOURCE == "merge_request_event"'

# Package Stage
package:
  stage: package
  tags:
    - $RUNNER_TAG
  before_script:
    - echo "$CI_REGISTRY_PASSWORD" | docker login $DOCKER_REGISTRY -u "$CI_REGISTRY_USER" --password-stdin
  script:
    - docker build -t $REPOSITORY_IMAGE .
    - docker push $REPOSITORY_IMAGE
  dependencies:
    - build
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop" || 
          $CI_COMMIT_TAG'

# Deploy Stage
deploy:
  stage: deploy
  tags:
    - $RUNNER_TAG
  before_script:
    - mkdir -p ~/.ssh
    - eval $(ssh-agent -s)
    - echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_rsa
    - chmod 600 ~/.ssh/id_rsa
    - ssh-add ~/.ssh/id_rsa
    - ssh-keyscan -H $DEPLOY_SERVER_IP >> ~/.ssh/known_hosts
  script:
    - |
      ssh $DEPLOY_USER@$DEPLOY_SERVER_IP << EOF
      cd $DOCKER_COMPOSE_PATH
      sed -i 's|image: $DOCKER_REGISTRY/$DOCKER_IMAGE:[^ ]*|image: $REPOSITORY_IMAGE|' docker-compose.yml
      docker-compose pull
      docker-compose up -d --force-recreate
      EOF
  dependencies:
    - package
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'
```