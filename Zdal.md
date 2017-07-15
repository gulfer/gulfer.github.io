# 分库分表实践

分库分表现在已经是一种较为常见的应对高并发、大数据量场景的解决方案。通常对数据库的拆分方式有3种，读写分离、垂直拆分和水平拆分。读写分离是指把单一按使用场景进行拆分，将读库与写库分离，使用于读多写少的情况，并且要配合写库与读库的同步来实施；垂直拆分是指将同类业务或关联性较强的表归并到一个数据库，实现业务层面的数据拆分，适用于一些业务逻辑较为独立的应用；水平拆分是指对数据量大且读写操作较频繁的数据库或数据表，按某唯一键进行拆分，将数据分散到不同的库或表上，通过减少单库单表的数据量来提升操作效率。分库分表实际上属于水平拆分。

## 分库分表常见方案

常见的分库分表方案又有两种，分别是id段分库分表和id取模分库分表。

id段分库分表的前提是确保id有序，这样就可以对id按号段进行划分，每一段id对应的记录落在对应的分库及分表上。假设分库数为m，单库分表数为j，单表记录数为o，id为i，可以确定i所属的分库号=i表示的有效序号/(j*o)，所属分表号=i/o。如下图所示：

![](https://github.com/gulfer/gulfer.github.io/blob/master/pic/split_1.png)

当数据量为2000万时，单表记录数o=100万，分库数m=2，单库分表数n=10，当id=13456789时，可根据上述公式计算出分库号=13456789/(10*1000000)=1，分表号n=13456789/1000000=13。此种方案最大的优点是，新增数据可直接通过库、表扩容进行扩展，避免数据重新分配导致的迁移，资源对数据量变化不敏感。缺点的数据分布不均容易造成写满的库表负载大，而未写满的库表负载小的情况，另外需要额外的序号生成器确保id有序。

另外一种方案是id去模分库分表，此方案对id生成规则没有要求，一般会直接对id进行hash后去模，将对应记录均匀分配到各库表中。假设分库数为m，单裤分表数为j，分表总数n=j*m，id为i，可确定i所属的分表号=i%(j * m)，分库号=(i%n)/j。如下图所示：

![](https://github.com/gulfer/gulfer.github.io/blob/master/pic/split_2.png)

我们可以预先创建4个分库，每个分库5张表，则m=4，j=5，n=m*j=20，当id=13456789时，根据上述公式可知，分表号=13456789%20=9，分库号=9/5=1。此方案的优点是所有数据按统一的规则均匀分布，避免了资源使用不均，缺点是需要预先规划好资源上限，当突破资源上限进行记录重新分配时，需要进行数据迁移。

## 索引表设计

在操作分库或分表时，比较常见的问题是当查询条件并不是分库分表键时，将无法路由到具体的分库或分表。通常做法是为每个查询条件均建立索引表，比如我们有一张用户信息表，分库分表键是用户id，而用户登录时可通过用户名、手机号登录并查询出用户id，这时就需要对用户名及手机号分别创建索引表。而如果我们把手机号、用户名作为唯一性的用户标识时，需要设计以登录名为主键的纵表或确定登录名的横标，当使用用户id查询时可以直接定位到记录。如下图所示：

![](https://github.com/gulfer/gulfer.github.io/blob/master/pic/index_table.png)

建立索引表或路由表可以解决非分库分表键查询的问题，但同时会增大数据冗余，而索引表本身一般也会做分库分表。

## Zdal使用实践

Zdal是支付宝开发的数据中间件框架，实现分库分表、failover切换等功能，支持常用的SQL语法。Zdal并不开源，git上也有一些重复的轮子，不过我们在实际项目中还是使用了Zdal。事实上各种分库分表框架的基本原理基本类似，所以在具体使用上，这里仍将以Zdal为例。

Zdal这类分库分表框架的基本原理是在JDBC层对库名、表名进行解析并替换，SQL执行完毕后会对ResultSet进行合并。而对应用来说，这些操作均被Zdal数据源屏蔽在下层，应用可以使用任何ORM框架编写SQL语句，集成毫无压力。Zdal的执行流程如下：

![](https://github.com/gulfer/gulfer.github.io/blob/master/pic/zdal1.png)

应用集成Zdal需增加三部分配置，分别是Zdal数据源配置、Zdal数据库配置、分库分表规则配置。数据源配置如下：

数据库配置如下：

分库分表规则配置如下：

```
    <bean id="appRule" class="com.alipay.zdal.rule.config.beans.AppRule">
		<property name="masterRule" ref="appRWRule"/>
	</bean>
	
	<bean id="appRWRule" class="com.alipay.zdal.rule.config.beans.ShardRule">
		<property name="tableRules">
			<map>
				<entry key="TABLE_1" value-ref="idRule"/>
			</map>
		</property>
	</bean>
	
	<bean id="idRule" class="com.alipay.zdal.rule.config.beans.TableRule" init-method="init">
		<property name="tbSuffix" value="resetForEachDB:[_0-_3]"/>
		<property name="dbIndexes" value="db0,db1"/>
		<property name="dbRuleArray">
			<list>
				<!-- <value>
					return (int)(Math.floor(Double.parseDouble(#ID#) / (2*50)));
				</value> -->
				<value>
					return (int)Math.floor(Integer.parseInt(#ID#) % (2*2) / 2);
				</value>
			</list>
		</property>
		<property name="tbRuleArray">
			<list>
				<!-- <value>
					return (Integer)(Math.floor(Double.parseDouble(#ID#) / 50));
				</value> -->
				<value>
					return (int)Math.floor(Integer.parseInt(#ID#) % (2*2));
				</value>
			</list>
		</property>
	</bean>
```

我们主要关注idRule，这里配置了具体分库分表规则。

## 小结

从SOA

