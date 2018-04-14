Mybatis
===
优雅的用注解形式开始Mybatis
---
依赖:注解式的mybatis需要spring支持，依赖方式有两种
---
共同依赖：数据库和连接池支持

在这里写下mysql和H2两种数据库

H2是内嵌的数据库，用来测试
* compile "mysql:mysql-connector-java:${mysqlConnectorVersion}"
* compile "com.zaxxer:HikariCP:${hikariVersion}"
* compile "com.h2database:h2:$h2Version"

第一种方式：springboot和mybatis的合体包，对低版本的spring-boot不支持
* compile "org.mybatis.spring.boot:mybatis-spring-boot-starter:${mybatisSpringVersion}"

第二种方式：spring+mybatis
* compile "org.mybatis:mybatis:${mybatisVersion}"
* compile "org.mybatis:mybatis-spring:${mybatisSpringVersion}"
* compile "org.springframework:spring-jdbc:${springVersion}"

mybatis配置
---
* mysql数据源配置
```
@Configuration
@MapperScan("com.yiran.service.galerie.repository.mapper")
@Profile(value = "default")
public class MysqlDataSourceConfig {

    private static final String URL = "jdbc:mysql://localhost:3306/galerie?zeroDateTimeBehavior=convertToNull&useSSL=false";
    private static final String USERNAME = "root";
    private static final String PASSWORD = "root";

    @Bean(destroyMethod = "close")
    public DataSource dataSource() {
        HikariDataSource dataSource = new HikariDataSource();
        dataSource.setJdbcUrl(URL);
        dataSource.setUsername(USERNAME);
        dataSource.setPassword(PASSWORD);
        dataSource.setIdleTimeout(120000);
        dataSource.setMinimumIdle(5);
        dataSource.setMaximumPoolSize(10);
        dataSource.setDriverClassName("com.mysql.jdbc.Driver");
        return dataSource;
    }

    @Bean
    public DataSourceTransactionManager transactionManager() {
        return new DataSourceTransactionManager(dataSource());
    }

    @Bean
    public SqlSessionFactory sqlSessionFactoryBean() throws Exception {
        SqlSessionFactoryBean sessionFactoryBean = new SqlSessionFactoryBean();
        sessionFactoryBean.setDataSource(dataSource());
        Properties properties = new Properties();
        properties.setProperty("dialect","mysql");
        sessionFactoryBean.setConfigurationProperties(properties);
        return sessionFactoryBean.getObject();
    }

}
```
* H2数据源配置
```
@Configuration
@MapperScan("com.yiran.service.galerie.repository.mapper")
@Profile(value = "test")
public class H2DataSourceConfig {
    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder().setType(EmbeddedDatabaseType.H2)
                .addScript("h2/galerie_bucket_rate_limit.sql")
                .addScript("h2/galerie_cdn_config.sql")
                .addScript("h2/galerie_cos_bucket_info_config.sql")
                .addScript("h2/galerie_oss_bucket_info_config.sql")
                .addScript("h2/galerie_oss_cos_gray_config.sql")
                .setScriptEncoding("UTF-8")
                .build();
    }

    @Bean
    public JdbcTemplate jdbcTemplate() {
        return new JdbcTemplate(dataSource());
    }

    @Bean
    public DataSourceTransactionManager transactionManager() {
        return new DataSourceTransactionManager(dataSource());
    }

    @Bean
    public SqlSessionFactory sqlSessionFactoryBean() throws Exception {
        SqlSessionFactoryBean sessionFactoryBean = new SqlSessionFactoryBean();
        sessionFactoryBean.setDataSource(dataSource());
        return sessionFactoryBean.getObject();
    }
}
```
开始mybatis的mapper之旅
---
* @Insert
    * #{}：首选，貌似是编译后注入sql
    * ${}：（暂时没用，以后更新）
* @Select
    * 结果映射：@Results @Result
    * （条件选择：待业务使用后更新）
* @Update
* @Delete
```
@Mapper
public interface BucketRateLimitConfigMapper {

    @Insert("INSERT INTO `bucket_rate_limit`(`bucket_tag`,`rate_limit`)" +
            "VALUES(#{bucketTag}, #{rateLimit})")
    void insert(BucketRateLimitConfig bucketRateLimitConfig);

    @Select("select * from `bucket_rate_limit`")
    @Results({
            @Result(column = "id", property = "id"),
            @Result(column = "bucket_tag", property = "bucketTag"),
            @Result(column = "rate_limit", property = "rateLimit")
    })
    List<BucketRateLimitConfig> selectAll();

    @Update("UPDATE `bucket_rate_limit` SET" +
            "`bucket_tag` = #{bucketTag}," +
            "`rate_limit` = #{rateLimit}" +
            "WHERE `id` = #{id};")
    void update(BucketRateLimitConfig bucketRateLimitConfig);

    @Delete("delete from `bucket_rate_limit` where id=#{id}")
    void delete(Long id);

    @Select("select * from `bucket_rate_limit` where bucket_tag=#{bucketTag}")
    @Results({
            @Result(column = "id", property = "id"),
            @Result(column = "bucket_tag", property = "bucketTag"),
            @Result(column = "rate_limit", property = "rateLimit")
    })
    BucketRateLimitConfig selectDetail(String bucketTag);

}
```
之后就可以@AutoWired了

H2测试
---
* 
```
  //声明使用测试配置
  @ActiveProfiles("test")
```
* 
```
//注入.sql作为启动容器的初始数据库内容配置
//sql文件在main/resource/base/h2/galerie_bucket_rate_limit.sql（哪里是classpath未仔细研究）
//以下代码位于test内
@SqlGroup({
    @Sql(scripts = "classpath:h2/galerie_bucket_rate_limit.sql", executionPhase = Sql.ExecutionPhase.BEFORE_TEST_METHOD),
})
```

方法级事务回滚
---
* 需要开启事务处理，mysql使用innodb
```
@Transactional
    public BaseResponse execute(CdnConfigCreateRequest request) {
        ...
    }
```