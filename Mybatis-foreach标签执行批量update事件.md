# Mybatis-plus存在的问题

## 问题背景

项目中遇到需要批量更新操作时，使用mapper.xml的<foreach>标签进行多语句执行操作

## 问题描述

在项目中正常使用mybatis-plus时，遇到了批量更新的语句，在xml中的sql语句如下

```xml
<update id="addStudy">
        <foreach collection="list" item="item" index="index" separator=";">
            update study set num = num - #{item.num} where id = #{item.id}
        </foreach>
</update>
```

这时候数据库url连接那里是启用了allowMultiQueries=true
按理说更新语句应该没问题，我们执行后控制台所打印的sql语句也是正常的
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/ea797a822d3eacbe689099eb3e9af7f5.png)
语句看起来没问题，复制过去navicat执行也没问题，当我以为万事大吉的时候，数据库数据抗议了，他出现问题了
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/b81d375ca65e7d439dd710f3a865d666.png)
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/5e70d9ccd2cc8e2c6be68ff30e33ff8e.png)
同样的减2，后两条却成倍减少，不信邪再执行一遍
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/4f83e81775d29e94196dd35c601a5073.png)
ok差距更大了，这实在摸不着头脑，向同事请教，反复排查，最后只能寄托在主管身上

## 解决办法

首先，未得到结果前，我尝试换了新一种执行办法，不用多语句更新操作是可行的

### 第一种解决办法

```java
public String updateInventoryDecreaseBatch(Map<String, Object> map) {
        List<InventoryFormDetail> newInvGoodsList = (List<InventoryFormDetail>) map.get("newInvGoodsList");

        StringBuilder sb = new StringBuilder();
        sb.append("UPDATE study SET num = CASE id ");
        for (InventoryFormDetail item : newInvGoodsList) {
            sb.append("WHEN ").append(item.id()).append(" THEN COALESCE(num, 0) - ").append(item.num()).append(" ");
        }
        sb.append("END WHERE id IN (");
        for (InventoryFormDetail item : newInvGoodsList) {
            sb.append(item.id()).append(",");
        }
        sb.deleteCharAt(sb.length() - 1); // 删除最后一个逗号

        return sb.toString();
   }
```

这里的操作其实就是执行了单语句的批量update操作

```mysql
UPDATE study SET num = CASE id 
WHEN 1 THEN COALESCE(num, 0) - 2
WHEN 2 THEN COALESCE(num, 0) - 2 
WHEN 3 THEN COALESCE(num, 0) - 2 
END 
WHERE id IN (1,2,3)
```

### 第二种解决办法

后续主管提供了druid的执行日志，并在日志中指出了问题所在
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/8c9860ffea7d878bc0c6572818ffe869.png)
问题就出现在框中，sql在执行前，执行了一遍explain，但是由于是多语句，被分号提前结束掉，后续的两条便不是explain而是直接执行，也就导致最后在执行时，其实后两条已经是第二遍了。
但是我在个人项目中，尝试复现这个问题时，复现不出来，同时我也打了一份我的日志
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/be0bc05a549aafed73546adf0b3274f8.png)
我并没有所谓的explain，起初都以为是我druid版本不一致，后续我将版本调成一致也没问题，也是没有explain

算了，不罗嗦太多了，直接进入主题，经过一系列查找，最终定位到该图
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/74e14ae244670e25e6891210e8d233aa.png)
问题在于mybatis的拦截器SqlExplainInterceptor，这个拦截器在后续版本被优化了，不会再出现类似问题，但是如果版本比较老，该拦截器有一定漏洞，会出现以上问题

解决办法很简单，升级mybatis-plus、重写拦截器、或注释掉该拦截器，建议是注释掉该拦截器，该拦截器是不推荐再生产环境使用的，在开发需要排查时用一用就好

以下附带前后新旧两种SqlExplainInterceptor的源码

```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by FernFlower decompiler)
//
@Intercepts({@Signature(
    type = Executor.class,
    method = "update",
    args = {MappedStatement.class, Object.class}
)})
public class SqlExplainInterceptor implements Interceptor {
    private static final Log logger = LogFactory.getLog(SqlExplainInterceptor.class);
    private final String minMySQLVersion = "5.6.3";
    private boolean stopProceed = false;

    public SqlExplainInterceptor() {
    }

    public Object intercept(Invocation invocation) throws Throwable {
        MappedStatement ms = (MappedStatement)invocation.getArgs()[0];
        if (ms.getSqlCommandType() == SqlCommandType.DELETE || ms.getSqlCommandType() == SqlCommandType.UPDATE) {
            Executor executor = (Executor)invocation.getTarget();
            Configuration configuration = ms.getConfiguration();
            Object parameter = invocation.getArgs()[1];
            BoundSql boundSql = ms.getBoundSql(parameter);
            Connection connection = executor.getTransaction().getConnection();
            String databaseVersion = connection.getMetaData().getDatabaseProductVersion();
            if (GlobalConfigUtils.getDbType(configuration).equals(DBType.MYSQL) && VersionUtils.compare("5.6.3", databaseVersion)) {
                logger.warn("Warn: Your mysql version needs to be greater than '5.6.3' to execute of Sql Explain!");
                return invocation.proceed();
            }

            this.sqlExplain(configuration, ms, boundSql, connection, parameter);
        }

        return invocation.proceed();
    }

    protected void sqlExplain(Configuration configuration, MappedStatement mappedStatement, BoundSql boundSql, Connection connection, Object parameter) {
        StringBuilder explain = new StringBuilder("EXPLAIN ");
        explain.append(boundSql.getSql());
        String sqlExplain = explain.toString();
        StaticSqlSource sqlsource = new StaticSqlSource(configuration, sqlExplain, boundSql.getParameterMappings());
        MappedStatement.Builder builder = new MappedStatement.Builder(configuration, "explain_sql", sqlsource, SqlCommandType.SELECT);
        builder.resultMaps(mappedStatement.getResultMaps()).resultSetType(mappedStatement.getResultSetType()).statementType(mappedStatement.getStatementType());
        MappedStatement queryStatement = builder.build();
        DefaultParameterHandler handler = new DefaultParameterHandler(queryStatement, parameter, boundSql);

        try {
            PreparedStatement stmt = connection.prepareStatement(sqlExplain);
            Throwable var13 = null;

            try {
                handler.setParameters(stmt);
                ResultSet rs = stmt.executeQuery();
                Throwable var15 = null;

                try {
                    while(rs.next()) {
                        if (!"Using where".equals(rs.getString("Extra"))) {
                            if (this.isStopProceed()) {
                                throw new MybatisPlusException("Error: Full table operation is prohibited. SQL: " + boundSql.getSql());
                            }
                            break;
                        }
                    }
                } catch (Throwable var40) {
                    var15 = var40;
                    throw var40;
                } finally {
                    if (rs != null) {
                        if (var15 != null) {
                            try {
                                rs.close();
                            } catch (Throwable var39) {
                                var15.addSuppressed(var39);
                            }
                        } else {
                            rs.close();
                        }
                    }

                }
            } catch (Throwable var42) {
                var13 = var42;
                throw var42;
            } finally {
                if (stmt != null) {
                    if (var13 != null) {
                        try {
                            stmt.close();
                        } catch (Throwable var38) {
                            var13.addSuppressed(var38);
                        }
                    } else {
                        stmt.close();
                    }
                }

            }

        } catch (Exception var44) {
            throw new MybatisPlusException(var44);
        }
    }

    public Object plugin(Object target) {
        return target instanceof Executor ? Plugin.wrap(target, this) : target;
    }

    public void setProperties(Properties prop) {
        String stopProceed = prop.getProperty("stopProceed");
        if (StringUtils.isNotEmpty(stopProceed)) {
            this.stopProceed = Boolean.valueOf(stopProceed);
        }

    }

    public boolean isStopProceed() {
        return this.stopProceed;
    }

    public void setStopProceed(boolean stopProceed) {
        this.stopProceed = stopProceed;
    }
}

```

```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by FernFlower decompiler)
//
@Intercepts({@Signature(
    type = Executor.class,
    method = "update",
    args = {MappedStatement.class, Object.class}
)})
public class SqlExplainInterceptor extends AbstractSqlParserHandler implements Interceptor {
    private static final Log logger = LogFactory.getLog(SqlExplainInterceptor.class);
    private Properties properties;

    public Object intercept(Invocation invocation) throws Throwable {
        Object[] args = invocation.getArgs();
        MappedStatement ms = (MappedStatement)args[0];
        Object parameter = args[1];
        Configuration configuration = ms.getConfiguration();
        Object target = invocation.getTarget();
        StatementHandler handler = configuration.newStatementHandler((Executor)target, ms, parameter, RowBounds.DEFAULT, (ResultHandler)null, (BoundSql)null);
        this.sqlParser(SystemMetaObject.forObject(handler));
        return invocation.proceed();
    }

    public Object plugin(Object target) {
        return target instanceof Executor ? Plugin.wrap(target, this) : target;
    }

    public void setProperties(Properties prop) {
        this.properties = prop;
    }

    public boolean equals(Object o) {
        if (o == this) {
            return true;
        } else if (!(o instanceof SqlExplainInterceptor)) {
            return false;
        } else {
            SqlExplainInterceptor other = (SqlExplainInterceptor)o;
            if (!other.canEqual(this)) {
                return false;
            } else if (!super.equals(o)) {
                return false;
            } else {
                Object this$properties = this.getProperties();
                Object other$properties = other.getProperties();
                if (this$properties == null) {
                    if (other$properties != null) {
                        return false;
                    }
                } else if (!this$properties.equals(other$properties)) {
                    return false;
                }

                return true;
            }
        }
    }

    protected boolean canEqual(Object other) {
        return other instanceof SqlExplainInterceptor;
    }

    public int hashCode() {
        int PRIME = true;
        int result = super.hashCode();
        Object $properties = this.getProperties();
        result = result * 59 + ($properties == null ? 43 : $properties.hashCode());
        return result;
    }

    public SqlExplainInterceptor() {
    }

    public Properties getProperties() {
        return this.properties;
    }

    public String toString() {
        return "SqlExplainInterceptor(properties=" + this.getProperties() + ")";
    }
}

```

[Mybatis-Plus（三）](https://blog.csdn.net/weixin_56697114/article/details/118873075)在该文中有介绍一些mybatis-plus的使用，其中就有提到mp的分析插件【在MP中提供了对SQL执行的分析的插件，可用作阻断全表更新、删除的操作，注意：该插件仅适用于开发环境，不适用于生产环境。】

以上文中涉及的图片代码均为后面复现所截
