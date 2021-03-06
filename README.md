# Spring Data Jpa Audit Example [![Build Status](https://travis-ci.org/rashidi/spring-boot-data-audit.svg?branch=master)](https://travis-ci.org/rashidi/spring-boot-data-audit)
Enable auditing with Spring Data Jpa's `@CreatedDate` and `@LastModified`

## Background
[Spring Data Jpa][1] provides auditing feature which includes `@CreateDate`, `@CreatedBy`, `@LastModifiedDate`, 
and `@LastModifiedBy`. In this example we will see how it can be implemented with very little configurations.

## Entity Class
In this example we have an entity class, [User][2] which contains information about the table structure. Initial 
structure is as follows:

```java
@Entity
@Table
public class User {
    
    private Long id;
    private String name;
    private String username;
    private ZonedDateTime created;
    private ZonedDateTime modified;
    
    @Id
    @GeneratedValue
    public Long getId() {
        return id;
    }
    
    public User setId(Long id) {
        this.id = id;
        return this;
    }
    
    @Column(nullable = false)
    public String getName() {
        return name;
    }
    
    public User setName(String name) {
        Assert.hasText(name, "name is required");

        this.name = name;
        return this;
    }
    
    @Column
    public String getUsername() {
        return username;
    }
    
    public User setUsername(String username) {
        Assert.hasText(username, "username is required");

        this.username = username;
        return this;
    }
    
    @Column(nullable = false, updatable = false)
    public ZonedDateTime getCreated() {
        return created;
    }
    
    public User setCreated(ZonedDateTime created) {
        this.created = created;
        return this;
    }
    
    @Column(nullable = false)
    public ZonedDateTime getModified() {
        return modified;
    }
    
    public User setModified(ZonedDateTime modified) {
        this.modified = modified;
        return this;
    }
}
```

As you can see it is a standard implementation of `@Entity` JPA class. We would like to keep track when an entry is 
created with `created` column and when it is modified with `modified` column.

## Enable JpaAudit
In order to enable [JPA Auditing][3] for this project will need to apply three annotations and a configuration class.
Those annotations are; `@EntityListener`, `@CreatedDate`, and `@LastModifiedDate`.

`@EntityListener` will be the one that is responsible to listen to any create or update activity. It requires 
`Listeners` to be defined. In this example we will use the default class, `EntityListeners`.

By annotating a column with `@CreatedDate` we will inform Spring that we need this column to have information on 
when the entity is created. While `@LastModifiedDate` column will be defaulted to `@CreatedDate` and will be updated
to the current time when the entry is updated.

The final look of `User` class:

```java
@Entity
@EntityListeners(AuditingEntityListener.class)
@Table
public class User {
    
    private Long id;
    private String name;
    private String username;
    private ZonedDateTime created;
    private ZonedDateTime modified;
    
    @Id
    @GeneratedValue
    public Long getId() {
        return id;
    }
    
    public User setId(Long id) {
        this.id = id;
        return this;
    }
    
    @Column(nullable = false)
    public String getName() {
        return name;
    }
    
    public User setName(String name) {
        Assert.hasText(name, "name is required");

        this.name = name;
        return this;
    }
    
    @Column
    public String getUsername() {
        return username;
    }
    
    public User setUsername(String username) {
        Assert.hasText(username, "username is required");

        this.username = username;
        return this;
    }
    
    @CreatedDate
    @Column(nullable = false, updatable = false)
    public ZonedDateTime getCreated() {
        return created;
    }
    
    public User setCreated(ZonedDateTime created) {
        this.created = created;
        return this;
    }
    
    @LastModifiedDate
    @Column(nullable = false)
    public ZonedDateTime getModified() {
        return modified;
    }
    
    public User setModified(ZonedDateTime modified) {
        this.modified = modified;
        return this;
    }
}
```

As you can see `Person` is now annotated with `@EntityListeners` while `created` and `modified` columns are annotated
with `@CreatedDate` and `@LastModifiedDate`. Next we will need to create a `Configuration` class to enable JpaAuditing.

In this project we have [AuditConfiguration][4] class which is responsible to inform Spring Data that we would like
to enable Auditing. This can be achieved with a simple annotation, `@EnableJpaAuditing`

```java
@Configuration
@EnableJpaAuditing
public class AuditConfiguration {
}
```

That's it! Our application has JPA Auditing feature enabled. The result can be seen in [SpringDataAuditApplicationTests][5].

## Verify Audit Implementation
There is no better way to verify an implementation other than running some tests. In our test class we have to scenario:

  - Create an entity which will have `created` and `modified` fields has values without us assigning them
  - Update created entity and `created` field will remain to have the same value while `modified` values will be updated
  
### Create an entity
In the following test we will see that values for `created` and `modified` are assigned by Spring itself:

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class SpringDataAuditApplicationTests {
    
    @Autowired
    private UserRepository userRepository;
    
    private User user;
    
    @Before
    public void create() {
        user = userRepository.save(
            new User().setName("Rashidi Zin").setUsername("rashidi.zin")
        );
        
        assertThat(user.getCreated())
            .isNotNull();
        
        assertThat(user.getModified())
            .isNotNull();
    }
    
    // rest of the content is omitted
}
```

As mentioned earlier, we did not assign values for `created` and `modified` fields but Spring will assign them for us.
Same goes with when we are updating an entry.

### Update an entity
In the following test we will change the `username` without changing `modified` field. We will expect that `modified`
field will have a recent time as compare to when it was created:

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class SpringDataAuditApplicationTests {
    
    @Autowired
    private UserRepository userRepository;
    
    private User user;
    
    @Test
    public void update() {
        ZonedDateTime created = user.getCreated();
        ZonedDateTime modified = user.getModified();
        
        userRepository.save(
            new User()
                .setId(user.getId())
                .setName(user.getName())
                .setUsername("rashidi")
        );
        
        User updatedUser = userRepository.findOne(user.getId());
        
        assertThat(updatedUser.getUsername())
            .isEqualTo("rashidi");
        
        assertThat(updatedUser.getCreated())
            .isEqualTo(created);
        
        assertThat(updatedUser.getModified())
            .isGreaterThan(modified);
    }
}
```

As you can see at our final verification we assert that `modified` field should have a greater value than it 
previously had.

## Conclusion
To recap. All we need in order to enable JPA auditing feature in this project are:

  - `@EnableJpaAuditing`
  - `@EntityListeners`
  - `@CreatedDate`
  - `@LastModifiedDate`

[1]: http://docs.spring.io/spring-data/jpa/docs/current/reference/html/
[2]: src/main/java/my/zin/rashidi/demo/data/audit/domain/User.java
[3]: http://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.auditing
[4]: src/main/java/my/zin/rashidi/demo/data/audit/configuration/AuditConfiguration.java
[5]: src/test/java/my/zin/rashidi/demo/data/audit/SpringDataAuditApplicationTests.java
