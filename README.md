# spring-docker-temp

## Configure spring-boot-maven-plugin to produce layered jars
```
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <layers>
                    <enabled>true</enabled>
                </layers>
            </configuration>
        </plugin>
    </plugins>
</build>
```

## Create Dockerfile
```
FROM quay.balgroupit.com/baloise-base-images/maven:3.6-jdk-8-alpine as builder
WORKDIR application

COPY pom.xml .
COPY src src

RUN mvn install

ARG JAR_FILE=target/*.jar
RUN mv $JAR_FILE application.jar
RUN java -Djarmode=layertools -jar application.jar extract

FROM quay.balgroupit.com/baloise-base-images/openjdk:8-jre-alpine3.9
WORKDIR application
COPY --from=builder application/dependencies/ ./
COPY --from=builder application/spring-boot-loader/ ./
COPY --from=builder application/snapshot-dependencies/ ./
COPY --from=builder application/application/ ./
ENTRYPOINT ["java", "org.springframework.boot.loader.JarLauncher"]
```

## Create Jenkinsfile
```
@Library('jenkins-shared-library@release') _

pipeline {

    agent {
        label 'podman'
    }

    options {
        skipStagesAfterUnstable()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '28'))
        timeout(time: 1, unit: 'HOURS')
        timestamps()
    }

    stages {
        stage("Build and Push") {
            steps {
                notifyBitBucket state: "INPROGRESS"
                script {
                    currentBuild.description = GIT_COMMIT
                }
                containerBuild(repository: 'customercontact/blackboard-rest-service', tags: [GIT_COMMIT])
            }
        }

        stage("Deploy to TEST") {
            when {
                branch 'master'
            }
            steps {
                containerDeploy(repositoryName: 'cat-non-prod', file: 'blackboard-rest-service-test/values.yaml', yamlPatches: ["blackboard-rest-service.image.tag": "${GIT_COMMIT}"])
            }
        }
    }

    post {
        success {
            notifyBitBucket state: "SUCCESSFUL"
        }

        fixed {
            mailTo status: "SUCCESS", actuator: true, recipients: [], logExtract: true
        }

        failure {
            notifyBitBucket state: "FAILED"
            mailTo status: "FAILURE", actuator: true, recipients: [], logExtract: true
        }
    }
}
```

## Add swagger-ui
```
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-ui</artifactId>
    <version>1.4.1</version>
</dependency>
```
### Adjust swagger path in application.proeprties
springdoc.swagger-ui.path=/
