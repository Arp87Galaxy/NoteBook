# mybatis 面试

## 1.参数处理

### 1.1单个参数:mybatis不会做特殊处理，

​	#{参数名/任意名}：取出参数值。

 

### 1.2多个参数:mybatis会做特殊处理。

​	多个参数会被封装成 一个map，

​		key：param1...paramN,或者参数的索引也可以

​		value：传入的参数值

​	#{}就是从map中获取指定的key的值；

 

​	异常：(错误操作产生的异常)

​	org.apache.ibatis.binding.BindingException: 

​	Parameter 'id' not found. 

​	Available parameters are [1, 0, param1, param2]

​	操作：(错误操作)

​		方法：public Employee getEmpByIdAndLastName(Integer id,String lastName);

​		取值：#{id},#{lastName}



### 1.3【命名参数】：

​	明确指定封装参数时map的key；@Param("id")

​	多个参数会被封装成 一个map，

​		key：使用@Param注解指定的值

​		value：参数值

​	#{指定的key}取出对应的参数值





POJO：

如果多个参数正好是我们业务逻辑的数据模型，我们就可以直接传入pojo；

​	#{属性名}：取出传入的pojo的属性值	



Map：

如果多个参数不是业务模型中的数据，没有对应的pojo，不经常使用，为了方便，我们也可以传入map

​	#{key}：取出map中对应的值



TO：

如果多个参数不是业务模型中的数据，但是经常要使用，推荐来编写一个TO（Transfer Object）数据传输对象

Page{

​	int index;

​	int size;

}



========================思考================================	

public Employee getEmp(@Param("id")Integer id,String lastName);

​	取值：id==>#{id/param1}   lastName==>#{param2}



public Employee getEmp(Integer id,@Param("e")Employee emp);

​	取值：id==>#{param1}    lastName===>#{param2.lastName/e.lastName}



\##特别注意：如果是Collection（List、Set）类型或者是数组，

​		 也会特殊处理。也是把传入的list或者数组封装在map中。

​			key：Collection（collection）,如果是List还可以使用这个key(list)

​				数组(array)

public Employee getEmpById(List<Integer> ids);

​	取值：取出第一个id的值：   #{list[0]}

 

========================结合源码，mybatis怎么处理参数==========================

总结：参数多时会封装map，为了不混乱，我们可以使用@Param来指定封装时使用的key；

\#{key}就可以取出map中的值；



(@Param("id")Integer id,@Param("lastName")String lastName);

ParamNameResolver解析参数封装map的；

//1、names：{0=id, 1=lastName}；构造器的时候就确定好了



​	确定流程：

​	1.获取每个标了**param注解**的参数的@Param的值：id，lastName；  赋值给name;

​	2.每次解析一个参数给map中保存信息：（key：参数索引，value：name的值）

​		name的值：

​			标注了param注解：注解的值

​			没有标注：

​				1.全局配置：useActualParamName（jdk1.8）(这个属性需要在全局配置文件的setting中设置))：name=参数名

​				2.name=map.size()；相当于当前元素的索引

​	{0=id, 1=lastName,2=2}

​    



args【1，"Tom",'hello'】:



public Object getNamedParams(Object[] args) {

​    final int paramCount = names.size();

​    //1、参数为null直接返回

​    if (args == null || paramCount == 0) {

​      return null;

​     

​    //2、如果只有一个元素，并且没有Param注解；args[0]：单个参数直接返回

​    } else if (!hasParamAnnotation && paramCount == 1) {

​      return args[names.firstKey()];

​      

​    //3、多个元素或者有Param标注

​    } else {

​      final Map<String, Object> param = new ParamMap<Object>();

​      int i = 0;

​      

​      //4、遍历names集合；{0=id, 1=lastName,2=2}

​      for (Map.Entry<Integer, String> entry : names.entrySet()) {

​      

​      	//names集合的value作为key;  names集合的key又作为取值的参考args[0]:args【1，"Tom"】:

​      	//eg:{id=args[0]:1,lastName=args[1]:Tom,2=args[2]}

​        param.put(entry.getValue(), args[entry.getKey()]);

​        

​        

​        // add generic param names (param1, param2, ...)param

​        //额外的将每一个参数也保存到map中，使用新的key：param1...paramN

​        //效果：有Param注解可以#{指定的key}，或者#{param1}

​        final String genericParamName = GENERIC_NAME_PREFIX + String.valueOf(i + 1);

​        // ensure not to overwrite parameter named with @Param

​        if (!names.containsValue(genericParamName)) {

​          param.put(genericParamName, args[entry.getKey()]);

​        }

​        i++;

​      }

​      return param;

​    }

  }

}

===========================参数值的获取======================================

\#{}：可以获取map中的值或者pojo对象属性的值；

${}：可以获取map中的值或者pojo对象属性的值；





select * from tbl_employee where id=${id} and last_name=#{lastName}

Preparing: select * from tbl_employee where id=2 and last_name=?

​	区别：

​		#{}:是以预编译的形式，将参数设置到sql语句中；PreparedStatement；防止sql注入

​		${}:取出的值直接拼装在sql语句中；会有安全问题；

​		大多情况下，我们去参数的值都应该去使用#{}；

  

​		原生jdbc不支持占位符的地方我们就可以使用${}进行取值

​		比如分表、排序。。。；按照年份分表拆分

​			select * from ${year}_salary where xxx;

​			select * from tbl_employee order by ${f_name} ${order}



\#{}:更丰富的用法：

​	规定参数的一些规则：

​	javaType、 jdbcType、 mode（存储过程）、 numericScale、

​	resultMap、 typeHandler、 jdbcTypeName、 expression（未来准备支持的功能）；



​	jdbcType通常需要在某种特定的条件下被设置：

​		在我们数据为null的时候，有些数据库可能不能识别mybatis对null的默认处理。比如Oracle（报错）；

  

​		JdbcType OTHER：无效的类型；因为mybatis对所有的null都映射的是原生Jdbc的OTHER类型，oracle不能正确处理;

  

​		由于全局配置中：jdbcTypeForNull=OTHER；oracle不支持；两种办法

​		1、#{email,jdbcType=OTHER};

​		2、jdbcTypeForNull=NULL

​			<setting name="jdbcTypeForNull" value="NULL"/>

   ### 1.4 @MapKey("object")

作用在方法上，这样返回的map中有一个{”object“，object},就不用再建一个object实体类了

## 2.缓存

一级缓存影响不可重复读隔离级别吗？

### 2.1 一级缓存

sqlsession级别 每次sqlsession关闭 缓存就存入

setting中的设置localCacheScope,默认值为session

当值为statement时,可以关闭一级缓存，sqlsession.clearcache（）也可以手动删除缓存

### 2.2 二级缓存 需要序列化

<cache/>同一namespace（mapper级别）

<cache-ref namespace="com.someone.application.data.SomeMapper"/> 引用其他namespace



```xml
<settings>
  <setting name="cacheEnabled" value="false" />
</settings>
```

```text
flase:
 			executor = new SimpleExecutor(this, transaction);
true:
            executor = new CachingExecutor((Executor)executor);
```

```xml
<mapper namespace="...UserMapper">
    <cache
  		eviction="FIFO"
 		flushInterval="60000"
 		size="512"
  		readOnly="true"/><!-- 加上该句即可，使用默认配置、还有另外一种方式，在后面写出 -->
    ...
</mapper>

```

```java
    public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
        BoundSql boundSql = ms.getBoundSql(parameterObject);
        CacheKey key = this.createCacheKey(ms, parameterObject, rowBounds, boundSql);        return this.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
    }    public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
        Cache cache = ms.getCache();        if (cache != null) {//第一个条件 定义需要使用的cache  
            this.flushCacheIfRequired(ms);            if (ms.isUseCache() && resultHandler == null) {//第二个条件 需要当前的查询语句是配置了使用cache的，即下面源码的useCache()是返回true的  默认是true
                this.ensureNoOutParams(ms, parameterObject, boundSql);
                List<E> list = (List)this.tcm.getObject(cache, key);                if (list == null) {
                    list = this.delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);                    this.tcm.putObject(cache, key, list);
                }                return list;
            }
        }        return this.delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
    }
```

**某个select语句不使用用二级缓存可以useCache=”false“，一级依然使用**

​									**flushCache="false" 每次执行完一级二级都会删除**

**<cache></cache>**

cache标签中的属性

eviction:缓存的回收策略：

   • LRU – 最近最少使用的：移除最长时间不被使用的对象。

   • FIFO – 先进先出：按对象进入缓存的顺序来移除它们。

   • SOFT – 软引用：移除基于垃圾回收器状态和软引用规则的对象。

   • WEAK – 弱引用：更积极地移除基于垃圾收集器状态和弱引用规则的对象。

   • 默认的是 LRU。

flushInterval：缓存刷新间隔,缓存多长时间清空一次，默认不清空，设置一个毫秒值

readOnly:是否只读：(true或者false)(默认是false)

​		true：只读；mybatis认为所有从缓存中获取数据的操作都是只读操作，不会修改数据。

​					mybatis为了加快获取速度，直接就会将数据在缓存中的引用交给用户。不安全,速度快

​		false：非只读：mybatis觉得获取的数据可能会被修改。

​      			 mybatis会利用序列化&反序列的技术克隆一份新的数据给你。安全，速度慢

 

size：缓存存放多少元素；

type=""：指定自定义缓存的全类名；

​	       实现Cache接口即可；

## 3 动态sql

OGNL表达式

### 3.1关键字

- if     if标签字段：test=“表达式”

- choose (when, otherwise)

- trim  可替换where和set（整个 串完成后添加前缀，并且if里面的and前后缀都可以覆盖），where（去掉if的前面的and）, set（主要用于拼装update的 ，添加一个SET 关键字 并里面的if拼装）

- foreach  List、Set 、Map ，数组。

  当使用 Map 对象（或者 Map.Entry 对象的集合）时，index 是键，item 是值。

  ```xml
  <select id="selectPostIn" resultType="domain.blog.Post">
    SELECT *
    FROM POST P
    WHERE ID in
    <foreach item="item" index="index" collection="list"
        open="(" separator="," close=")">
          #{item}
    </foreach>
  </select>
  ```

### 3.2内置参数

```
_parameter 代表整个参数 
				1.单个参数 _parameter的值就是这个参数
				2.多个参数 会被封装成map _parameter的值就是map对象
_databaseId 如果全局配置中配置了 databaseIdProvider。 _databaseId就是该id
```

### 3.3 bind 

绑定一个自定义变量pattern到当前操作标签上下文

解决类似like 不能“  %#{name}%  "的问题

```xml
<select id="selectBlogsLike" resultType="Blog">
  <bind name="pattern" value="'%' + _parameter.getTitle() + '%'" /> 使用+连接
  SELECT * FROM BLOG
  WHERE title LIKE #{pattern}
</select>
```

### 3.4 sql标签与select同级,include标签select子级

<sql id="sqlid">

字符串。。。

</sql>

在标签里

<select id="" >

​		<include refid="sqlid"></include>

</select>

## 4 映射文件  Mapper.xml



### 4.1标签

- `cache` – 该命名空间的缓存配置。
- `cache-ref` – 引用其它命名空间的缓存配置。
- `resultMap` – 描述如何从数据库结果集中加载对象，是最复杂也是最强大的元素。
- `sql` – 可被其它语句引用的可重用语句块。
- `insert` – 映射插入语句。
- `update` – 映射更新语句。
- `delete` – 映射删除语句。
- `select` – 映射查询语句。

### 4.2 insert, update 和 delete

|                    |                                                              |
| ------------------ | :----------------------------------------------------------- |
| `id`               | 在命名空间中唯一的标识符，可以被用来引用这条语句。           |
| `parameterType`    | 将会传入这条语句的参数的类全限定名或别名。这个属性是可选的，因为 MyBatis 可以通过类型处理器（TypeHandler）推断出具体传入语句的参数，默认值为未设置（unset）。 |
| `parameterMap`     | 用于引用外部 parameterMap 的属性，目前已被废弃。请使用行内参数映射和 parameterType 属性。 |
| `flushCache`       | 将其设置为 true 后，只要语句被调用，都会导致本地缓存和二级缓存被清空，默认值：（对 insert、update 和 delete 语句）true。 |
| `timeout`          | 这个设置是在抛出异常之前，驱动程序等待数据库返回请求结果的秒数。默认值为未设置（unset）（依赖数据库驱动）。 |
| `statementType`    | 可选 STATEMENT，PREPARED 或 CALLABLE。这会让 MyBatis 分别使用 Statement，PreparedStatement 或 CallableStatement，默认值：PREPARED。 |
| `useGeneratedKeys` | （仅适用于 insert 和 update）这会令 MyBatis 使用 JDBC 的 getGeneratedKeys 方法来取出由数据库内部生成的主键（比如：像 MySQL 和 SQL Server 这样的关系型数据库管理系统的自动递增字段），默认值：false。 |
| `keyProperty`      | （仅适用于 insert 和 update）指定能够唯一识别对象的属性，MyBatis 会使用 getGeneratedKeys 的返回值或 insert 语句的 selectKey 子元素设置它的值，默认值：未设置（`unset`）。如果生成列不止一个，可以用逗号分隔多个属性名称。 |
| `keyColumn`        | （仅适用于 insert 和 update）设置生成键值在表中的列名，在某些数据库（像 PostgreSQL）中，当主键列不是表中的第一列的时候，是必须设置的。如果生成列不止一个，可以用逗号分隔多个属性名称。 |
| `databaseId`       | 如果配置了数据库厂商标识（databaseIdProvider），MyBatis 会加载所有不带 databaseId 或匹配当前 databaseId 的语句；如果带和不带的语句都有，则不带的会被忽略。 |

#### 4.2.1 insert和update 自增主键获取

1. insert和update **返回值是修改的条数，失败抛异常**，如果配置自增主键获取，**传入id不为null，就不会使用数据库自增主键** ，返回值传入id是null，**就会将id setter入pojo的主键属性**，
   **并且innodb引擎只会将id最大的存在内存，重启失效**

2.  配置自增主键获取

   1. useGeneratedKeys=”true”，keyProperty =”id“

   2. 如果多行插入，返回所有的自增主键

      ```xml
      <insert id="insertAuthor" useGeneratedKeys="true"
          keyProperty="id">
        insert into Author (username, password, email, bio) values
        <foreach item="item" collection="list" separator=",">
          (#{item.username}, #{item.password}, #{item.email}, #{item.bio})
        </foreach>
      </insert>
      ```

### 4.3 查询

|                        |                                                              |
| ---------------------- | ------------------------------------------------------------ |
| `id`                   | 在命名空间中唯一的标识符，可以被用来引用这条语句。           |
| `parameterType`        | 将会传入这条语句的参数的类全限定名或别名。这个属性是可选的，因为 MyBatis 可以通过类型处理器（TypeHandler）推断出具体传入语句的参数，默认值为未设置（unset）。 |
| *parameterMap*（废弃） | *用于引用外部 parameterMap 的属性，目前已被废弃。请使用行内参数映射和 parameterType 属性。* |
| `resultType`           | 期望从这条语句中返回结果的类全限定名或别名。 注意，如果返回的是集合，那应该设置为集合包含的类型，而不是集合本身的类型。 resultType 和 resultMap 之间只能同时使用一个。 |
| `resultMap`            | 对外部 resultMap 的命名引用。结果映射是 MyBatis 最强大的特性，如果你对其理解透彻，许多复杂的映射问题都能迎刃而解。 resultType 和 resultMap 之间只能同时使用一个。 |
| `flushCache`           | 将其设置为 true 后，只要语句被调用，都会导致本地缓存和二级缓存被清空，默认值：false。 |
| `useCache`             | 将其设置为 true 后，将会导致本条语句的结果被二级缓存缓存起来，默认值：对 select 元素为 true。 |
| `timeout`              | 这个设置是在抛出异常之前，驱动程序等待数据库返回请求结果的秒数。默认值为未设置（unset）（依赖数据库驱动）。 |
| `fetchSize`            | 这是一个给驱动的建议值，尝试让驱动程序每次批量返回的结果行数等于这个设置值。 默认值为未设置（unset）（依赖驱动）。 |
| `statementType`        | 可选 STATEMENT，PREPARED 或 CALLABLE。这会让 MyBatis 分别使用 Statement，PreparedStatement 或 CallableStatement，默认值：PREPARED。 |
| `resultSetType`        | FORWARD_ONLY，SCROLL_SENSITIVE, SCROLL_INSENSITIVE 或 DEFAULT（等价于 unset） 中的一个，默认值为 unset （依赖数据库驱动）。 |
| `databaseId`           | 如果配置了数据库厂商标识（databaseIdProvider），MyBatis 会加载所有不带 databaseId 或匹配当前 databaseId 的语句；如果带和不带的语句都有，则不带的会被忽略。 |
| `resultOrdered`        | 这个设置仅针对嵌套结果 select 语句：如果为 true，将会假设包含了嵌套结果集或是分组，当返回一个主结果行时，就不会产生对前面结果集的引用。 这就使得在获取嵌套结果集的时候不至于内存不够用。默认值：`false`。 |
| `resultSets`           | 这个设置仅适用于多结果集的情况。它将列出语句执行后返回的结果集并赋予每个结果集一个名称，多个名称之间以逗号分隔。 |

#### 4.3.1 ResultType简单pojo

 	1.  返回List <pojo>
      1.  ResultType="pojo"
 	2.  返回map
            	1.  ResultType="map"   map里面的key默认是id 
       	2.  可用@Mapkey（”“）指定

当不满足第三范式后 ，即表连接的情况下，用到下面

#### 4.3.2 ResultMap 

​	1对1情况下，一个员工有一个部门，可以通过property=dept.name，通过<result cloumn="d_name" property="dept.name">

#### 4.3.3 ResultMap 关联  association（select和resultmap 属性）一对一

  1. 嵌套查询（嵌套select查询，也叫分步查询，主要为了延迟加载）：通过执行另外一个 SQL 映射语句来加载期望的复杂类型。

     fetchType`可选的。有效值为 `lazy` 和 `eager`。 指定属性后，将在映射中忽略全局配置参数 `lazyLoadingEnabled`，使用属性的值。当某一个不需要延迟加载设置时，可用这个来关闭

     

     

     ```xml
     <resultMap id="blogResult" type="Blog">
       <association property="author" column="author_id" javaType="Author" select="selectAuthor"/>//select可以跨namespace指定，column是传给select的值可以column="{prop1=col1,prop2=col2}"传多值
     </resultMap>
     
     <select id="selectBlog" resultMap="blogResult">
       SELECT * FROM BLOG WHERE ID = #{id}
     </select>
     
     <select id="selectAuthor" resultType="Author">
       SELECT * FROM AUTHOR WHERE ID = #{id}
     </select>
     
     //上面执行selectBlog方法会发现里面association关联了selectAuthor，于是再执行该方法
     ```

  2. 嵌套结果（嵌套结果映射）：使用嵌套的结果映射来处理连接结果的重复子集。

     ```xml
     <resultMap id="blogResult" type="Blog">
       <id property="id" column="blog_id" />
       <result property="title" column="blog_title"/>
       <association property="author" column="blog_author_id" javaType="Author" resultMap="authorResult"/>
         //如果没有写resultMap="authorResult"，也可以直接在<association>标签里面使用<id></id>
         //<result></result>标签指定属性，上面
     </resultMap>
     
     <resultMap id="authorResult" type="Author">
       <id property="id" column="author_id"/>
       <result property="username" column="author_username"/>
       <result property="password" column="author_password"/>
       <result property="email" column="author_email"/>
       <result property="bio" column="author_bio"/>
     </resultMap>
     ```

#### 4.3.4 ResultMap 集合 collection 一对多

  		1.  嵌套查询（嵌套select查询）
    		2.  嵌套结果（嵌套结果映射）

#### 4.3.5 多结果，存储过程中返回多个表的结果resultsets

#### 4.3.6 鉴别器discriminator 

根据某列的值，改变封装结果映射

```xml
<resultMap id="vehicleResult" type="Vehicle">
  <id property="id" column="id" />
  <result property="vin" column="vin"/>
  <result property="year" column="year"/>
  <result property="make" column="make"/>
  <result property="model" column="model"/>
  <result property="color" column="color"/>
  <discriminator javaType="int" column="vehicle_type">
    <case value="1" resultMap="carResult"/>
    <case value="2" resultMap="truckResult"/>
    <case value="3" resultMap="vanResult"/>
    <case value="4" resultMap="suvResult"/>
  </discriminator>
</resultMap>i 
```

## 5. 插件原理

四大对象：

​		ParameterHandler：处理SQL的参数对象

​		ResultSetHandler：处理SQL的返回结果集

​		StatementHandler：数据库的处理对象，用于执行SQL语句

​		Executor：MyBatis的执行器，用于执行增删改查操作

Interceptor接口：

```java
@Intercepts({@Signature(type = ResultSetHandler.class,method ="handleResultSets",args = Statement.class)})//type指定四大对象
public class MyFirstInterceptor implements Interceptor {

     /** @Description 拦截目标对象的目标方法 **/
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        System.out.println("拦截的目标对象："+invocation.getTarget());
        Object object = invocation.proceed();
        return object;
    }
     /**
      * @Description 包装目标对象 为目标对象创建代理对象
      * @Param target为要拦截的对象
      * @Return 代理对象
      */
    @Override
    public Object plugin(Object target) {
        System.out.println("将要包装的目标对象："+target);
        return Plugin.wrap(target,this);
    }
    /** 获取配置文件的属性 **/
    @Override
    public void setProperties(Properties properties) {
        System.out.println("插件配置的初始化参数："+properties);
    }
}
```

```xml
 <plugins
     //配置在前的插件后执行
        <plugin interceptor="mybatis.interceptor.MyFirstInterceptor">
            <!--配置参数-->
            <property name="name" value="Bob"/>
        </plugin>
    </plugins>
```

## 6.配置

- [properties（属性）](https://mybatis.org/mybatis-3/zh/configuration.html#properties)
- [settings（设置）](https://mybatis.org/mybatis-3/zh/configuration.html#settings)
- [typeAliases（类型别名）](https://mybatis.org/mybatis-3/zh/configuration.html#typeAliases)
- [typeHandlers（类型处理器）](https://mybatis.org/mybatis-3/zh/configuration.html#typeHandlers)
- [objectFactory（对象工厂）](https://mybatis.org/mybatis-3/zh/configuration.html#objectFactory)
- [plugins（插件）](https://mybatis.org/mybatis-3/zh/configuration.html#plugins)
- environments（环境配置）
  - environment（环境变量）
    - transactionManager（事务管理器）
    - dataSource（数据源）

```xml
<settings>
  <setting name="cacheEnabled" value="true"/>
  <setting name="lazyLoadingEnabled" value="true"/>
  <setting name="multipleResultSetsEnabled" value="true"/>
  <setting name="useColumnLabel" value="true"/>
  <setting name="useGeneratedKeys" value="false"/>
  <setting name="autoMappingBehavior" value="PARTIAL"/>
  <setting name="autoMappingUnknownColumnBehavior" value="WARNING"/>
  <setting name="defaultExecutorType" value="SIMPLE"/>
  <setting name="defaultStatementTimeout" value="25"/>
  <setting name="defaultFetchSize" value="100"/>
  <setting name="safeRowBoundsEnabled" value="false"/>
  <setting name="mapUnderscoreToCamelCase" value="false"/>
  <setting name="localCacheScope" value="SESSION"/>
  <setting name="jdbcTypeForNull" value="OTHER"/>
  <setting name="lazyLoadTriggerMethods" value="equals,clone,hashCode,toString"/>
</settings>
```