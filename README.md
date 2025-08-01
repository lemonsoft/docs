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

ou want to know the Java class file major version (e.g., 52 for Java 8) of each dependency.

To do this, you'll need to:

Extract the JAR files from Gradle’s dependency cache, and

Run javap -verbose or jdeps to inspect a .class file inside each JAR.

🔁 Recommended Alternative Workflow
✅ Step 1: Print All Dependency JARs (Gradle Task)
Add to build.gradle:

groovy
Copy
Edit
task printDependencyJars {
    doLast {
        configurations.runtimeClasspath.resolvedConfiguration.resolvedArtifacts.each {
            println it.file
        }
    }
}
Run it:

bash
Copy
Edit
./gradlew -q printDependencyJars > jars.txt
✅ Step 2: Extract and Print major version Using javap
Run this shell script to analyze the class file version (Java major version):

function Get-JavaVersionFromMajor {
    param([int]$major)

    switch ($major) {
        45 { return "Java 1.1" }
        46 { return "Java 1.2" }
        47 { return "Java 1.3" }
        48 { return "Java 1.4" }
        49 { return "Java 5" }
        50 { return "Java 6" }
        51 { return "Java 7" }
        52 { return "Java 8" }
        53 { return "Java 9" }
        54 { return "Java 10" }
        55 { return "Java 11" }
        56 { return "Java 12" }
        57 { return "Java 13" }
        58 { return "Java 14" }
        59 { return "Java 15" }
        60 { return "Java 16" }
        61 { return "Java 17" }
        62 { return "Java 18" }
        63 { return "Java 19" }
        64 { return "Java 20" }
        65 { return "Java 21" }
        Default { return "Unknown" }
    }
}

Get-Content "jars.txt" | ForEach-Object {
    $jarPath = $_
    if (Test-Path $jarPath) {
        $classEntry = & jar tf $jarPath | Where-Object { $_ -like "*.class" } | Select-Object -First 1
        if ($classEntry) {
            $javapOutput = & javap -verbose -cp $jarPath $classEntry 2>$null
            $majorVersion = ($javapOutput | Select-String "major version").ToString().Split()[-1]
            $javaVersion = Get-JavaVersionFromMajor -major [int]$majorVersion
            Write-Output "$jarPath => major version $majorVersion (Java $javaVersion)"
        } else {
            Write-Output "$jarPath => No .class file found"
        }
    } else {
        Write-Output "$jarPath => File not found"
    }
}



############################
package com.example.dialect;

import java.sql.Types;

import org.hibernate.dialect.Dialect;
import org.hibernate.type.StandardBasicTypes;
import org.hibernate.dialect.function.StandardSQLFunction;

public class SnowflakeVariantDialect extends Dialect {

    public SnowflakeVariantDialect() {
        super();

        // Register basic SQL types
        registerColumnType(Types.VARCHAR, "VARCHAR");
        registerColumnType(Types.INTEGER, "NUMBER(38,0)");
        registerColumnType(Types.BIGINT, "NUMBER(38,0)");
        registerColumnType(Types.BOOLEAN, "BOOLEAN");
        registerColumnType(Types.DOUBLE, "FLOAT");
        registerColumnType(Types.DATE, "DATE");
        registerColumnType(Types.TIMESTAMP, "TIMESTAMP");

        // Register VARIANT as custom SQL type
        registerColumnType(Types.JAVA_OBJECT, "VARIANT");

        // Optional: register PARSE_JSON as SQL function if needed
        registerFunction("parse_json", new StandardSQLFunction("PARSE_JSON", StandardBasicTypes.STRING));
    }

    @Override
    public boolean supportsIdentityColumns() {
        return true;
    }

    @Override
    public String getIdentityColumnString() {
        return "AUTOINCREMENT";
    }

    @Override
    public String getIdentitySelectString() {
        return "SELECT LAST_INSERT_ID()";
    }

    @Override
    public boolean supportsSequences() {
        return false;
    }

    @Override
    public boolean supportsLimit() {
        return true;
    }

    @Override
    public String getLimitString(String sql, boolean hasOffset) {
        return sql + (hasOffset ? " LIMIT ? OFFSET ?" : " LIMIT ?");
    }
}

**********************************
package com.example.dialect;

import java.sql.Types;

import org.hibernate.dialect.Dialect;
import org.hibernate.dialect.identity.IdentityColumnSupport;
import org.hibernate.dialect.identity.IdentityColumnSupportImpl;
import org.hibernate.engine.jdbc.env.spi.NameQualifierSupport;
import org.hibernate.type.StandardBasicTypes;

public class SnowflakeCustomDialect extends Dialect {

    public SnowflakeCustomDialect() {
        super();

        // Register SQL types
        registerColumnType(Types.VARCHAR, "VARCHAR");
        registerColumnType(Types.BIGINT, "NUMBER(38,0)");
        registerColumnType(Types.INTEGER, "NUMBER(10,0)");
        registerColumnType(Types.BOOLEAN, "BOOLEAN");
        registerColumnType(Types.DOUBLE, "FLOAT");
        registerColumnType(Types.DATE, "DATE");
        registerColumnType(Types.TIMESTAMP, "TIMESTAMP");
        registerColumnType(Types.JAVA_OBJECT, "VARIANT");

        // Optional: register Snowflake functions
        registerFunction("parse_json", new org.hibernate.dialect.function.StandardSQLFunction("PARSE_JSON", StandardBasicTypes.STRING));
    }

    // ✅ Identity column support
    @Override
    public IdentityColumnSupport getIdentityColumnSupport() {
        return new SnowflakeIdentityColumnSupport();
    }

    // Optional: tell Hibernate we don't support sequences
    @Override
    public boolean supportsSequences() {
        return false;
    }

    @Override
    public boolean supportsIdentityColumns() {
        return true;
    }

    @Override
    public String getIdentityColumnString() {
        return "AUTOINCREMENT"; // or "IDENTITY"
    }

    @Override
    public String getIdentitySelectString() {
        return "SELECT LAST_INSERT_ID()"; // Not actually supported; Snowflake handles this automatically
    }

    // Optional: help Hibernate with object qualification
    @Override
    public NameQualifierSupport getNameQualifierSupport() {
        return NameQualifierSupport.BOTH;
    }

    @Override
    public boolean supportsInsertSelectIdentity() {
        return true;
    }

    @Override
    public String appendIdentitySelectToInsert(String insertSQL) {
        return insertSQL; // Snowflake returns the ID without special handling
    }

    @Override
    public boolean supportsJdbcConnectionLobCreation() {
        return false;
    }
}
*********************************

package com.example.dialect;

import org.hibernate.dialect.identity.IdentityColumnSupportImpl;

public class SnowflakeIdentityColumnSupport extends IdentityColumnSupportImpl {

    @Override
    public boolean supportsIdentityColumns() {
        return true;
    }

    @Override
    public String getIdentitySelectString(String table, String column, int type) {
        // Not used by Snowflake directly; return dummy
        return "SELECT LAST_INSERT_ID()";
    }

    @Override
    public String getIdentityColumnString(int type) {
        return "AUTOINCREMENT"; // or "IDENTITY"
    }

    @Override
    public boolean hasDataTypeInIdentityColumn() {
        return false;
    }
}

****************
public class SnowflakeCustomDialect extends org.hibernate.dialect.Dialect {

    public SnowflakeCustomDialect() {
        super();

        registerColumnType(Types.BIGINT, "NUMBER(38,0)");
        registerColumnType(Types.VARCHAR, "VARCHAR");
        registerColumnType(Types.BOOLEAN, "BOOLEAN");

        // Important
        registerColumnType(Types.JAVA_OBJECT, "VARIANT");
    }

    @Override
    public IdentityColumnSupport getIdentityColumnSupport() {
        return new IdentityColumnSupportImpl() {
            @Override
            public boolean supportsIdentityColumns() {
                return true;
            }

            @Override
            public String getIdentitySelectString(String table, String column, int type) {
                return null; // Snowflake handles identity transparently
            }

            @Override
            public String getIdentityColumnString(int type) {
                return "AUTOINCREMENT";
            }
        };
    }

    @Override
    public boolean supportsInsertSelectIdentity() {
        return true;
    }

    @Override
    public String appendIdentitySelectToInsert(String insertSQL) {
        return insertSQL; // Hibernate assumes the database will return the ID
    }

    @Override
    public boolean supportsIdentityColumns() {
        return true;
    }

    @Override
    public boolean supportsSequences() {
        return false;
    }
}
***********
You will manually fetch the next value using SELECT MAX(id) + 1 (or a sequence, if you use one).

Step 1: Define a Custom Generator
java
Copy
Edit
package com.example.snowflake;

import org.hibernate.id.IdentifierGenerator;
import org.hibernate.engine.spi.SharedSessionContractImplementor;

import java.io.Serializable;
import java.sql.ResultSet;
import java.sql.Statement;

public class SnowflakeIdentityGenerator implements IdentifierGenerator {
    @Override
    public Serializable generate(SharedSessionContractImplementor session, Object object) {
        try (Statement stmt = session.connection().createStatement()) {
            ResultSet rs = stmt.executeQuery("SELECT MAX(ID) + 1 FROM EMPLOYEE");
            if (rs.next()) {
                return rs.getLong(1);
            }
        } catch (Exception e) {
            throw new RuntimeException("Failed to generate identity value", e);
        }
        throw new RuntimeException("No ID generated");
    }
}
This mimics an identity behavior. Replace MAX(ID) + 1 with a call to a sequence or custom logic if needed.

Step 2: Use the Generator in Your Entity
java
Copy
Edit
@Entity
@Table(name = "EMPLOYEE")
public class Employee {

    @Id
    @GeneratedValue(generator = "snowflake-id")
    @GenericGenerator(name = "snowflake-id", strategy = "com.example.snowflake.SnowflakeIdentityGenerator")
    @Column(name = "ID")
    private Long id;

    @Column(name = "EMP_NAME")
    private String name;

    @Column(name = "EMAIL_ADDRESS")
    private String email;

    // Constructors, getters, setters
}
******
