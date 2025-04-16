# docs
dependencies {
    // Spring Boot Starters
    implementation 'org.springframework.boot:spring-boot-starter-batch'
    implementation 'org.springframework.boot:spring-boot-starter-web' // For REST calls
    implementation 'org.springframework.boot:spring-boot-starter-data-jdbc'
    implementation 'org.springframework.boot:spring-boot-starter-actuator' // For monitoring
    implementation 'org.springframework.boot:spring-boot-starter-aop' // For Resilience4j
    
    // Spring Cloud (for SCDF)
    implementation 'org.springframework.cloud:spring-cloud-starter-task'
    
    // Database & Snowflake
    implementation 'net.snowflake:snowflake-jdbc:3.14.1'
    implementation 'com.zaxxer:HikariCP:5.0.1' // Connection pool
    
    // REST Client
    implementation 'org.springframework:spring-webflux' // For reactive WebClient
    implementation 'io.projectreactor.netty:reactor-netty' // WebClient transport
    
    // Resilience
    implementation 'io.github.resilience4j:resilience4j-spring-boot3:2.1.0'
    implementation 'io.github.resilience4j:resilience4j-reactor:2.1.0'
    
    // JSON Processing
    implementation 'com.fasterxml.jackson.core:jackson-databind'
    implementation 'com.fasterxml.jackson.datatype:jackson-datatype-jsr310'
    
    // Utilities
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    implementation 'org.apache.commons:commons-lang3:3.12.0'
    
    // Testing
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.batch:spring-batch-test'
    testImplementation 'com.h2database:h2' // For tests
    testImplementation 'io.github.resilience4j:resilience4j-test:2.1.0'
}
