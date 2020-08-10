**首先我们常用的注解包括(\*@Entity、@Table、@Id、@IdClass、@GeneratedValue、@Basic、@Transient、@Column、@Temporal、@Enumerated、@Lob\*)**

1. **@Entity使用此注解定义的对象将会成为被JPA管理的实体，将映射到指定的数据库表@Entity(name =“user”)其中name默认是此实体类的名字，全局唯一。**
2. **@Table指定此实体类对应的数据库的表名。若注解不加名字则系统认为表名和实体类的名字相同**
3. **@Id定义字段为数据库的主键，一个实体里面必须有一个。**
4. **@IdClass利用外部类的联合主键，其中外部类必须满足一下几点要求**

- 必须实现Serializable接口。
- 必须有默认的public无参数的构造方法。
- 必须覆盖equals和hashCode方法。equals方法用于判断两个对象是否相同，EntityManger通过find方法来查找Entity时是根据equals的返回值来判断的。hashCode方法返回当前对象的哈希码，生成的hashCode相同的概率越小越好，算法可以进行优化。

1. **@GeneratedValue为主键生成策略**

   ```
   默认为AUTO即JPA自动选择合适的策略
   
   IDENTITY 适用于MySQL，策略为自增
   SEQUENCE 通过序列生成主键通过@SquenceGenerator指定序列名MySQL不支持
   TABLE 框架由表模拟产生主键，使用该策略有利于数据库移植
   12345
   ```

2. **@Basic表示此字段是映射到数据库，如果实体字段上没有任何注解默认为@Basic。其中可选参数为@Basic(fetch =FetchType.LAZY, optional =false)其中fetch默认为EAGER立即加载，LAZY为延迟加载、optional表示该字段是否可以为null**

3. **@Transient和@Basic的作用相反，表示该字段不是一个到数据库表的字段映射，JPA映射数据库的时候忽略此字段。**

4. **@Column定义实体内字段对应的数据库中的列名**

```java
@Column(name = "real_name", unique = true, nullable = false, insertable = false, updatable = false, columnDefinition = "varchar", length = 100)
1
```

- name对应数据库的字段名，可选默认字段名和实体属性名一样
- unique是否唯一，默认false，可选
- nullable是否允许为空。可选，默认为true
- insertable执行insert的时候是否包含此字段。可选，默认true
- updatable执行update的时候是否包含此字段。可选，默认true
- columnDefinition表示该字段在数据库中的实际类型
- length数据库字段的长度，可选，默认25

1. **@Temporal用来设置Date类型的属性映射到对应精度的字段**

```java
@Temporal(TemporalType.DATE)    //映射为只有日期
@Temporal(TemporalType.TIME)    //映射为只有时间
@Temporal(TemporalType.TIMESTAMP)  //映射为日期+时间
123
```

1. **@Lob将字段映射成数据库支持的大对象类型,支持一下两种数据库类型的字段。（注意：Clob、Blob占用的内存空间较大，一般配合@Basic(fetch
   = FetchType.LAZY)将其设置为延迟加载）**

- Clob：字段类型为Character[]、char[]、String将被映射为Clob
- Blob：字段类型为Byte[]、byte[]和实现了Serializable接口的类型将被映射为Blob类型



## 接下来介绍关联关系注解（@JoinColumn、@OneToOne、@OneToMany、@ManyToOne、@ManyToMany、@JoinTable、@OrderBy）

- **@JoinColumn定义外键关联字段名称，其中属性意义如下**
  - name表示目标表的字段名，必填
  - referencedColumnName本实体表的字段名，非必填，默认是本表的ID
  - unique外键字段是否唯一，可选，默认false
  - nullable外键字段是否允许为空，可选，默认true
  - insertable新增操作的时候是否跟随一起新增，可选，默认true
  - updatable更新时候是否一起更新。可选，默认true
  - @JoinColumn主要配合@OneToOne、@ManyToOne、@OneToMany一起使用，单独使用没有任何意义。
  - @JoinColumns定义多个字段的关联关系

**2. @OneToOne一对一关联关系**

```java
@OneToOne(targetEntity = SysRole.class, cascade = {CascadeType.PERSIST,CascadeType.REMOVE,CascadeType.REFRESH,CascadeType.MERGE},fetch = FetchType.LAZY,optional = false,mappedBy = "userId",orphanRemoval = true)
1
```

- targetEntity非必填，默认为该字段的类型
- cascade级联操作策略CascadeType.PERSIST级联新建，CascadeType.REMOVE级联删除，CascadeType.REFRESH级联刷新，CascadeType.MERGE级联更新，CascadeType.ALL四项全选
- fetch数据加载方式，默认EAGER（立即加载），LAZY（延迟加载）
- optional是否允许为空。可选，默认为true
- mappedBy关联关系被谁维护，非必填，一般不需要特别指定。注意：只有关系维护方才能操作两者的关系，被维护方即使设置了维护方的属性进行存储也不会更新外键关联。mappedBy不能与@JoinColumn或者@JoinTable同时使用。mappedBy的值指的是另一方的实体里面属性的字段，而不是数据库字段，也不是实体对象的名字，即另一方配置了@JoinColumn或者@JoinTable注解的属性的字段名称
- orphanRemoval是否级联删除，和CascadeType.REMOVE效果一样，两种只要配置一种就会自动级联删除

@OneToOne需要配合@JoinColumn一起使用，可以双向关联，也可以只配置一方。下面我们举一个例子：假设一个用户只拥有一个角色SysUser如下

```java
@OneToOne
@JoinColumn(name = "role_id",referencedColumnName = "user_id")
private SysRole role;
//若需要双向关联则SysRole的内容如下
@OneToOne(mappedBy = "role")
private SysUser user;
//当然也可以不用选择mappedBy，使用下面效果也一样
@OneToOne
@JoinColumn(name = "user_id",referencedColumnName = "role_id")
private SysUser user;
12345678910
```

**3.@OneToMany和@ManyToOne一对多和多对一的关联关系**

```java
@Entity
@Table(name="user")
class User implements Serializable{
    private Long userId;
    @OneToMany(cascade = CascadeType.ALL,fetch = FetchType.LAZY,mappedBy = "user")
    private Set<Role> setRole;
}

@Entity
@Table(name="role")
class Role{
    private Long roleId;
    @ManyToOne(cascade = CascadeType.ALL,fetch = FetchType.EAGER)
    @JoinColumn(name = "user_id")//user_id字段作为外键
    private User user;
}
12345678910111213141516
```

**4.@OrderBy关联查询的时候排序，一般和@OneToMany一起使用**

```java
@Entity
@Table(name="user")
class User implements Serializable{
    private Long userId;
    @OneToMany(cascade = CascadeType.ALL,fetch = FetchType.LAZY,mappedBy = "user")
    @OrderBy("role_name DESC")
    private Set<Role> setRole;
}
12345678
```

**5.@JoinTable关联关系表一般和@ManyToMany一起使用，@ManyToMany表示多对多，也有单双向之分，单双向和注解无关，只看实体类之间是否相互引用**

```java
@JoinTable(name = "sys_user_role",
		joinColumns = {@JoinColumn(name = "user_id", referencedColumnName = "id")},
        inverseJoinColumns = {@JoinColumn(name = "role_id", referencedColumnName = "id")})
@ManyToMany(cascade = {CascadeType.REFRESH}, fetch = FetchType.EAGER)
private List<SysRole> roles;
12345
```

- @JoinTable中name表示中间关联关系表名
- @JoinTable中的joinColumns表示主链接表的字段，即当前对象内对应的连接字段
- @JoinTable中inverseJoinColumns表示被连接的表的外键字段

**6.Left Join、Inner join和@EntityGraph**

当使用@ManyToMany、@ManytoOne、@OneToMany、@OneToOne关联关系的时候SQL执行查询的时候总是一条主查询语句和N条子查询语句组成，运行的效率较地下，如果子对象有N个就会执行N+1条SQL，JPA2.1推出的@EntityGraph、@NamedEntityGraph用来提高查询效率@NamedEntityGraph配置在@Entity上面，而@EntityGraph配置在Repository的查询方法上面

```java
@NamedEntityGraph(name = "User.addressEntityList",attributeNodes = {@NamedAttributeNode("setRole"),@NamedAttributeNode("dept")})
@Entity
@Table(name="user")
public class User implements Serializable{
    private Long userId;
    @OneToMany(cascade = CascadeType.ALL,fetch = FetchType.LAZY,mappedBy = "user")
    @OrderBy("role_name DESC")
    private Set<Role> setRole;

    @OneToOne
    @JoinColumn(name = "dept_id",referencedColumnName = "user_id")
    private Department dept;
}
12345678910111213
```

然后只需要在Repository查询方法上面加上@EntityGraph注解即可，其中value就是@NamedEntityGraph中的Name，配置如下

```java
@EntityGraph(value ="User.addressEntityList")
List<User> findAll();
12
```

**对于关系查询需要注意以下事项：**

- 所有的注解要么全部配置在字段上，要么全部配置在get方法上面，不能混用，混用项目就会启动不起来
- 所有关联都支持单双向关联。当JSON序列化的时候使用双向注解会产生死循环，需要人为手动转换一次，或者使用@JsonIgnore
- 所有的关联表一般不需要建立外键索引