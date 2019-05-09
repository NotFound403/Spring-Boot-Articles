---
title: Spring Boot 2实践系列(三十一)：Spring Batch 批处理框架详解和集成
comments: true
date: 2018-09-06 11:18:29
tags: [spring batch,批处理]
categories: [Spring Boot 2实践系列]
---
　　Spring Batch 是一款轻量级，全面，用来处理大量数据操作的批处理框架，旨帮助企业开发重要的批处理应用。从数据库、文件或队列中读取大量数据，按要求进行处理转换后输出指定形式的数据。

　　Spring Batch 提供了可重复使用的功能，这些功能对于处理大量记录至关重要，包括记录/跟踪，事务管理，作业(**job**)统计，作业(**job**)重启，跳过和资源管理等。 它还提供更高级优化和分区技术用于实现极高容量和高性能的批处理作业。 作业的运行的实例状态、执行数据和参数可以配置持久化到数据库，可以随时监听作用的执行状态。

　　Spring Batch 不是一个调度框架(如 Quartz)，而是与调度应用结合使用，不是替代关系。Spring Batch 自动执行基本批处理的迭代，提供处理类似事务的功能，通常在脱机环境中处理，无需任何用户交互。

　　[Spring Batch 官网](https://spring.io/projects/spring-batch),[Spring Batch v4.0.1 Releas 参考文档](https://docs.spring.io/spring-batch/4.0.x/reference/html/index.html)
<!-- more -->
## Spring Batch ##
### 应用场景 ###
Spring Batch 适用于需要处理大量数据的业务，如大批量数据转换输出、对账任务、数据迁移、数据统计分析、发送批量通知或指令等；定期执行批处理任务，整个处理过程可以自动化和周期化运行。

### 组成部分 ###
Spring Batch 主要由以下几部分组成：

|名称               |用途                                  |
|:-----------------|:------------------------------------|
|JobRepository     |用于持久化 job 运行时的元数据|
|JobLanucher       |用于启动Job的接口|
|Job               |实际要执行的任务，包含一个或多个 Step|
|Step              |Step-步骤包含 ItemReader、ItemProcessor 和 ItemWriter|
|ItemReader        |用于读取数据的接口|
|ItemProcessor     |用于处理数据的接口|
|ItemWriter        |用于输出数据的接口|

[Spring Batch 4 提供了一组实现 ItemReader 和 ItemWriter 的构建器, 参考官网](https://docs.spring.io/spring-batch/4.0.x/reference/html/whatsnew.html#whatsNew)，支持从多种数据源中读取数据，可以以多种方式输出数据，[支持的 ItemReader 和 ItemWriter 直接点击参考官网](https://docs.spring.io/spring-batch/4.0.x/reference/html/appendix.html#listOfReadersAndWriters)。

### 核心概念 ###
![Spring Batch 领域模型](http://112.74.59.39:90/images/1537255614349.png)
上图显示了构成Spring Batch 的关键概念。 Job有一个到多个步骤，每个步骤只有一个 ItemReader，一个 ItemProcessor 和一个 ItemWriter 。 需要启动作业（使用JobLauncher），并且需要存储有关当前正在运行的进程的元数据（在JobRepository中）。

Spring Batch 批处理流程可以分3个基础阶段：分别是**获取源数据、数据处理、数据输出**。
在使用时，先创建 Spring Batch 的配置类，在配置类上添加`@EnableBatchProcessing`注解开启批处理的支持，在 Java 配置文件中 注入 Spring Batch 组成部分的 Bean。
1. **ItemReader**
读取源数据，ItemReader 有多个实现，可以从**数据库、MQ、文件、流**读取数据。
2. **ItemProcessor**
对源数据进行业务处理，实现 ItemProcessor 接口，重写 process 方法，方法输入的参数是从 ItemReader 读取到的数据，返回的数据给 ItemWriter，如果业务处理无效时返回 null，表示不输出该数据(项目)。如对数据进行转换、抽取、校对、统计运算等。
3. **ItemWriter**
输出数据到目标地方，可以输出存储到**数据库、文件**，还可以以**流、消息**的方式输出。
4. **Step**
创建作业的执行步骤，需要注入 ItemReader、ItemProcessor、ItemWriter。 Step 是个领域对象，封装了批处理作业的执行步骤。
5. **Job**
封装整个批处理过程的实体，是 Step 的容器。在配置中需要注入 Step。 若需要监听 Job 的执行情况，则定义一个实现 JobExecutionListener 接口的监听器类或继承 JobExecutionListenerSupport，实现里面的方法，在 Job 里注入监听器。
Spring Batch 以 SimpleJob 类的形式提供了 Job 接口的默认简单实现，它在Job之上创建了一些标准功能。
6. **JobLanucher**
Job 调度器，从 JobRepository 获取有交换的 JobExecution 并执行 Job 。若是要在定时任务中执行 Job，只需在定时任务中执行 JobLauncher 的 run 方法。
7. **JobRepository**
创建 Job 容器，给 Job 运行时的实例数据提供持久化操作。它为JobLauncher，Job 和 Step 实现提供 CRUD 操作。 首次启动 Job 时，将从存储库中获取 JobExecution，并且在执行过程中，StepExecution和 JobExecution 实现将通过它来持久化的存储库，支持内存存储和数据库存储，配置存储到数据库可以随时监控批处理的执行状态。
使用java配置时，`@EnableBatchProcessing` 注解默认提供了 Spring Batch的简单配置，将 JobRepository 作为自动配置的组件之一提供。

还可以在 JobParameters 中绑定参数，在 Bean 定义的时候使用一个特殊的 Bean 生命周期注解 @StepScope 然后通过 @Value 注入此参数。

[Spring Batch 术语 -> The Domain Language of Batch 官方文档](https://docs.spring.io/spring-batch/4.0.x/reference/html/domain.html#step)

### 元数据 ###
Spring Batch 为多种数据库提供了创建元数据表的 SQL 脚本，元数据表用于存储 Job 实例运行时的状态、参数等数据，Spring Batch 会根据数据源的类型去执行对应的元数据表SQL脚本。Job 启动执行首先会通过 JobRepository 从数据库中获取 Job 和 Step, 如果元数据表不存在则会报错。

[元数据结构 -> Meta-Data Schema 官方文档](https://docs.spring.io/spring-batch/4.0.x/reference/html/schema-appendix.html#metaDataSchema)

Spring Batch 支持多种数据库的元数据表SQL脚本在 org.springframework.batch.core 包路径下。以 MySQL 为例(schema-mysql.sql)，会创建 9 个表，其中核心业务表有六张：
1. **BATCH_JOB_INSTANCE**
存放 job 实例数据，包含 JOB_INSTANCE_ID, VERSION, JOB_NAME, JOB_KEY 数据。
2. **BATCH_JOB_EXECUTION**
存放 job 执行情况数据, 包括id、版本、开始时间、结束时间、状态等。每次执行 Job , 总会有一个新的 JobExecution, 该表中就会新增一行。
3. **BATCH_JOB_EXECUTION_PARAMS**
存放传递给 job 的键/值对参数,值如果是多种类型会被存放到类型对应的字段中。
4. **BATCH_JOB_EXECUTION_CONTEXT**
每个JobExecution只有一个Job ExecutionContext，它包含特定作业执行所需的所有作业级数据。 此数据通常表示失败后必须检索的状态，以便JobInstance可以“从停止的位置开始”。
5. **BATCH_STEP_EXECUTION**
每一个 Step 至少为每创建一个 JobExecution 新增一条数据。包含step名称、开始时间、结束时间、状态、提交次数、读次数、写次数、读忽略次数、写忽略次数、回滚次数等数据。
6. **BATCH_STEP_EXECUTION_CONTEXT**
每个StepExecution只有一个ExecutionContext，它包含特定步骤执行需要持久化的所有数据。 此数据通常表示失败后必须检索的状态，以便JobInstance可以“从停止的位置开始”。
7. **BATCH_JOB_EXECUTION_SEQ**
8. **BATCH_JOB_SEQ**
9. **BATCH_STEP_EXECUTION_SEQ**
以上SEQ三个表是序列表，因有的数据库不支持主键递增来作为唯一标识，所以使用单独的表来记录每个序列(唯一标识)。

## Spring Boot 支持 ##
Spring Boot 为 Spring Batch 提供了自动配置的支持，通过添加` @EnableBatchProcessing`注解来启动自动配置，自动配置的源码位于 org.springframework.boot.autoconfigure.batch 包下。

自动配置为 Spring Batch 配置了数据源、提供了数据源初始化、支持随应用启动执行Job(需配置)、默认提供了简单的 Job 操作(SimpleJobOperator); 在应用启动时就初始化创建了 transactionManager、jobRepository、jobLauncher、jobExplorer。所以Spring Boot 集成 Spring Batch，在 Java 配置文件中 transactionManager、jobRepository、jobLauncher 的配置可以省略。

Spring Boot 自动配置提供了基于 JPA 的自动初始化 Spring Batch 存储批处理记录的数据库，当程序启动时，会自动执行定义的 Job 的 Bean 。

Spring Boot 2.0.5.RELEASE 为 Spring Batch 提供了如下属性配置：
```
# SPRING BATCH (BatchProperties)
spring.batch.initialize-schema=embedded # Database schema initialization mode.
spring.batch.job.enabled=true # Execute all Spring Batch jobs in the context on startup.
spring.batch.job.names= # Comma-separated list of job names to execute on startup (for instance, `job1,job2`). By default, all Jobs found in the context are executed.
spring.batch.schema=classpath:org/springframework/batch/core/schema-@@platform@@.sql # Path to the SQL file to use to initialize the database schema.
spring.batch.table-prefix= # Table prefix for all the batch meta-data tables.
```

## Spring Boot 集成 ##
[官方示例](http://spring.io/guides/gs/batch-processing/)，将 CSV 文件中的数据 使用 JDBC 批处理的方式导入数据库。
1. 添加依赖
``` xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-batch</artifactId>
</dependency>
```
2. 创建封装源数据的实体类和导出数据(示例是同一个)
``` java
public class Person {

    private String lastName;
    private String firstName;

    public Person() {
    }

    public Person(String lastName, String firstName) {
        this.lastName = lastName;
        this.firstName = firstName;
    }

	//-----set/get方法------
}
```

3. 创建 Spring Batch 配置
``` java
@Configuration
@EnableBatchProcessing
public class BatchConfig {

    @Autowired
    private JobBuilderFactory jobBuilderFactory;

    @Autowired
    private StepBuilderFactory stepBuilderFactory;

    /**
     * @desc: 从文件中读取源数据
     * @param: []
     * @return: org.springframework.batch.item.file.FlatFileItemReader<com.springboot.springbatch.entity.Person>
     **/
    @Bean
    public FlatFileItemReader<Person> reader() {
        // 读取文件
        FlatFileItemReader<Person> itemReader = new FlatFileItemReader<>();
        // 设置文件路径
        itemReader.setResource(new ClassPathResource("person-data.csv"));

        // 数据和领域模型类做对应映射
        DefaultLineMapper<Person> lineMapper = new DefaultLineMapper<>();
        DelimitedLineTokenizer lineTokenizer = new DelimitedLineTokenizer();
        BeanWrapperFieldSetMapper<Person> fieldSetMapper = new BeanWrapperFieldSetMapper<>();

        fieldSetMapper.setTargetType(Person.class);
        lineTokenizer.setNames(new String[]{"firstName", "lastName"});
        lineMapper.setLineTokenizer(lineTokenizer);
        lineMapper.setFieldSetMapper(fieldSetMapper);
        itemReader.setLineMapper(lineMapper);

        return itemReader;
    }

    /**
     * @desc: 注入数据处理器
     * @author: gxing
     * @date: 2018/9/17 10:57
     * @param: []
     * @return: com.springboot.springbatch.job.PersonItemProcessor
     **/
    @Bean
    public PersonItemProcessor processor() {
        return new PersonItemProcessor();
    }

    /**
     * @desc: 输出数据,自动配置了DataSource,以参数方式注入
     * @param: [dataSource]
     * @return: org.springframework.batch.item.database.JdbcBatchItemWriter<com.springboot.springbatch.entity.Person>
     **/
    @Bean
    public JdbcBatchItemWriter<Person> writer(DataSource dataSource) {
        // 写入到数据库
        JdbcBatchItemWriter<Person> itemWriter = new JdbcBatchItemWriterBuilder<Person>()
                .itemSqlParameterSourceProvider(new BeanPropertyItemSqlParameterSourceProvider<Person>())
                .sql("INSERT INTO person (first_name, last_name) VALUES (:firstName, :lastName)")
                .dataSource(dataSource)
                .build();
        return itemWriter;
    }

    /**
     * @desc: 定义作业。作业是根据步骤构建的。
     * @param: [listener, step1]
     * @return: org.springframework.batch.core.Job
     **/
    @Bean
    public Job importUserJob(JobCompletionNotificationListener listener, Step step1) {
        return jobBuilderFactory.get("importUserJob")
                .incrementer(new RunIdIncrementer())
                .listener(listener)
                .flow(step1)
                .end()
                .build();
    }

    /**
     * @desc: 定义单个Step步骤, 每个步骤涉及读者，处理器和编写者
     * @param: [itemWriter]
     * @return: org.springframework.batch.core.Step
     **/
    @Bean
    public Step step1(JdbcBatchItemWriter<Person> itemWriter) {

        //定义一次写入数据量,此处是10条
        //chunk()前辍<Person, Person>表示输入和输出类型,并与ItemReader <Person>和ItemWriter <Person>对齐
        //在使用之前注入 ItemReader、ItemProcessor 和 ItemWriter
        return stepBuilderFactory.get("step1").<Person, Person>chunk(10000)
                .reader(reader())
                .processor(processor())
                .writer(itemWriter)
                .build();
    }
}
```
4. 创建 Job 监听器
``` java
/**
 * @name: JobCompletionNotificationListener
 * @desc: 完成作业通知监听器
 * 继承JobExecutionListenerSupport,或实现 JobExecutionListener接口
 **/
@Component
public class JobCompletionNotificationListener extends JobExecutionListenerSupport {

    private static final Logger logger = LogManager.getLogger(JobCompletionNotificationListener.class);

    private final JdbcTemplate jdbcTemplate;

    public JobCompletionNotificationListener(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    @Override
    public void afterJob(JobExecution jobExecution) {
        if(jobExecution.getStatus() == BatchStatus.COMPLETED){
            logger.info("!!! JOB FINISHED! Time to verify the results");
            Integer count = jdbcTemplate.queryForObject("select count(*) from person", Integer.class);
            logger.info("!!! Found Person Number:{}",count);
        }


    }

    @Override
    public void beforeJob(JobExecution jobExecution) {
        logger.info("Job Ready OK.......");
    }
}
```
5. 创建数据处理器
``` java
/**
 * @Name: PersonItemProcessor
 * @Desc: 批处理中的一个常见范例是获取数据，对其进行转换，然后将其传输到其他位置。
 * 在这里编写一个简单的转换器，将名称转换为大写。
 **/
public class PersonItemProcessor implements ItemProcessor<Person, Person> {

    private static final Logger logger = LogManager.getLogger(PersonItemProcessor.class);

    /**
     * @desc: 不要求输入和输出类型相同。
     * 实际上，在读取一个数据源之后，有时应用程序的数据流需要不同的数据类型。
     * @param: [person]
     * @return: com.springboot.springbatch.entity.Person
     **/
    @Override
    public Person process(Person person) throws Exception {

        final String firstName = person.getFirstName().toUpperCase();
        final String lastName = person.getLastName().toUpperCase();
        final Person transformedPerson = new Person()
                .setFirstName(firstName).setLastName(lastName);
        return transformedPerson;
    }
}
```
6. application.properties 配置
```
#=============jdbc dataSource=========================
spring.datasource.name=druidDataSource
spring.datasource.type=com.alibaba.druid.pool.DruidDataSource
#spring.datasource.url=jdbc:mysql://localhost:3306/sakila?characterEncoding=utf-8&allowMultiQueries=true&autoReconnect=true
spring.datasource.url=jdbc:log4jdbc:mysql://localhost:3306/sakila?characterEncoding=utf-8&allowMultiQueries=true&autoReconnect=true
spring.datasource.username=panda
spring.datasource.password=123456
#spring.datasource.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.driver-class-name=net.sf.log4jdbc.sql.jdbcapi.DriverSpy
spring.datasource.druid.initial-size=5
spring.datasource.druid.max-active=20
spring.datasource.druid.min-idle=5
spring.datasource.druid.max-wait=10
spring.datasource.druid.validationQuery=SELECT 1
#初始化表结构(执行DDL SQL语句)
spring.datasource.initialization-mode=always
# 如果没有配置初始化的schema,则数据源在初始化时会从类路径下加载执行 schema-all.sql 和 schema.sql 文件
#spring.datasource.schema=classpath:person-table.sql
spring.datasource.sql-script-encoding=UTF-8

# SPRING BATCH (BatchProperties)
#初始化Spring Batch 元数据表
spring.batch.initialize-schema=always
#是否随应用启动执行job
spring.batch.job.enabled=true
#spring.batch.job.names= # Comma-separated list of job names to execute on startup (for instance, `job1,job2`). By default, all Jobs found in the context are executed.
#spring.batch.schema=classpath:org/springframework/batch/core/schema-@@platform@@.sql
#spring.batch.table-prefix=
```
7. CSV数据源,可以连续累积拷贝数据快速创建大量数据
```
Jill,Doe
Joe,Doe
Justin,Doe
Jane,Doe
John,Doe
Trump,Donald
Obama,Barack
```
6. 实体类对应的表的SQL
``` sql
DROP TABLE IF EXISTS person;

CREATE TABLE `person` (
  `id` BIGINT(11) NOT NULL AUTO_INCREMENT,
  `first_name` VARCHAR(20) DEFAULT NULL,
  `last_name` VARCHAR(20) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=INNODB DEFAULT CHARSET=latin1;


-- Spring Batch 启动必须初始化的元数据(记录job和step),默认是嵌入的内部数据库,
-- 当使用mysql时,初始化无数据必须配置 spring.batch.initialize-schema=always ,
-- 当 Spring Batch 提供的元数据的初始化 SQL 没有判断表是否存在,当应用重启时会报表已存在的错误,以下增加表已存在时删除的操作
-- 或者 spring.batch.initialize-schema= ,第一次使用 always, 下次启动时改为 never 。
# drop table if exists batch_job_seq;
# drop table if exists batch_job_execution_seq;
# drop table if exists batch_job_execution_params;
# drop table if exists batch_job_execution_context;
# drop table if exists batch_step_execution_context;
# drop table if exists batch_step_execution_seq;
# drop table if exists batch_step_execution;
# drop table if exists batch_job_execution;
# drop table if exists batch_job_instance;
```

[**项目源码 -> GitHub**](https://github.com/gxing19/Spring-Boot-Example/tree/master/spring-boot-spring-batch)


[其它可参考]
[Spring Batch 轻量级批处理框架实践](https://mp.weixin.qq.com/s/8OX8P9Li-nZ9tr4BHewrfA)
[一篇文章全面解析大数据批处理框架Spring Batch](https://mp.weixin.qq.com/s/DAAYyifUJ49TDvp87TWdxQ)
[大数据批处理框架Spring Batch+spring boot+quartz](https://blog.csdn.net/william_jm/article/details/78964538)
[Spring Batch在大型企业中的最佳实践](https://mp.weixin.qq.com/s/ocneY0xTYGMf7BmJ28m7jA)
[构建企业级批处理应用（一）](https://mp.weixin.qq.com/s/i-W-S4lpn108pZAlk-K7xA)
[构建企业级批处理应用（二）](https://mp.weixin.qq.com/s/UuMqz4WxjkB1EvIfQYbpnA)