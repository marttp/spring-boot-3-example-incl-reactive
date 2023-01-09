# spring-boot-webflux-entity 
ตัวอย่างการเขียน Spring-boot WebFlux Entity 

# 1. เพิ่ม Dependencies

pom.xml 
``` xml
...
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.1.5.RELEASE</version>
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webflux</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <dependency>
        <groupId>org.hibernate</groupId>
        <artifactId>hibernate-core</artifactId>
        <version>5.4.3.Final</version>
    </dependency>

    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <version>42.2.5</version>
    </dependency>

    <dependency>
        <groupId>com.zaxxer</groupId>
        <artifactId>HikariCP</artifactId>
        <version>3.3.1</version>
    </dependency>

    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <scope>provided</scope>
    </dependency>
</dependencies>

...
```
### dependencies
- `spring-boot-starter-webflux` ใช้สำหรับเขียน webflux   
- `spring-boot-starter-data-jpa` ไว้สำหรับเขียนคำสั่ง query, method query   
- `hibernate-core`  สำหรับทำ ORM (Object Relational Mapping) ไว้เขียนพวก entity class สำหรับ mapping java class ไปยัง database table รวมถึงการ mapping พวก relation ต่าง ๆ ของ table เช่น One to One, One to Many, Many to Many 
- `postgresql` เป็น postgresql database driver  
- `HikariCP` เป็นตัวจัดการ database connection pool  
- `lombok` เป็น annotation code generator สามารถ generate code at compile time ได้ ทำให้เราไม่ต้องเขียน code บางส่วนเอง เช่น getter setter method ตัว lombox จะทำให้   

# 2. เขียน Main Class 

AppStarter.java  
``` java
@SpringBootApplication
@ComponentScan(basePackages = {"com.pamarin"}) 
public class AppStarter {

    public static void main(String[] args) {
        SpringApplication.run(AppStarter.class, args);
    }

}
```

# 3. เขียน Entity

User.java  
```java
@Data
@Entity
@Table(name = User.TABLE_NAME)
public class User implements Serializable {

    public static final String TABLE_NAME = "user";

    @Id
    private String id;

    @Column(name = "username", nullable = false, unique = true)
    private String username;

    @Column(name = "password", nullable = false)
    private String password;

    @OneToMany(fetch = FetchType.LAZY, mappedBy = "user")
    private List<UserAuthority> userAuthorities;

    public List<UserAuthority> getUserAuthorities() {
        if (userAuthorities == null) {
            userAuthorities = new ArrayList<>();
        }
        return userAuthorities;
    }

}
```
Authority.java  
```java 
@Data
@Entity
@Table(name = Authority.TABLE_NAME)
public class Authority implements Serializable {

    public static final String TABLE_NAME = "authority";

    @Id
    private String id;

    @Column(name = "name", unique = true, nullable = false)
    private String name;

    private String description;

    @OneToMany(fetch = FetchType.LAZY, mappedBy = "authority")
    private List<UserAuthority> userAuthorities;

    public List<UserAuthority> getUserAuthorities() {
        if (userAuthorities == null) {
            userAuthorities = new ArrayList<>();
        }
        return userAuthorities;
    }

}
```

UserAuthority.java  
```java
@Data
@Entity
@Table(name = UserAuthority.TABLE_NAME)
public class UserAuthority implements Serializable {

    public static final String TABLE_NAME = "user_authority";

    @Data
    @Embeddable
    public static class UserAuthorityPK implements Serializable {

        @Column(name = "user_id")
        private String userId;

        @Column(name = "authority_id")
        private String authorityId;

    }

    @EmbeddedId
    private UserAuthorityPK id;

    @ManyToOne
    @JoinColumn(name = "user_id", referencedColumnName = "id", insertable = false, updatable = false)
    private User user;

    @ManyToOne
    @JoinColumn(name = "authority_id", referencedColumnName = "id", insertable = false, updatable = false)
    private Authority authority;

}
```

### วิธีการเขียน Entity Class 
1. New java class ขึ้นมา 
2. Implements `java.io.Serializable`
3. เขียน Attributes / Properties / Columns ตามที่ต้องการ ด้วย `@Column` (อย่าลืมกำหนด Constrains)
4. ใส่ annotation `@Entity` 
5. ใส่ Annotation `@Table` เพื่อ map ไปยัง table ที่ต้องการ (name = ???)
6. กำหนด Id หรือ Primary Key ของ Table ด้วย `@Id`
    - กรณีที่เป็น Composite Key (Key ร่วม) จะใช้ `@EmbeddedId` และ Key class จะใส่ `@Embeddable`  
7. ประกาศ Getter / Setter / HashCode / Equals method ซึ่งสามารถใช้ `@Data` ของ lombox ช่วยได้  (มันจะ Generate ให้ Auto)  
8. เขียน Relation 
    - `@OneToOne` คือ class นี้ map ไปยัง attribute ที่ประกาศไว้แบบ 1:1 
    - `@OneToMany` คือ class นี้ map ไปยัง attribute ที่ประกาศไว้แบบ 1:M
    - `@ManyToOne` คือ class นี้ map ไปยัง attribute ที่ประกาศไว้แบบ M:1 
    - `@ManyToMany` คือ class นี้ map ไปยัง attribute ที่ประกาศไว้แบบ M:N    
     ซึ่งกรณีนี้ Hibernate จะสร้าง table ใหม่ขึ้นมาเชื่อมกลายเป็น 3 tables และมี relation เป็นแบบ   
    `TABLE_A` 1 <-> M `TABLE_C` N <-> 1 `TABLE_B`      
    (`TABLE_A` และ `TABLE_B` เป็น table ที่เรากำหนด ส่วน `TABLE_C` Hibernate จะ Generate ให้)    
    แต่วิธีนี้ผมไม่ค่อยจะใช้สักเท่าไหร่ เห็นข้อเสียของมันคือ Table กลาง (`TABLE_C`) เรา Custom เองไม่ได้ ผมจะใช้แค่  `@OneToMany` กับ `@ManyToOne` 
9. Join Columns (ถ้ามี)

# 4. Config application.properties
```properties
#------------------------------------ JPA --------------------------------------
spring.jpa.hibernate.ddl-auto=none
spring.jpa.properties.hibernate.cache.use_second_level_cache=false
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.hibernate.use-new-id-generator-mappings=true
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.proc.param_null_passing=true
spring.jpa.properties.hibernate.default_schema=*****

#------------------------------------ Hikari -----------------------------------
spring.datasource.hikari.minimumIdle=1
spring.datasource.hikari.maximumPoolSize=10
spring.datasource.hikari.idleTimeout=30000
spring.datasource.hikari.connectionTestQuery=SELECT 1 FROM DUAL
spring.datasource.hikari.validationTimeout=3000

#------------------------------------ Postgresql -------------------------------
spring.datasource.url=jdbc:postgresql:*****
spring.datasource.username=*****
spring.datasource.password=*****
spring.datasource.driver-class-name=org.postgresql.Driver
spring.datasource.platform=postgres
spring.datasource.type=org.postgresql.ds.PGSimpleDataSource
```

- `spring.jpa.hibernate.ddl-auto` เป็นการบอก Hibernate ว่าให้ทำคำสั่ง DDL (Data Definition Language) อะไร ตอน Start Application 
  - `none` คือ ไม่ต้องทำอะไร
  - `create` คือ ให้ทำการสร้าง table จาก entity ที่ประกาศไว้ ตอน start application  
  - `update` คือ ให้ทำการ update table ตาม entity ที่ประกาศไว้ ตอน start application  
  - `create-drop` คือ ให้ create และ drop table หลังจากเลิกใช้งาน จะใช้ตอนเขียน test คือ create ใช้งานเสร็จแล้ว drop ทิ้ง    
  - `validate` คือ ให้ทำการเช็ค database schema หรือ table ว่ามีการเปลี่ยนแปลงหรือไม่ ถ้ามีการเปลี่ยนแปลง จะ error ตอน start application    

ถ้าเป็นการเขียน entity ครั้งแรก (ตอน dev) ให้ใช้ `create` หลังจากนั้นแนะนำให้ใช้ `update` หรือ `none`    
ส่วน production ให้เป็น `none` แล้วใช้วิธี create database จาก sql script เอา ไม่ควรเปิด `create` หรือ `update` ตอนใช้งาน production *** ห้ามเด็ดขาด ***

# 5. Build
cd ไปที่ root ของ project จากนั้น  
``` shell 
$ mvn clean install
```

# 6. Run 
``` shell 
$ mvn spring-boot:run \
    -Dserver.port=8080 \
    -Dspring.datasource.url=jdbc:postgresql://<HOST>:<PORT>/<DATABASE_NAME>?sslmode=require \
    -Dspring.datasource.username=<DATABASE_USERNAME> \
    -Dspring.datasource.password=<DATABASE_PASSWORD> \
    -Dspring.jpa.properties.hibernate.default_schema=<DATABASE_SCHEMA>
```
ให้เปลี่ยน ค่า `<>` เป็นของตัวเองน่ะครับ
- HOST คือ ip หรือ domain name ของ database server 
- PORT คือ port ที่ใช้ 
- DATABASE_NAME คือ ชื่อ database 
- DATABASE_USERNAME คือ ชื่อ username ที่ login เข้าใช้งาน database 
- DATABASE_PASSWORD คือ รหัสผ่านที่คู่กับ username ที่ใช้ 
- DATABASE_SCHEMA คือ database schema ที่่ใช้ 

# ดูผลลัพธ์ที่ Console และ Database
