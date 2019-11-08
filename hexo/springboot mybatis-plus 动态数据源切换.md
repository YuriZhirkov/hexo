# springboot mybatis-plus 动态数据源切换

##  一、前言 
 由于项目中读写分离，或者分库分表导致数据库连接有很多。这个时候我们常常会切换多数据源进行业务的合并，但是mybatis-plus 里面内置的方法却很难试用，需要编写大量代码来实现，非常繁琐，我在网上找了很多demo部分还不能成功，走了很弯路以后，我发现
 maonidou(苞米豆)团队针对springboot使用多数据源提供了一个启动器:dynamic-datasource-spring-boot-starter
 可以快速启动数据源切换，只需要一个jar包和几句注解即可搞定，话不多说，让我们一起来看看。

 > 注：本文参考：https://www.cnblogs.com/SimpleWu/p/10930388.html

## 二、快速开始

 - 首先通过idea创建一个新的springboot项目，如果已经有项目可以只接引入配置即可。
 - 需要加入的jar
   
       <!-- mybatis plus -->
		<dependency>
			<groupId>com.baomidou</groupId>
			<artifactId>mybatis-plus-boot-starter</artifactId>
			<version>3.1.1</version>
		</dependency>

		<!-- mysql -->
		<dependency>
			<groupId>mysql</groupId>
			<artifactId>mysql-connector-java</artifactId>
			<version>5.1.47</version>
		</dependency>


		<!-- druid数据连接池 -->
		<dependency>
			<groupId>com.alibaba</groupId>
			<artifactId>druid-spring-boot-starter</artifactId>
			<version>1.1.10</version>
		</dependency>
	
		<!-- 动态数据源(重点) -->
		<dependency>
			<groupId>com.baomidou</groupId>
			<artifactId>dynamic-datasource-spring-boot-starter</artifactId>
			<version>2.5.4</version>
		</dependency>

- yml 文件配置 application.yml

```
spring:
  datasource:
    # 动态数据源配置
    dynamic:
      primary: master
      druid: #以下是全局默认值，可以全局更改
        initial-size: 0
        max-active: 8
        min-idle: 2
        max-wait: -1
        min-evictable-idle-time-millis: 30000
        max-evictable-idle-time-millis: 30000
        time-between-eviction-runs-millis: 0
        validation-query: select 1
        validation-query-timeout: -1
        test-on-borrow: false
        test-on-return: false
        test-while-idle: true
        pool-prepared-statements: true
        max-open-prepared-statements: 100
        filters: stat,wall
        share-prepared-statements: true
      # 配置需要的数据源
      datasource:
        # master为默认的数据源
        master:
          username: root
          password: root
          url: jdbc:mysql://127.0.0.1:3306/wh?@useUnicode=true&characterEncoding=UTF-8
          driver-class-name: com.mysql.jdbc.Driver
          druid:
            initial-size: 5
        test:
          username: root
          password: root
          url: jdbc:mysql://127.0.0.1:3306/test?@useUnicode=true&characterEncoding=UTF-8
          driver-class-name: com.mysql.jdbc.Driver
          druid:
            initial-size: 6
```

-  如果你项目中使用的  application.properties 
```
#设置默认的数据源或者数据源组,默认值即为master
spring.datasource.dynamic.primary=master
#主库配置
spring.datasource.dynamic.datasource.master.username=root
spring.datasource.dynamic.datasource.master.password=root
spring.datasource.dynamic.datasource.master.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.dynamic.datasource.master.url=jdbc:mysql://127.0.0.1:3306/wh?@useUnicode=true&characterEncoding=UTF-8useSSL=false&characterEncoding=utf8
#从库配置
spring.datasource.dynamic.datasource.test.username=root
spring.datasource.dynamic.datasource.test.password=root
spring.datasource.dynamic.datasource.test.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.dynamic.datasource.test.url=mysql://127.0.0.1:3306/test?@useUnicode=true&characterEncoding=UTF-8useSSL=false&characterEncoding=utf8
```
- 在代码中切换注解使用
```
	@Autowired
    private DeptMapper deptMapper;

    /**
     * 查询所有数据
     * @return
     */
    public List<Dept> getList() {
        return deptMapper.selectList(null);
    }

    /**
     * 通过mapper文件SQL进行查询
     * @param deptNo
     * @return
     */
    @DS("test") // 指定数据库连接
    public Map deptMapper(Integer deptNo) {
        return deptMapper.queryDept(deptNo);
    }

    /**
     * 分页查询自定数据库
     * @return
     */
    @DS("master") // 指定数据库连接
    public IPage<Dept> queryPageDept() {
        IPage<Dept> page = new Page(1,2);
        page =  deptMapper.selectPage(page,null);
        return page;
    }

    /**
     * 修改
     * @return
     */
    @DS("master")
    @Transactional//本地事务开启
    public boolean save() {
        Dept vo = new Dept();
        vo.setAvgSal(55.5);
        vo.setCreateDate(new Date());
        vo.setDName("业务部");
        vo.setLoc("上海");
        vo.setDeptNo(2300);
        return deptMapper.insert(vo) > 0;
    }

    /**
     * 删除
     * @return
     */
    @DS("master")
    @Transactional//本地事务开启
    public boolean remove() {
        return deptMapper.deleteById(2300) > 0;
    }

    /**
     * 修改
     * @return
     */
    @DS("test")
    @Transactional//本地事务开启
    public boolean update() {
        Dept vo = new Dept();
        vo.setAvgSal(55.5);
        vo.setCreateDate(new Date());
        vo.setDName("业务部");
        vo.setLoc("上海");
        vo.setDeptNo(2300);
        return deptMapper.updateById(vo) > 0;
    }
```

- 结果


http://localhost:9527/whsoftinc/dept/deptList 查询主数据库
```
[{"deptNo":1,"avgSal":500.0,"createDate":"2019-09-07 00:00:00","loc":"武汉","dname":"销售部（主）"},{"deptNo":1002,"avgSal":500.0,"createDate":"2019-09-07 00:00:00","loc":"北京","dname":"开发部（主）"},{"deptNo":1003,"avgSal":500.0,"createDate":"2019-09-07 00:00:00","loc":"上海","dname":"财务部（主）"},{"deptNo":1004,"avgSal":500.0,"createDate":"2019-09-07 00:00:00","loc":"上海","dname":"财务部（主）"}]
```

http://localhost:9527/whsoftinc/dept/deptList 查询从数据库

```
{"loc":"上海","avgSal":500.0,"dName":"财务部（从）","deptNo":1003,"createDate":"2019-09-07"}
```


  ## 三、注意事项

  **@DS** 可以注解在方法上和类上，同时存在方法注解优先于类上注解。

注解在service实现或mapper接口方法上，但强烈不建议同时在service和mapper注解。 (可能会有问题)

如果不加入主键则使用默认数据源。

DruidDataSourceAutoConfigure会注入一个DataSourceWrapper，其会在原生的spring.datasource下找url,username,password等。而我们动态数据源的配置路径是变化的,所以需要排除：
```
spring:
  autoconfigure:
    exclude: com.alibaba.druid.spring.boot.autoconfigure.DruidDataSourceAutoConfigure
```

或者在类上排除：

```
@SpringBootApplication(exclude = DruidDataSourceAutoConfigure.class)
```

 1. 本框架只做 切换数据源 这件核心的事情，并不限制你的具体操作，切换了数据源可以做任何CRUD
 2. 配置文件所有以下划线 _ 分割的数据源 首部 即为组的名称，相同组名称的数据源会放在一个组下。
 3. 切换数据源即可是组名，也可是具体数据源名称，切换时默认采用负载均衡机制切换。
 4. 默认的数据源名称为 master ，你可以通过spring.datasource.dynamic.primary修改。
 5. 方法上的注解优先于类上注解。


  ## 四、代码demo参考
> 本文代码地址：https://github.com/wuhengc/springboot-dynamic-datasource
> 个人博客地址：https://www.whsoftinc.com
> 如有疑问请联系我：33993155