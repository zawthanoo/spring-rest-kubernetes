# Building docker image for Java Maven Project using CircleCi

Before setup, we need to prepare to login [circle](https://circleci.com/vcs-authorize/) with `git` account.

1. Go `Add PROJECTS`
2. Chose you desired project and `setup or follow project`
3. Prepare environment varaible in project setting. The variable are use in `config.yml`. If you don't want to put credentials info in `config.yml`, use environment variables. Eg:,

   - CIRCLE_PROJECT_REPONAME : https://hub.docker.com/
   - DOCKER_USER : exampleuser
   - DOCKER_PASS : examplepassword

### 1. `circleci` setup in project.

```bash
├── app-root
│   ├── .circleci
│   │   ├── config.yml
│   │   ├── setting.xml
│   ├── docker
│   │   ├── app  
│   │   │   ├── .gitkeep
│   │   ├── Dockerfile
│   │   └── wrapper.conf
└── mule-project.xml
└── pom.xml
```
Here `config.yml` configuration.

Prepare remote environment to build the project and to create the docker image. As `circleci` support the `docker`, build enviroment will be use `docker` as below.  `circleci/openjdk:8-jdk` image already have `maven` with `java`.

```yml
version: 2
jobs:
  build:
    docker:
      # specify the version you desire here
      - image: circleci/openjdk:8-jdk  
```

Prepare `working directory` and [environment](https://circleci.com/docs/2.0/java-oom/)

```yml
    working_directory: ~/repo

    environment:
      # Customize the JVM maximum heap limit
      MAVEN_OPTS: -Xmx3200m
```

Checkout git project and build remote environment

```yml
    steps:
      - checkout
      - setup_remote_docker
```
Pull/down and cache the dependencies lib. As there is no cache for firsttime build, it will be download/pull from public repo(Eg, maven-public)

```yml
      # Download and cache dependencies
      - restore_cache:
          keys:
          - v1-dependencies-{{ checksum "pom.xml" }}
          # fallback to using the latest cache if no exact match is found
          - v1-dependencies-
```
If you don't have custom/private repo, you can skip this.

```yml
      - run: mvn -s .circleci/settings.xml dependency:go-offline
```
save dependencies lib as cache. Next time, it will be pull/down the necessary lib.

```yml
      - save_cache:
          key: v1-dependencies-{{ checksum "pom.xml" }}
          paths:
            - ~/.m2
```
Build the project using maven and deploy artifact without snapshots-version, eg, 1.2.3-SNAPSHOT
   
```yml
      - 
        run: "mvn package"
      - 
        store_test_results: 
          path: target/surefire-reports
      - 
        store_artifacts: 
          path: target/<<your-project-name>>-0.0.1-SNAPSHOT.jar
      - 
        run: 
          name: "remove snapshot version"
          command: "if mvn -q -Dexec.executable=\"echo\" -Dexec.args='${project.version}' --non-recursive exec:exec | grep -q \"SNAPSHOT\"; then mvn versions:set -DremoveSnapshot; fi;\n\
              echo $(mvn -q -Dexec.executable=\"echo\" -Dexec.args='${project.version}' --non-recursive exec:exec)-$(echo $CIRCLE_SHA1 | cut -c -7)-$CIRCLE_BUILD_NUM > tempvers.txt\n\
              mvn versions:set -DnewVersion=$(cat tempvers.txt) \n"    
```

Build docker image according to Dockerfile which is under docker directory.

```yml
- 
        run: 
          name: "Build application Docker image"
          command: |
              cp target/*.war docker/app/
              docker login -u $DOCKER_USER -p $DOCKER_PASS
              docker build --no-cache -t $DOCKER_USER/$CIRCLE_PROJECT_REPONAME:$(cat tempvers.txt) docker
```
Finally, keep and save the created docker image to docker image registory. Example. `Docker Hub`, `JFrog`.
```yml
              docker push $DOCKER_USER/$CIRCLE_PROJECT_REPONAME:$(cat tempvers.txt)
```

# 2. Pull/Setup `docker` image

```Dockerfile
FROM tomcat:8.5.35
LABEL maintainer="Zaw Than Oo <zawthanoo.1986@gmail.com>"
COPY ./app/*.war /usr/local/tomcat/webapps
```
