<!-- tags:[testcontainers, m1] -->

# 💻 m1 + mysql8 + testcontainers
backend 개발을 하며 test 환경을 설정하는 것은 필수적인 과정이다.

docker가 나오고 나서부터는 편하게 설정하기 위해 docker-compose를 활용하거나 testcontainers를 주로 활용하는데,
개인 노트북을 m1으로 바꾸고 나서 처음으로 testcontainers를 mysql로 설정하며 생긴 이슈를 정리해보았다.

## 👨‍💻 결과
시간이 없으신 분들을 위해 바로 성공한 코드부터..

보통은 testContainers를 사용할때에 [junit container 방식](https://www.testcontainers.org/test_framework_integration/junit_5/)으로 구현하는데
나는 개인적으로 test에 abstract 클래스나 너무 많은 어노테이션을 넣는 것을 선호하지 않기 때문에 [별도로 lifecycle을 관리하는 방식](https://www.testcontainers.org/test_framework_integration/manual_lifecycle_control/)으로 구현하였다.

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

아래는 위의 컨테이너를 dataSource로 연결하는 부분이다.
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

## 🚨 맞닥뜨린 에러들
### 🔧 mysql docker hub m1 미지원 이슈
```
no matching manifest for linux/arm64/v8 in the manifest list entries
```
먼저 `MySQLContainer<Nothing>("mysql:8.0.26")` 버전으로 시도하니 위와 같은 에러를 맞닥뜨렸다.
찾아보니 [mysql dockerhub](https://hub.docker.com/_/mysql)에 linux/amd64버전만 올라와있고 m1에 해당하는 linux/arm64/v8으로 빌드된 버전이 없었다.
따라서 찾아보니 [mysql-server dockerhub](https://hub.docker.com/r/mysql/mysql-server)에는 m1에 해당하는 버전도 올라와있어서 해당 버전으로 변경하였다.

### 🔧 testcontainers 이미지 호환 이슈
```
java.lang.IllegalStateException: Failed to verify that image 'mysql/mysql-server:8.0.26' is a compatible substitute for 'mysql'. This generally means that you are trying to use an image that Testcontainers has not been designed to use. If this is deliberate, and if you are confident that the image is compatible, you should declare compatibility in code using the `asCompatibleSubstituteFor` method. For example:
   DockerImageName myImage = DockerImageName.parse("mysql/mysql-server:8.0.26").asCompatibleSubstituteFor("mysql");
```
아무래도 docker image를 mysql에서 mysql-server로 바꾸니 정말 testcontainers 라이브러리가 사용하는 mysql이 맞는지를 물어보는 에러가 발생했다.
따라서 에러에서 설명하는 대로 설정을 수정하였다.
```kotlin
DockerImageName.parse("mysql/mysql-server:8.0.26")
                        .asCompatibleSubstituteFor("mysql")
                        .let { compatibleImageName -> MySQLContainer<Nothing>(compatibleImageName) }
```

---
### 🔧 mysql8 docker 설정 이슈
```
java.sql.SQLException: null,  message from server: "Host '172.17.0.1' is not allowed to connect to this MySQL server"
```
기존에는 주로 postgresql을 활용해서 인지 mysql8에서 필요한 옵션들을 조금 놓쳤던 것 같다.
mysql 8 docker 설정들을 구글링해서 env를 추가해서 최종적으로 연동에 성공하였다.
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

## 😄 결론
m1을 쓰다보니 실행속도나 배터리 시간 등 얻은 이득이 크지만 반대로 아직 생태계가 조금 부족한 부분들이 단점인 것 같다. 하지만 늘 그렇듯이 해결방안은 있기 마련이다. **끝!**
