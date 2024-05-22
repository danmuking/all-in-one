## 1.1 什么是缓存
在讲解MyBatis的缓存机制之前，我们先来了解什么是缓存。
缓存就是将我们经常查询的数据的结果保存到一个内存中（缓存就是内存中的一个对象），那么在下一次查询的时候就不用到数据库文件中查询，而是从内存中获取，从而减少与数据库的交付次数提高了响应速度。
假如有一条数据的查询量非常大，且内容基本不变，反复查询就会让数据库压力变大，这时我们就可以将数据存在内存缓存中，这样就大大提高的了查询效率，同时缓解了数据库压力。
## 1.2 MyBatis的缓存机制
MyBatis提供了一级缓存和二级缓存

- 一级缓存：也称为本地缓存，用于保存用户在一次会话过程中查询的结果，用户一次会话中只能使用一个sqlSession，一级缓存是默认开启的。**在同一个 sqlSession 中两次执行相同的 sql 语句，第一次执行完毕会将数据库中查询的数据写到缓存（内存），第二次会从缓存中获取,从而提高查询效率**。当一个 sqlSession 结束后该 sqlSession 中的 一级缓存也就不存在了。
- 二级缓存：也称为全局缓存，是mapper级别的缓存，是针对一个表的查结果的存储，可以共享给所有针对这张表的查询的用户。**二级缓存是多个 SqlSession 共享的，其作用域是 mapper 的同一个 namespace，不同的 sqlSession 两次执行相同 namespace 下的 sql 语句且向 sql 中传递参数也相同，即最终执行相同的 sql 语句，第一次执行完毕会将数据库中查询的数据写到缓存（内存），第二次会从缓存中获取数据，而不再从数据库查询，从而提高查询效率。**Mybatis默认没有开启二级缓存，需要手动配置。
那么关于一级缓存和二级缓存有哪些特点呢？我们继续往下看。
## 2. MyBatis的一级缓存
一级缓存区域是根据 SqlSession 为单位划分的。 每次查询会先从缓存区域找，如果找不到从数据库查询，查询到数据将数据写入缓存。 Mybatis 内部存储缓存使用一个 HashMap，key 为hashCode+sqlId+Sql 语句。value 为从查询出来映射生成的 java 对象。当sqlSession 执行 insert、update、delete 等操作并 commit 提交后，会清空缓存区域，此时缓存失效。
**2.1 一级缓存分析**
我们先编写一段测试代码并观察执行结果：

| **@Test public void test001(){ SqlSession sqlSession = MyBatisTools.getSqlSession(); UsersMapper usersMapper = sqlSession.getMapper(UsersMapper.class); Users users1 = usersMapper.getUserById(1); System.out.println("Users1=="+users1); Users users2 = usersMapper.getUserById(1); System.out.println("Users2=="+users2); System.out.println("users1与users2是否相等："+(users1==users2)); sqlSession.close(); }** |
| --- |

执行结果如下图：
![](https://raw.githubusercontent.com/danmuking/image/main/f9a6936f1516ce7c21e7920b715a76b5.png)
可以观察到上图所示结果：两次查询中，只执行了一次SQL，并且通过一级缓存查询的对象是相等的，因为第二次查询结果是从一级缓存中取出。
**2.2 一级缓存失效的原因**

- **原因一：使用不同的SqlSession对象导致无法看到一级缓存工作。**
测试代码：
| **@Test public void test002(){ SqlSession sqlSession = MyBatisTools.getSqlSession(); UsersMapper usersMapper = sqlSession.getMapper(UsersMapper.class); Users users1 = usersMapper.getUserById(1); System.out.println("Users1=="+users1); //新建了一个sqlSession SqlSession sqlSession2 = MyBatisTools.getSqlSession(); UsersMapper usersMapper2 = sqlSession2.getMapper(UsersMapper.class); Users users2 = usersMapper2.getUserById(1); System.out.println("Users2=="+users2); System.out.println("users1与users2是否相等："+(users1==users2)); sqlSession.close(); sqlSession2.close(); }** |
| --- |

可以看到执行结果获取的两个对象已经不相等，一级缓存已经失效。
![](https://raw.githubusercontent.com/danmuking/image/main/f162e91d595b6edaf0fbd77eca30f636.png)

- **原因二：在一个SqlSession使用相同条件，但是，此时在查询之间进行数据修改操作会导致一级缓存失效。**
| **@Test public void test003(){ SqlSession sqlSession = MyBatisTools.getSqlSession(); UsersMapper usersMapper = sqlSession.getMapper(UsersMapper.class); Users users1 = usersMapper.getUserById(1); System.out.println("Users1=="+users1); //两次查询中间执行数据增加、修改或删除操作 Users users = new Users(); users.setName("Lisa"); users.setPwd("123456"); usersMapper.addUser(users); Users users2 = usersMapper.getUserById(1); System.out.println("Users2=="+users2); System.out.println("users1与users2是否相等："+(users1==users2)); sqlSession.commit(); sqlSession.close(); }** |
| --- |

执行结果如下：
![](https://raw.githubusercontent.com/danmuking/image/main/d5cd96ee8eaa73f7cfcf6b24b9a611f2.webp)

- **原因三：在一个SqlSession使用相同查询条件此时手动刷新缓存时导致一级缓存失败。**
| **@Test public void test004(){ SqlSession sqlSession = MyBatisTools.getSqlSession(); UsersMapper usersMapper = sqlSession.getMapper(UsersMapper.class); Users users1 = usersMapper.getUserById(1); System.out.println("Users1=="+users1); //手动刷新用户一级缓存,导致用户一级缓存原有的内容消失掉 sqlSession.clearCache(); Users users2 = usersMapper.getUserById(1); System.out.println("Users2=="+users2); System.out.println("users1与users2是否相等："+(users1==users2)); sqlSession.close(); }** |
| --- |

![](https://raw.githubusercontent.com/danmuking/image/main/12c301d4598f7e9bc8a0fc15398ede52.png)
## **3 MyBatis的二级缓存**
**3.1 如何开启二级缓存**
①在mybatis核心配置文件中加入以下代码

| **<setting name="cacheEnabled" value="true"/>** |
| --- |

②然后在对应的mapper.xml里面加入配置

| **<cache eviction="FIFO" flushInterval="6000" readOnly="true" size="50"></cache>** |
| --- |

其中，cache标签的属性介绍如下：

| **属性** | **说明** |
| --- | --- |
| type | cache使用的类型，默认是PerpetualCache |
| eviction | 定义回收的策略，常见的有FIFO，LRU。LRU（默认）：最近最少使用，移除最长时间不被使用的对象FIFO：先进先出，安对象进入缓存的顺序来移除它们 SOFT：软引用，移除基于垃圾回收器的状态和软引用规则的对象 WEAK: 弱引用，更积极的移除基于垃圾收集器状态和弱引用规则的对象 |
| flushInterval | 配置一定时间自动刷新缓存，单位是毫秒 |
| size | 最多缓存对象的个数 |
| readOnly | 是否只读，若配置可读写，则需要对应的实体类能够序列化。 |
| blocking | 若缓存中找不到对应的key，是否会一直blocking，直到有对应的数据进入缓存 |

③二级缓存测试代码：

| **@Test public void test005(){ SqlSession sqlSession = MyBatisTools.getSqlSession(); UsersMapper usersMapper = sqlSession.getMapper(UsersMapper.class); Users users1 = usersMapper.getUserById(1); System.out.println("Users1=="+users1); sqlSession.close(); //新建一个sqlSession SqlSession sqlSession2 = MyBatisTools.getSqlSession(); UsersMapper usersMapper2 = sqlSession2.getMapper(UsersMapper.class); Users users2 = usersMapper2.getUserById(1); System.out.println("Users2=="+users2); sqlSession2.close(); System.out.println("users1与users2是否相等："+(users1==users2)); }** |
| --- |

执行结果：只执行了一次SQL，对象相等。
![](https://raw.githubusercontent.com/danmuking/image/main/8225d51c75f092f5a3be12f7976c14a3.png)
**3.2 cache标签的readOnly**
当将cache的readOnly属性改成false

| **<cache eviction="FIFO" flushInterval="6000" readOnly="true" size="50"></cache>** |
| --- |

需要让Users类实现序列化接口

| **@Datapublic class Users implements Serializable{ private Integer id; private String name; private String pwd;}** |
| --- |

再次运行3.1的测试test005()，得到如下结果：可以看到运行成功，并且只执行了一次SQL语句，第二次查询缓存命中。但是查询出来的两个对象却不相等了。
![](https://raw.githubusercontent.com/danmuking/image/main/deb68dc37ca2b681c8ccf238efaeacf5.png)
这是由于readOnly属性的两个值，导致从缓存中获取数据的方式不一致导致：
true：只读，mybatis认为所有从缓存中获取数据的操作都是只读操作，不会修改数据。 mybatis为了加快获取速度，直接会将数据在缓存中的引用交给用户，不安全，但速度快。
false：非只读，mybatis觉得获取的数据可能会被修改 mybatis会利用序列化&反序列化的技术克隆一份新的数据给你，安全，但速度慢。
所以，当我们将readOnly改为false之后，Mybatis会用到序列化&反序列化技术克隆一份新的数据，所以对应的pojo需要实现序列化。 并且是复制一份新的数据给到用户，所以两次查询的对象并不相等了
**3.3 二级缓存失效的原因**

- **原因一：二级缓存进行增删改操作也会刷新二级缓存，导致二级缓存失效。**
| **@Test public void test006(){ SqlSession sqlSession = MyBatisTools.getSqlSession(); UsersMapper usersMapper = sqlSession.getMapper(UsersMapper.class); Users users1 = usersMapper.getUserById(1); System.out.println("Users1=="+users1); sqlSession.close(); SqlSession sqlSession3 = MyBatisTools.getSqlSession(); UsersMapper usersMapper3 = sqlSession3.getMapper(UsersMapper.class); Users users = new Users(); users.setName("Lily"); users.setPwd("123456"); usersMapper3.addUser(users); sqlSession3.commit(); sqlSession3.close(); //新建一个sqlSession SqlSession sqlSession2 = MyBatisTools.getSqlSession(); UsersMapper usersMapper2 = sqlSession2.getMapper(UsersMapper.class); Users users2 = usersMapper2.getUserById(1); System.out.println("Users2=="+users2); sqlSession2.close(); System.out.println("users1与users2是否相等："+(users1==users2)); }** |
| --- |

测试结果：可以看到如下图执行了两次SQL，说明二级缓存失效
![](https://raw.githubusercontent.com/danmuking/image/main/ba9b1d057ea0b1e51f876ff0b28a8a0a.png)

- 原因二：使用useCache 设置禁用二级缓存。在 statement 中设置 useCache=“false”，可以禁用当前 select 语句的二级缓存，即每次都会去数据库查询。例如
| **<select id="getUserById" parameterType="integer" resultType="net.togogo.pojo.Users" useCache="false"> select * from users where id=#{id}</select>** |
| --- |

- 原因三：使用flushCache=“true” 属性刷新了缓存。设置 statement 配置中的 flushCache=“true” 属性，默认情况下为 true，即刷新缓存，一般执行完 commit 操作都需要刷新缓存，flushCache=“true” 表示刷新缓存，这样可以避免增删改操作而导致的脏读问题。
| **<select id="findAll" resultMap="userMap" useCache="false" flushCache="true"> select * from user u left join orders o on u.id = o.uid</select>** |
| --- |

