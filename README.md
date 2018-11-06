# Liquibase

### This documentation was created from the liquibase source documentation at: https://www.liquibase.org/documentation/

## Purpose of this document:
The purpose of this document is to create useful, organized documentation on usage and behavior of liquibase. It is also to document some general best practices observed by actual users of Liquibase on large projects to produce an all inclusive reference for using Liquibase.

## Why liquibase?
Liquibase operates as an open-source library with tooling to easily orchestrate database structure changes and data migrations in a trackable, consistent and repeatable way.  This allows developers to easily make and deliver database updates as well as migrate existing data to all environments.  Liquibase is database agnostic which allows changes to be migrated across different database types, IE: Oracle, Mysql, H2.

## <a name="databasesupport"></a>Database Support
Liquibase supports the following databases:
* MySQL
* PostgreSQL
* Oracle
* Sql Server
* Sybase_Enterprise
* DB2
* Apache Derby
* HSQL
* H2
* Informix
* Firebird
* SQLite

## How it works
Liquibase uses a databasechangelog file to orchestrate which sql should be run in which order, so it runs in a consistent and repeatable state across all environments, from local environments on personal machines up to production servers.  It also removes DBA micromanagement of development changes pertaining to schema structure and data migrations.

A databasechangelog file consists of a master.xml file which can point to individual changeset files or directories(IE: sprint176) of many changeset files.  The folders and files are executed in the order specified by the master.xml.  However, when a folder is specified, the files in that folder run alpha-numberically based on their file names and each changeset in the file runs based on it's position in the changeset file.

Example master.xml file:

```xml
    <?xml version="1.0" encoding="UTF-8"?>

    <databaseChangeLog
            xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xmlns:ext="http://www.liquibase.org/xml/ns/dbchangelog-ext"
            xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.1.xsd
            http://www.liquibase.org/xml/ns/dbchangelog-ext http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-ext.xsd">

        <include="someLiquibaseFile" relativeToChangelogFile="true"/>
        <includeAll="sprint111" relativeToChangeLogFile="true"/>

    </databaseChangeLog>
```

Each changeset file should have a consistent naming pattern to ensure it runs in the intended order, reflects when it was created, and the intent.

*IE: 201811051613_111_creating_person_table.xml (YYYYMMDDHHMM_STORYNUMBER_SOME_DESCRIPTION.xml)*

Example changeset file:

```xml
    <?xml version="1.0" encoding="UTF-8"?>

    <databaseChangeLog
            xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xmlns:ext="http://www.liquibase.org/xml/ns/dbchangelog-ext"
            xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.1.xsd
            http://www.liquibase.org/xml/ns/dbchangelog-ext http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-ext.xsd">

        <preConditions>
            <runningAs username="liquibase"/>
        </preConditions>

        <changeSet id="201811051613_111_creating_person_table" author="efoster">
            <createTable tableName="person">
                <column name="id" type="int" autoIncrement="true">
                    <constraints primaryKey="true" nullable="false"/>
                </column>
                <column name="firstname" type="varchar(50)"/>
                <column name="lastname" type="varchar(50)">
                    <constraints nullable="false"/>
                </column>
                <column name="state" type="char(2)"/>
            </createTable>
        </changeSet>

        <changeSet id="201811051613_111_add_username_to_person_table" author="efoster">
            <addColumn tableName="person">
                <column name="username" type="varchar(8)"/>
            </addColumn>
        </changeSet>
        <changeSet id="201811051613_111_add_state_lookup_table_to_person" author="efoster">
            <addLookupTable
                existingTableName="person" existingColumnName="state"
                newTableName="state" newColumnName="id" newColumnDataType="char(2)"/>
        </changeSet>

    </databaseChangeLog>
```

Each set of changes to the database is generally in a separate changeset so changes are organized by their purpose and intent, generally one file or more per story.  Each changeset should have a unique ID and all should follow consistent formats to ensure they always run in the intended order. 

*IE: 20181105_1601_687_Adding_ID_To_Table (YYYYMMDD_HHMM_STORYNUMBER_SOME_DESCRIPTION)*

### XML to SQL Conversion
Liquibase is written in an XML format and is the preferred way to write queries when possible.  This suffices for most queries, but for more advanced queries or even PL/SQL raw SQL can be used.  Liquibase will convert all XML into actual raw SQL which will be executed against the database.

#### Why to use liquibase xml schema tags
* simple operations have default rollback operations when using xml tags, meaning a rollback operation will perform as expected if manually invoked
* XML tags provide a schema validation exposing many required fields for a query
* XML tags are generally cleaner and more consistent
* XML tags are database agnostic
* When a simple query run using an xml tag fails, it will rollback and leave the database in a known state

### Transaction Commits and Rollbacks
Liquibase is setup to run each changeset in a transaction so it can rollback to a safe state in the event of a failure.

XML schema tags are preferred due to their ability to easily rollback and automatically create rollback statements for manual rollback invocation.  However this is also to prevent failures which put the database in an indeterminant state.

Every changeset may also have a rollback query supplied which can be used to provide a custom rollback behavior to ensure failures and manual rollback invocations work as intended

IE:  If you place 5 statements in a single raw SQL block, and one fails in between, liquibase will mark all 5 as failed but might not rollback properly.  This means during the next run liquibase will attempt to run all 5 commands, even though 1 or more might have already ran and committed.

The above scenario is the exact illustration of why raw SQL should generally be avoided, however in any event if a correct rollback was provided, it helps prevent leaving the database in a flawed state after a failure of that changeset.

```xml
   <changeSet id="changeRollback" author="efoster">
        <createTable tableName="changeRollback1">
            <column name="id" type="int"/>
        </createTable>
        <rollback>
            <dropTable tableName="changeRollback1"/>
        </rollback>
    </changeSet>
```

### Tracking and Checksums
Liquibase creates a checksum of each file.  This is used for tracking and ensuring files are not changed after a run has occurred.  When a changeset is executed, liquibase stores the changeset id, the file name/path, the checksum, changeset author and other auditing information into a `DATABASECHANGELOG` table.  This is used to determine if a changeset has already run.  On subsequent runs, as long as a change set does not have an attribute `alwaysRun="true" `, then liquibase will not rerun it.  This is how liquibase maintains a ledger of which changes have been run and need to be run.

The unique Checksum prevents tampering with liquibase files after they have been run and therefore it is essential that a liquibase changeset file NEVER be modified after being run in any managed or customer environment.  Modifying a file locally prior to delivery is fine, but in order to rerun a file, the changelog entry must be deleted to ensure the file reruns.

Additionally, a `CHANGELOGLOCK` table is used to ensure multiple liquibase runs don't occurr concurrently.

### Database Agnostic Behavior
Liquibase is designed to be database agnostic, which is exactly why the preferred format is to use XML schema tags.  

Liquibase will convert the XML into raw SQL for the targetted database type.  The default XML tags in the schema facilitate this as well as using a `property`.  A property acts as an alias and can be used for a common operation that is database technology specific. (IE: Dates, MySQL and Oracle use different function calls for this)

Example and how it's used:
```xml
    <property name="now" value="sysdate" dbms="oracle"/>
    <property name="now" value="now()" dbms="mysql"/>
    <property name="now" value="now()" dbms="postgresql"/>

    <column name="Join_date" defaultValueFunction="${now}"/>
```

### Advanced Features
* Preconditions - allow you to conditionally run the changeset based on the current database state:
```xml
    <!-- as a precondition to the entire changeset/log file -->
    <preConditions>
            <dbms type="oracle" />
            <runningAs username="SYSTEM" />
    </preConditions>

    <!-- as a precondition to the individual changeset -->
    <changeSet id="1" author="efoster">
        <preConditions onFail="WARN">
            <sqlCheck expectedResult="0">select count(*) from oldtable</sqlCheck>
        </preConditions>
        <comment>Comments should go after preCondition. If they are before then liquibase usually gives error.</comment>
        <dropTable tableName="oldtable"/>
    </changeSet>
```

* DBMS - allows you to target a changeset to a specific database type, for supported types refer to [Database Support](#databasesupport)

```xml
    <changeSet id="2" dbms="oracle" author="efoster">
```

* Context - allow you to set a context in which the changeset should run, for instance they can reflect environments, so you could target a changeset to only run locally or in a managed environment

```xml
    <changeSet id="2" author="bob" context="test">
    <changeSet id="2" author="bob" context="!test">
    <changeSet id="2" author="bob" context="test or dev">
    <changeSet id="2" author="bob" context="!test and !dev">
    <!-- test, dev, prod is the same as test or dev or prod -->
    <changeSet id="2" author="bob" context="test, dev, prod">
```
* Labels - allow you to setup complex logic for which changesets should be run during runtime
```xml
```
* Properties - already mentioned above in *Database Agnostic Behavior*

### Labels vs Contex
Labels and Contex tags seem very similar, however they have different usecases and behave differently at runtime than one might expect.

* Labels
* Contexts
### Liquibase Tagging
Liquibase allows tagging of the database so identify a particular state of the database

```shell
mvn liquibase:rollback -Dliquibase.rollbackTag=1.0
```
### Liquibase Rollbacks
The database can be manually rolled back to a point in time if desired using liquibase.  Specifically, rolled back to a certain changeset

```shell
    # Roll back to a specific tag
    mvn liquibase:rollback <tag>
    # Roll back to a specific date / time
    mvn liquibase:rollbackToDate <date/time>
    # Roll back a certain number of change sets
    mvn liquibase:rollbackCount <value> 
```

### Liquibase Generating Change Logs (Exporting)
```shell
    liquibase --driver=oracle.jdbc.OracleDriver \
        --classpath=\path\to\classes:jdbcdriver.jar \
        --changeLogFile=com/example/db.changelog.xml \
        --url="jdbc:oracle:thin:@localhost:1521:XE" \
        --username=scott \
        --password=tiger \
        generateChangeLog
```

### Liquibase through Maven
Liquibase has a maven plugin which allows liquibase commands to be executed through maven goals and commands.

To get started add the following to your pom.xml and update the information to reflect your configuration
```xml
 <plugin>
      <groupId>org.liquibase</groupId>
      <artifactId>liquibase-maven-plugin</artifactId>
      <version>3.0.5</version>
      <configuration>
        <changeLogFile>src/main/resources/org/liquibase/business_table.xml</changeLogFile>
          <driver>oracle.jdbc.driver.OracleDriver</driver>
          <url>jdbc:oracle:thin:@tf-appserv-linux:1521:xe</url>
          <username>liquibaseTest</username>
          <password>pass</password>
        </configuration>
      <executions>
        <execution>
          <phase>process-resources</phase>
          <goals>
            <goal>update</goal>
          </goals>
        </execution>
      </executions>
    </plugin>
```

Example usage to execute liquibase:
```shell
    mvn liquibase:update
```

The above command can be used to deploy database changes at runtime without any need to restart the application as long as the changes aren't application breaking changes, IE: remove a table referenced or used by Hibernate.

### Liquibase through Spring
A liquibase spring bean can also be used to automatically run liquibase at application start thus automating the entire process once an artifact is built and deployed.  This is great for migration to managed environments since it requires no extra steps during the build or deploy.  The liquibase spring bean is also auto configured by spring-boot and will run as long as the bean is defined in the spring project.

To get started simply add liquibase to your dependencies in your pom.xml:
```xml
    <dependency>
        <groupId>org.liquibase</groupId>
        <artifactId>liquibase-core</artifactId>
        <version><!-- desired version here --></version>
    </dependency>
```

Then define a spring bean to invoke:
```java
    @Bean
    public SpringLiquibase liquibase() {
        SpringLiquibase liquibase = new SpringLiquibase();
        liquibase.setChangeLog("classpath:liquibase-changeLog.xml")
        // dataSource() would be the spring boot configured data source
        liquibase.setDataSource(dataSource());
        return liquibase;
    }
```

### Using PLSQL and Raw SQL
Liquibase supports using raw sql and even plsql.  It is general god practice to avoid a raw SQL statement unless explicitly required to accomplish the task, and in that event it is critical to supply the rollback block mentioned in the *Transaction Commits and Rollbacks* section.

### ChangeSets
Every changeset should have a unique id.  A great starting format is `YYYYMMDDHHMM_STORY#_DESCRIPTION_OF_THE_CHANGE`
* YYYYMMDDHHMM - ball park date and military time the change was delivered
* STORY# - general references a story or workitem tied to the change
* DESCRIPTION - quick, clear, concise description of the change

Every changeset should also contain an author tag, generally with the first initial and last name of the author delivering the change.
```shell
    <changeSet author="efoster" id="201811061514_1_creating_bonus_table">
        <createTable name="BONUS">
            <column name="ENAME" type="VARCHAR2(10,0)"/>
            <column name="JOB" type="VARCHAR2(9,0)"/>
            <column name="SAL" type="NUMBER(22,0)"/>
            <column name="COMM" type="NUMBER(22,0)"/>
        </createTable>
    </changeSet>
```
