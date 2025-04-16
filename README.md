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
