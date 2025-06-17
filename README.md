plugins {
    id 'java'
    id 'org.springframework.boot' version '2.7.18'
    id 'io.spring.dependency-management' version '1.1.3'
    id 'com.palantir.docker' version '0.35.0'
}

group = 'com.example'
version = '1.0.0'
sourceCompatibility = '11'  // Spring Boot 2.x works best with Java 11

configurations {
    compileOnly {
        extendsFrom annotationProcessor
    }
}

repositories {
    mavenCentral()
}

ext {
    resilience4jVersion = '1.7.1'
    snowflakeJdbcVersion = '3.14.1'
}

dependencies {
    // Spring Boot Starters
    implementation 'org.springframework.boot:spring-boot-starter-batch'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-jdbc'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'org.springframework.boot:spring-boot-starter-aop'
    
    // Spring Cloud Task (for SCDF)
    implementation 'org.springframework.cloud:spring-cloud-starter-task:2.4.5'
    
    // Database & Snowflake
    implementation "net.snowflake:snowflake-jdbc:${snowflakeJdbcVersion}"
    implementation 'com.zaxxer:HikariCP:4.0.3' // Compatible with Spring Boot 2.7.x
    
    // REST Client
    implementation 'org.springframework:spring-webflux:5.3.29'
    implementation 'io.projectreactor.netty:reactor-netty:1.0.29'
    
    // Resilience4j (compatible with Spring Boot 2.7.x)
    implementation "io.github.resilience4j:resilience4j-spring-boot2:${resilience4jVersion}"
    implementation "io.github.resilience4j:resilience4j-reactor:${resilience4jVersion}"
    
    // JSON Processing
    implementation 'com.fasterxml.jackson.core:jackson-databind:2.13.5'
    implementation 'com.fasterxml.jackson.datatype:jackson-datatype-jsr310:2.13.5'
    
    // Utilities
    compileOnly 'org.projectlombok:lombok:1.18.28'
    annotationProcessor 'org.projectlombok:lombok:1.18.28'
    implementation 'org.apache.commons:commons-lang3:3.12.0'
    
    // Testing
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.batch:spring-batch-test'
    testImplementation 'com.h2database:h2:2.1.214'
    testImplementation "io.github.resilience4j:resilience4j-test:${resilience4jVersion}"
}

dependencyManagement {
    imports {
        mavenBom "org.springframework.cloud:spring-cloud-dependencies:2021.0.8"
    }
}

tasks.named('test') {
    useJUnitPlatform()
}

docker {
    name "${project.group}/${bootJar.archiveBaseName.get()}:${version}"
    files bootJar.archiveFile.get()
    buildArgs(['JAR_FILE': "${bootJar.archiveFileName.get()}"])
    tag 'latest'
}

task buildDockerImage(type: Exec) {
    dependsOn bootJar
    commandLine 'docker', 'build', '-t', "${project.group}/${bootJar.archiveBaseName.get()}:${version}", '.'
}



####################
{
  "name": "deleteOldSnapshotsWithReleasesByGroup",
  "type": "groovy",
  "content": "import org.sonatype.nexus.repository.storage.Query;\nimport org.sonatype.nexus.repository.storage.StorageFacet;\nimport org.sonatype.nexus.repository.Repository;\nimport org.sonatype.nexus.repository.storage.Component;\nimport java.time.Instant;\nimport java.time.temporal.ChronoUnit;\n\n// === Configuration ===\ndef repositoryName = \"my-snapshot-repo\"\ndef targetGroup = \"com.mycompany\"\ndef minAgeDays = 30  // Only delete if older than this\n\ndef repo = repository.repositoryManager.get(repositoryName)\nif (!repo) {\n    log.warn(\"Repository not found: ${repositoryName}\")\n    return\n}\n\ndef tx = repo.facet(StorageFacet).txSupplier().get()\ntx.begin()\ntry {\n    def cutoffDate = Instant.now().minus(minAgeDays, ChronoUnit.DAYS)\n    def allComponents = tx.findComponents(Query.builder().where(\"group = :group\").param(\"group\", targetGroup).build(), [repo])\n    def grouped = [:].withDefault { [:].withDefault { [] } }\n\n    allComponents.each { Component c ->\n        def key = c.group + \":\" + c.name\n        grouped[key][c.version] << c\n    }\n\n    grouped.each { key, versionsMap ->\n        versionsMap.each { version, components ->\n            if (version.endsWith(\"-SNAPSHOT\")) {\n                def releaseVersion = version.replace(\"-SNAPSHOT\", \"\")\n                if (versionsMap.containsKey(releaseVersion)) {\n                    components.each { Component snapshotComponent ->\n                        def createdDate = snapshotComponent.created?.toInstant()\n                        if (createdDate != null && createdDate.isBefore(cutoffDate)) {\n                            log.info(\"Deleting snapshot ${snapshotComponent.group}:${snapshotComponent.name}:${snapshotComponent.version} (created ${createdDate}) — release ${releaseVersion} exists and age > ${minAgeDays} days\")\n                            tx.deleteComponent(snapshotComponent)\n                        } else {\n                            log.info(\"Skipping ${snapshotComponent.group}:${snapshotComponent.name}:${snapshotComponent.version} — too recent or no creation date\")\n                        }\n                    }\n                }\n            }\n        }\n    }\n\n    tx.commit()\n} catch (Exception e) {\n    log.error(\"Error while deleting components: \", e)\n    tx.rollback()\n} finally {\n    tx.close()\n}"
}


################### Gradle script to print java version of dependencis ################

#!/bin/bash

echo "Dependency JARs and required Java versions:"
echo "---------------------------------------------"

while read jar; do
    if [[ -f "$jar" ]]; then
        version=$(jdeps -verbose "$jar" 2>/dev/null | grep "class file version" | head -n1 | awk '{print $NF}')
        if [[ -n "$version" ]]; then
            echo "$jar => class file version $version (Java $(get_java_version $version))"
        else
            echo "$jar => Unknown"
        fi
    fi
done < jars.txt

# Helper function: map class file version to Java version
get_java_version() {
    case $1 in
        52) echo "8" ;;
        53) echo "9" ;;
        54) echo "10" ;;
        55) echo "11" ;;
        56) echo "12" ;;
        57) echo "13" ;;
        58) echo "14" ;;
        59) echo "15" ;;
        60) echo "16" ;;
        61) echo "17" ;;
        62) echo "18" ;;
        63) echo "19" ;;
        64) echo "20" ;;
        65) echo "21" ;;
        *)  echo "Unknown" ;;
    esac
}

############################
