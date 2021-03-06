<!-- tags:[testcontainers, m1] -->

# ๐ป m1 + mysql8 + testcontainers
backend ๊ฐ๋ฐ์ ํ๋ฉฐ test ํ๊ฒฝ์ ์ค์ ํ๋ ๊ฒ์ ํ์์ ์ธ ๊ณผ์ ์ด๋ค.

docker๊ฐ ๋์ค๊ณ  ๋์๋ถํฐ๋ ํธํ๊ฒ ์ค์ ํ๊ธฐ ์ํด docker-compose๋ฅผ ํ์ฉํ๊ฑฐ๋ testcontainers๋ฅผ ์ฃผ๋ก ํ์ฉํ๋๋ฐ,
๊ฐ์ธ ๋ธํธ๋ถ์ m1์ผ๋ก ๋ฐ๊พธ๊ณ  ๋์ ์ฒ์์ผ๋ก testcontainers๋ฅผ mysql๋ก ์ค์ ํ๋ฉฐ ์๊ธด ์ด์๋ฅผ ์ ๋ฆฌํด๋ณด์๋ค.

## ๐จโ๐ป ๊ฒฐ๊ณผ
์๊ฐ์ด ์์ผ์  ๋ถ๋ค์ ์ํด ๋ฐ๋ก ์ฑ๊ณตํ ์ฝ๋๋ถํฐ..

๋ณดํต์ testContainers๋ฅผ ์ฌ์ฉํ ๋์ [junit container ๋ฐฉ์](https://www.testcontainers.org/test_framework_integration/junit_5/)์ผ๋ก ๊ตฌํํ๋๋ฐ
๋๋ ๊ฐ์ธ์ ์ผ๋ก test์ abstract ํด๋์ค๋ ๋๋ฌด ๋ง์ ์ด๋ธํ์ด์์ ๋ฃ๋ ๊ฒ์ ์ ํธํ์ง ์๊ธฐ ๋๋ฌธ์ [๋ณ๋๋ก lifecycle์ ๊ด๋ฆฌํ๋ ๋ฐฉ์](https://www.testcontainers.org/test_framework_integration/manual_lifecycle_control/)์ผ๋ก ๊ตฌํํ์๋ค.

```kotlin
import org.springframework.core.Ordered
import org.springframework.core.annotation.Order
import org.springframework.stereotype.Component
import org.testcontainers.containers.MySQLContainer
import org.testcontainers.utility.DockerImageName
import javax.annotation.PreDestroy

@Component
@Order(value = Ordered.HIGHEST_PRECEDENCE)
class MySqlTestContainer {

    @PreDestroy
    fun stop() {
        MY_SQL_CONTAINER.stop()
    }

    companion object {
        @JvmStatic
        val MY_SQL_CONTAINER: MySQLContainer<*> =
                // image for linux/arm64/v8 m1 support
                DockerImageName.parse("mysql/mysql-server:8.0.26")
                        .asCompatibleSubstituteFor("mysql")
                        .let { compatibleImageName -> MySQLContainer<Nothing>(compatibleImageName) }
                        .apply {
                            withDatabaseName(DATABASE_NAME)
                            withUsername(USERNAME)
                            withPassword(PASSWORD)
                            withEnv("MYSQL_USER", USERNAME)
                            withEnv("MYSQL_PASSWORD", PASSWORD)
                            withEnv("MYSQL_ROOT_PASSWORD", PASSWORD)
                            start()
                        }

        const val DATABASE_NAME: String = "batch_test"
        const val USERNAME: String = "root"
        const val PASSWORD: String = "password"
    }
}
```

์๋๋ ์์ ์ปจํ์ด๋๋ฅผ dataSource๋ก ์ฐ๊ฒฐํ๋ ๋ถ๋ถ์ด๋ค.
```kotlin
import org.springframework.boot.jdbc.DataSourceBuilder
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.context.annotation.DependsOn
import org.testcontainers.containers.MySQLContainer
import javax.sql.DataSource

@Configuration
class TestDataSourceConfiguration {

    @Bean
    @DependsOn("mySqlTestContainer")
    fun dataSource(): DataSource =
            DataSourceBuilder.create()
                    .url("jdbc:mysql://localhost:" +
                            "${MySqlTestContainer.MY_SQL_CONTAINER.getMappedPort(MySQLContainer.MYSQL_PORT)}/" +
                            MySqlTestContainer.DATABASE_NAME)
                    .driverClassName("com.mysql.cj.jdbc.Driver")
                    .username(MySqlTestContainer.USERNAME)
                    .password(MySqlTestContainer.PASSWORD)
                    .build()

}

```

## ๐จ ๋ง๋ฅ๋จ๋ฆฐ ์๋ฌ๋ค
### ๐ง mysql docker hub m1 ๋ฏธ์ง์ ์ด์
```
no matching manifest for linux/arm64/v8 in the manifest list entries
```
๋จผ์  `MySQLContainer<Nothing>("mysql:8.0.26")` ๋ฒ์ ์ผ๋ก ์๋ํ๋ ์์ ๊ฐ์ ์๋ฌ๋ฅผ ๋ง๋ฅ๋จ๋ ธ๋ค.
์ฐพ์๋ณด๋ [mysql dockerhub](https://hub.docker.com/_/mysql)์ linux/amd64๋ฒ์ ๋ง ์ฌ๋ผ์์๊ณ  m1์ ํด๋นํ๋ linux/arm64/v8์ผ๋ก ๋น๋๋ ๋ฒ์ ์ด ์์๋ค.
๋ฐ๋ผ์ ์ฐพ์๋ณด๋ [mysql-server dockerhub](https://hub.docker.com/r/mysql/mysql-server)์๋ m1์ ํด๋นํ๋ ๋ฒ์ ๋ ์ฌ๋ผ์์์ด์ ํด๋น ๋ฒ์ ์ผ๋ก ๋ณ๊ฒฝํ์๋ค.

### ๐ง testcontainers ์ด๋ฏธ์ง ํธํ ์ด์
```
java.lang.IllegalStateException: Failed to verify that image 'mysql/mysql-server:8.0.26' is a compatible substitute for 'mysql'. This generally means that you are trying to use an image that Testcontainers has not been designed to use. If this is deliberate, and if you are confident that the image is compatible, you should declare compatibility in code using the `asCompatibleSubstituteFor` method. For example:
   DockerImageName myImage = DockerImageName.parse("mysql/mysql-server:8.0.26").asCompatibleSubstituteFor("mysql");
```
์๋ฌด๋๋ docker image๋ฅผ mysql์์ mysql-server๋ก ๋ฐ๊พธ๋ ์ ๋ง testcontainers ๋ผ์ด๋ธ๋ฌ๋ฆฌ๊ฐ ์ฌ์ฉํ๋ mysql์ด ๋ง๋์ง๋ฅผ ๋ฌผ์ด๋ณด๋ ์๋ฌ๊ฐ ๋ฐ์ํ๋ค.
๋ฐ๋ผ์ ์๋ฌ์์ ์ค๋ชํ๋ ๋๋ก ์ค์ ์ ์์ ํ์๋ค.
```kotlin
DockerImageName.parse("mysql/mysql-server:8.0.26")
                        .asCompatibleSubstituteFor("mysql")
                        .let { compatibleImageName -> MySQLContainer<Nothing>(compatibleImageName) }
```

---
### ๐ง mysql8 docker ์ค์  ์ด์
```
java.sql.SQLException: null,  message from server: "Host '172.17.0.1' is not allowed to connect to this MySQL server"
```
๊ธฐ์กด์๋ ์ฃผ๋ก postgresql์ ํ์ฉํด์ ์ธ์ง mysql8์์ ํ์ํ ์ต์๋ค์ ์กฐ๊ธ ๋์ณค๋ ๊ฒ ๊ฐ๋ค.
mysql 8 docker ์ค์ ๋ค์ ๊ตฌ๊ธ๋งํด์ env๋ฅผ ์ถ๊ฐํด์ ์ต์ข์ ์ผ๋ก ์ฐ๋์ ์ฑ๊ณตํ์๋ค.
```kotlin
    companion object {
        @JvmStatic
        val MY_SQL_CONTAINER: MySQLContainer<*> =
                // image for linux/arm64/v8 m1 support
                DockerImageName.parse("mysql/mysql-server:8.0.26")
                        .asCompatibleSubstituteFor("mysql")
                        .let { compatibleImageName -> MySQLContainer<Nothing>(compatibleImageName) }
                        .apply {
                            withDatabaseName(DATABASE_NAME)
                            withUsername(USERNAME)
                            withPassword(PASSWORD)
                            withEnv("MYSQL_USER", USERNAME)
                            withEnv("MYSQL_PASSWORD", PASSWORD)
                            withEnv("MYSQL_ROOT_PASSWORD", PASSWORD)
                            start()
                        }

        const val DATABASE_NAME: String = "batch_test"
        const val USERNAME: String = "root"
        const val PASSWORD: String = "password"
    }

```

## ๐ ๊ฒฐ๋ก 
m1์ ์ฐ๋ค๋ณด๋ ์คํ์๋๋ ๋ฐฐํฐ๋ฆฌ ์๊ฐ ๋ฑ ์ป์ ์ด๋์ด ํฌ์ง๋ง ๋ฐ๋๋ก ์์ง ์ํ๊ณ๊ฐ ์กฐ๊ธ ๋ถ์กฑํ ๋ถ๋ถ๋ค์ด ๋จ์ ์ธ ๊ฒ ๊ฐ๋ค. ํ์ง๋ง ๋ ๊ทธ๋ ๋ฏ์ด ํด๊ฒฐ๋ฐฉ์์ ์๊ธฐ ๋ง๋ จ์ด๋ค. **๋!**
