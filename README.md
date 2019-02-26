# Building docker image for Mule Project using CircleCi

Before setup, we need to prepare to login [circle](https://circleci.com/vcs-authorize/) with `git` account.

1. Go `Add PROJECTS`
2. Chose you desired project and `setup or follow prject`
3. Prepare environment varaible in project setting. The variable are use in `config.yml`. If you don't want to put credentials info in `config.yml`, use environment variables. Eg:,

   - CIRCLE_PROJECT_REPONAME : www.docker-hube.com
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
│   │   ├── domains
│   │   │   ├── .gitkeep
│   │   ├── Dockerfile
│   │   └── wrapper.conf
└── mule-project.xml
└── pom.xml
```
Here `config.yml` configuration.

Prepare remote build environment to build the project and to create the docker image. As `circleci` supprot the `docker`, build enviroment will be use `docker` as below.  `circleci/openjdk:8-jdk` image already have `maven` with `java`.

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
Pull/down and cache the dependencies lib. For the first time, there is no cache. It will be download/pull form public repo(Eg, maven-public)

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
Build the mule-project using `maven` and deploy artifact without snapshoots, eg, `1.2.3-SNAPSHOT` 

```yml
     # run maven build and deploy artifact (no snapshots)
      - run: 
          name: remove snapshot version
          command: |
            if mvn -s .circleci/settings.xml -q -Dexec.executable="echo" -Dexec.args='${project.version}' --non-recursive exec:exec | grep -q "SNAPSHOT"; then mvn versions:set -DremoveSnapshot; fi;
            echo $(mvn -s .circleci/settings.xml -q -Dexec.executable="echo" -Dexec.args='${project.version}' --non-recursive exec:exec)-$(echo $CIRCLE_SHA1 | cut -c -7)-$CIRCLE_BUILD_NUM > tempvers.txt
            mvn versions:set -DnewVersion=$(cat tempvers.txt) 
      - run: mvn -s .circleci/settings.xml -DskipTests clean package
      - run: mvn -s .circleci/settings.xml -DskipTests -DaltSnapshotDeploymentRepository=nexus::default::$REPO_URL/maven-snapshots/ -DaltReleaseDeploymentRepository=nexus::default::$REPO_URL/maven-releases/ deploy
   
```

Build `docker` image for mule-project
```yml
      - run:  
          name: Build application Docker image
          command: |  
            cp target/*.zip docker/apps/
            docker login -u $DOCKER_USER -p $DOCKER_PASS $DOCKER_REPO
            docker build --no-cache -t $DOCKER_REPO/$DOCKER_REPO_ORG/$CIRCLE_PROJECT_REPONAME:$(cat tempvers.txt) docker

```

Finally, keep and save the created docker image to docker image registory. Example. `Docker Hub`, `JFrog`.
```yml
            docker push $DOCKER_REPO/$DOCKER_REPO_ORG/$CIRCLE_PROJECT_REPONAME:$(cat tempvers.txt)
```

If your project is java project, it will be like

* buld java project
* remove shapshot version
* build docker 
* keep and save docker image registory
   
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
      - 
        run: 
          command: |
              cp target/*.war docker/app/
              docker login -u $DOCKER_USER -p $DOCKER_PASS
              docker build --no-cache -t $DOCKER_USER/$CIRCLE_PROJECT_REPONAME:$(cat tempvers.txt) docker
              docker push $DOCKER_USER/$CIRCLE_PROJECT_REPONAME:$(cat tempvers.txt)
          name: "Build application Docker image"
```

