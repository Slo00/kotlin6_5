# Nobel Prize API — задание 5 (модуль 6)

Серверное приложение на **Ktor**, которое:

- авторизует пользователей по JWT (BCrypt для паролей);
- импортирует данные из публичного **Nobel Prize API** (`api.nobelprize.org/v1/prize.json`) — разово, при первом старте;
- хранит данные в **PostgreSQL** на **neon.tech** через **Exposed + HikariCP**;
- позволяет добавлять/удалять любимые премии в избранное;
- логирует все запросы в консоль (`CallLogging`);
- отдаёт описание API через **Swagger UI** на `/docs`.

Архитектура — минимальная Clean Architecture внутри одного файла, разделённая секциями (конфиг → модели → таблицы → репозиторий → роуты).

## Структура проекта

```
task5_server/
├── build.gradle.kts                       # зависимости (Ktor, Exposed, HikariCP, JWT, BCrypt)
├── settings.gradle.kts
├── gradle.properties                      # указатель на JDK 21
├── gradlew, gradlew.bat, gradle/wrapper/  # gradle wrapper
└── src/main/
    ├── kotlin/Application.kt              # ВЕСЬ код сервера
    └── resources/logback.xml              # настройки логов
```

## Таблицы в БД

| Таблица       | Поля                                                              |
|---------------|-------------------------------------------------------------------|
| `users`       | id, username, password_hash, role                                 |
| `prizes`      | id, award_year, category, full_name, motivation, detail_link      |
| `laureates`   | id, prize_id, full_name, portion, motivation, portrait_url        |
| `user_prizes` | user_id, prize_id, added_at                                       |

## Эндпоинты

**Открытые:**

| Метод | Путь            | Что делает                                |
|-------|-----------------|-------------------------------------------|
| POST  | `/login`        | Возвращает JWT-токен                       |
| GET   | `/prizes`       | Список всех премий из БД                   |
| GET   | `/docs`         | Swagger UI с документацией                 |
| GET   | `/openapi.yaml` | Само OpenAPI-описание                      |

**Защищённые (нужен `Authorization: Bearer <token>`):**

| Метод  | Путь                              | Что делает                          |
|--------|-----------------------------------|-------------------------------------|
| GET    | `/users/me`                       | Профиль текущего пользователя       |
| GET    | `/users/me/prizes`                | Избранные премии                    |
| POST   | `/users/me/prizes/{prizeId}`      | Добавить премию в избранное         |
| DELETE | `/users/me/prizes/{prizeId}`      | Убрать из избранного                |

Дефолтный пользователь — `admin / admin123`, создаётся автоматически при первом запуске.

---

## Как запустить

```bash
cd task5_server
./gradlew run
```

Сервер поднимется на `http://localhost:8080`. После первого старта пройдёт ~2–5 минут на скачивание Nobel API и заливку ~680 премий в neon. На последующих запусках импорт пропускается.

### Пример запросов

```bash
# Логин — получаем токен
TOKEN=$(curl -s -X POST http://localhost:8080/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123"}' | jq -r .token)

# Все премии
curl -s http://localhost:8080/prizes | jq '. | length'

# Профиль
curl -s -H "Authorization: Bearer $TOKEN" http://localhost:8080/users/me

# Добавить премию №5 в избранное
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
     http://localhost:8080/users/me/prizes/5

# Избранные премии
curl -s -H "Authorization: Bearer $TOKEN" http://localhost:8080/users/me/prizes

# Удалить из избранного
curl -s -X DELETE -H "Authorization: Bearer $TOKEN" \
     http://localhost:8080/users/me/prizes/5
```

---

## Исходный код

### `build.gradle.kts`

```kotlin
plugins {
    kotlin("jvm") version "2.0.20"
    kotlin("plugin.serialization") version "2.0.20"
    application
}

group = "com.example"
version = "1.0.0"

application {
    mainClass.set("ApplicationKt")
}

kotlin {
    jvmToolchain(21)
}

repositories {
    mavenCentral()
}

val ktorVersion = "3.0.3"

dependencies {
    // Ktor сервер
    implementation("io.ktor:ktor-server-core:$ktorVersion")
    implementation("io.ktor:ktor-server-netty:$ktorVersion")
    implementation("io.ktor:ktor-server-content-negotiation:$ktorVersion")
    implementation("io.ktor:ktor-serialization-kotlinx-json:$ktorVersion")
    implementation("io.ktor:ktor-server-auth:$ktorVersion")
    implementation("io.ktor:ktor-server-auth-jwt:$ktorVersion")
    implementation("io.ktor:ktor-server-call-logging:$ktorVersion")
    implementation("io.ktor:ktor-server-status-pages:$ktorVersion")
    implementation("io.ktor:ktor-server-cors:$ktorVersion")

    // Ktor клиент для похода в Nobel API
    implementation("io.ktor:ktor-client-core:$ktorVersion")
    implementation("io.ktor:ktor-client-cio:$ktorVersion")
    implementation("io.ktor:ktor-client-content-negotiation:$ktorVersion")

    // База
    implementation("org.jetbrains.exposed:exposed-core:0.55.0")
    implementation("org.jetbrains.exposed:exposed-dao:0.55.0")
    implementation("org.jetbrains.exposed:exposed-jdbc:0.55.0")
    implementation("org.jetbrains.exposed:exposed-java-time:0.55.0")
    implementation("org.postgresql:postgresql:42.7.4")
    implementation("com.zaxxer:HikariCP:6.0.0")

    // Пароли + JWT
    implementation("at.favre.lib:bcrypt:0.10.2")
    implementation("com.auth0:java-jwt:4.4.0")

    // Логи
    implementation("ch.qos.logback:logback-classic:1.5.6")
}
```

### `settings.gradle.kts`

```kotlin
rootProject.name = "nobel-prize-api"
```

### `gradle.properties`

```properties
kotlin.code.style=official
org.gradle.jvmargs=-Xmx2g
# Используем заранее скачанный JDK 21 из Gradle toolchains
org.gradle.java.home=/Users/kvltyapka/.gradle/jdks/eclipse_adoptium-21-aarch64-os_x.2/jdk-21.0.7+6/Contents/Home
```

### `src/main/resources/logback.xml`

```xml
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="STDOUT"/>
    </root>

    <logger name="org.eclipse.jetty" level="WARN"/>
    <logger name="io.netty" level="WARN"/>
    <logger name="Exposed" level="WARN"/>
</configuration>
```

### `src/main/kotlin/Application.kt`

```kotlin
import at.favre.lib.crypto.bcrypt.BCrypt
import com.auth0.jwt.JWT
import com.auth0.jwt.JWTVerifier
import com.auth0.jwt.algorithms.Algorithm
import com.zaxxer.hikari.HikariConfig
import com.zaxxer.hikari.HikariDataSource
import io.ktor.client.HttpClient
import io.ktor.client.call.body
import io.ktor.client.engine.cio.CIO
import io.ktor.client.plugins.contentnegotiation.ContentNegotiation as ClientNegotiation
import io.ktor.client.request.get
import io.ktor.http.ContentType
import io.ktor.http.HttpStatusCode
import io.ktor.serialization.kotlinx.json.json
import io.ktor.server.application.Application
import io.ktor.server.application.call
import io.ktor.server.application.install
import io.ktor.server.auth.Authentication
import io.ktor.server.auth.authenticate
import io.ktor.server.auth.jwt.JWTPrincipal
import io.ktor.server.auth.jwt.jwt
import io.ktor.server.auth.principal
import io.ktor.server.engine.embeddedServer
import io.ktor.server.netty.Netty
import io.ktor.server.plugins.calllogging.CallLogging
import io.ktor.server.plugins.contentnegotiation.ContentNegotiation
import io.ktor.server.plugins.cors.routing.CORS
import io.ktor.server.plugins.statuspages.StatusPages
import io.ktor.server.request.receive
import io.ktor.server.response.respond
import io.ktor.server.response.respondText
import io.ktor.server.routing.delete
import io.ktor.server.routing.get
import io.ktor.server.routing.post
import io.ktor.server.routing.routing
import kotlinx.serialization.Serializable
import kotlinx.serialization.json.Json
import org.jetbrains.exposed.dao.id.IntIdTable
import org.jetbrains.exposed.sql.Database
import org.jetbrains.exposed.sql.SchemaUtils
import org.jetbrains.exposed.sql.SqlExpressionBuilder
import org.jetbrains.exposed.sql.Table
import org.jetbrains.exposed.sql.and
import org.jetbrains.exposed.sql.deleteWhere
import org.jetbrains.exposed.sql.insert
import org.jetbrains.exposed.sql.javatime.datetime
import org.jetbrains.exposed.sql.selectAll
import org.jetbrains.exposed.sql.transactions.experimental.newSuspendedTransaction
import org.jetbrains.exposed.sql.transactions.transaction
import org.slf4j.LoggerFactory
import java.time.LocalDateTime
import java.util.Date

private val log = LoggerFactory.getLogger("App")

// =====================================================================
// Конфиг — читаем из переменных окружения, иначе берём дефолты
// =====================================================================

object Cfg {
    val dbUrl   = System.getenv("DB_URL")
        ?: "jdbc:postgresql://ep-spring-sea-aqgeaiva.c-8.us-east-1.aws.neon.tech/neondb?sslmode=require"
    val dbUser  = System.getenv("DB_USER")  ?: "neondb_owner"
    val dbPass  = System.getenv("DB_PASS")  ?: "npg_D4xSRFiXepl8"
    val jwtSecret = System.getenv("JWT_SECRET") ?: "nobel-prize-api-jwt-secret-key-2026"
    const val JWT_ISSUER   = "nobel-api"
    const val JWT_AUDIENCE = "nobel-clients"
    const val JWT_REALM    = "Nobel API"
}

// =====================================================================
// Модели для API (то, что отдаём клиенту)
// =====================================================================

@Serializable
data class LoginRequest(val username: String, val password: String)

@Serializable
data class TokenResponse(val token: String)

@Serializable
data class UserDto(val id: Int, val username: String, val role: String)

@Serializable
data class LaureateDto(
    val id: Int,
    val fullName: String,
    val portion: String?,
    val motivation: String?,
    val portraitUrl: String?
)

@Serializable
data class PrizeDto(
    val id: Int,
    val awardYear: Int,
    val category: String,
    val fullName: String,
    val motivation: String?,
    val detailLink: String?,
    val laureates: List<LaureateDto> = emptyList()
)

@Serializable
data class MessageResponse(val message: String)

// DTO для Nobel API (то что приходит из api.nobelprize.org/v1/prize.json)
@Serializable
private data class NobelResponse(val prizes: List<NobelPrize> = emptyList())

@Serializable
private data class NobelPrize(
    val year: String,
    val category: String,
    val overallMotivation: String? = null,
    val laureates: List<NobelLaureate> = emptyList()
)

@Serializable
private data class NobelLaureate(
    val id: String,
    val firstname: String? = null,
    val surname: String? = null,
    val motivation: String? = null,
    val share: String? = null
)

// =====================================================================
// Таблицы Exposed
// =====================================================================

object Users : IntIdTable("users") {
    val username     = varchar("username", 64).uniqueIndex()
    val passwordHash = varchar("password_hash", 100)
    val role         = varchar("role", 16).default("user")
}

object Prizes : IntIdTable("prizes") {
    val awardYear  = integer("award_year")
    val category   = varchar("category", 64)
    val fullName   = varchar("full_name", 256)
    val motivation = text("motivation").nullable()
    val detailLink = varchar("detail_link", 512).nullable()

    init { uniqueIndex(awardYear, category) }
}

object Laureates : IntIdTable("laureates") {
    val prizeId     = reference("prize_id", Prizes)
    val fullName    = varchar("full_name", 256)
    val portion     = varchar("portion", 16).nullable()
    val motivation  = text("motivation").nullable()
    val portraitUrl = varchar("portrait_url", 512).nullable()
}

object UserPrizes : Table("user_prizes") {
    val userId  = reference("user_id", Users)
    val prizeId = reference("prize_id", Prizes)
    val addedAt = datetime("added_at").clientDefault { LocalDateTime.now() }

    override val primaryKey = PrimaryKey(userId, prizeId)
}

// =====================================================================
// Инициализация БД (HikariCP + Exposed) и тестовые данные
// =====================================================================

fun initDatabase() {
    val hk = HikariConfig().apply {
        jdbcUrl = Cfg.dbUrl
        driverClassName = "org.postgresql.Driver"
        username = Cfg.dbUser
        password = Cfg.dbPass
        maximumPoolSize = 10
        minimumIdle = 2
        idleTimeout = 300_000
        maxLifetime = 1_800_000
        connectionTimeout = 30_000
    }
    Database.connect(HikariDataSource(hk))

    transaction {
        SchemaUtils.create(Users, Prizes, Laureates, UserPrizes)

        // Заводим админа, если его ещё нет
        val exists = Users.selectAll().where { Users.username eq "admin" }.any()
        if (!exists) {
            Users.insert {
                it[username]     = "admin"
                it[passwordHash] = hashPassword("admin123")
                it[role]         = "admin"
            }
            log.info("Создан пользователь admin / admin123")
        }
    }
}

// =====================================================================
// Пароли — обёртка над BCrypt
// =====================================================================

fun hashPassword(plain: String): String =
    BCrypt.withDefaults().hashToString(12, plain.toCharArray())

fun verifyPassword(plain: String, hash: String): Boolean =
    BCrypt.verifyer().verify(plain.toCharArray(), hash).verified

// =====================================================================
// JWT — генерация токена и верификатор для Ktor
// =====================================================================

fun generateToken(user: UserDto): String {
    val algorithm = Algorithm.HMAC256(Cfg.jwtSecret)
    return JWT.create()
        .withIssuer(Cfg.JWT_ISSUER)
        .withAudience(Cfg.JWT_AUDIENCE)
        .withClaim("uid", user.id)
        .withClaim("username", user.username)
        .withClaim("role", user.role)
        .withExpiresAt(Date(System.currentTimeMillis() + 24 * 60 * 60 * 1000))
        .sign(algorithm)
}

fun jwtVerifier(): JWTVerifier =
    JWT.require(Algorithm.HMAC256(Cfg.jwtSecret))
        .withIssuer(Cfg.JWT_ISSUER)
        .withAudience(Cfg.JWT_AUDIENCE)
        .build()

// =====================================================================
// Загрузка призов из Nobel Prize API (разовый ручной перенос)
// =====================================================================

private const val NOBEL_URL = "https://api.nobelprize.org/v1/prize.json"

suspend fun importPrizesFromNobelApi() {
    // Если в таблице уже что-то есть — больше не лезем в API.
    val already = newSuspendedTransaction { Prizes.selectAll().count() }
    if (already > 0) {
        log.info("В БД уже {} премий, импорт пропущен", already)
        return
    }

    val client = HttpClient(CIO) {
        install(ClientNegotiation) {
            json(Json { ignoreUnknownKeys = true; isLenient = true })
        }
    }

    log.info("Загружаем премии из Nobel API...")
    val parsed: NobelResponse = try {
        client.get(NOBEL_URL).body()
    } catch (e: Exception) {
        log.warn("Не удалось скачать данные из Nobel API: {}", e.message)
        client.close()
        return
    }
    client.close()

    newSuspendedTransaction {
        for (p in parsed.prizes) {
            val year = p.year.toIntOrNull() ?: continue
            val catNice = p.category.replaceFirstChar { it.uppercase() }
            val link = "https://www.nobelprize.org/prizes/${p.category}/${p.year}/summary/"

            val prizeId = Prizes.insert {
                it[awardYear]  = year
                it[category]   = p.category
                it[fullName]   = "The Nobel Prize in $catNice"
                it[motivation] = p.overallMotivation
                it[detailLink] = link
            } get Prizes.id

            for (l in p.laureates) {
                Laureates.insert {
                    it[Laureates.prizeId] = prizeId
                    it[fullName]   = listOfNotNull(l.firstname, l.surname).joinToString(" ").ifBlank { "Unknown" }
                    it[portion]    = l.share?.let { s -> "1/$s" }
                    it[motivation] = l.motivation
                    it[portraitUrl] = null // в v1 API нет портретов
                }
            }
        }
    }
    log.info("Импорт завершён, всего премий: {}", parsed.prizes.size)
}

// =====================================================================
// Маленький "репозиторий" — функции к БД
// =====================================================================

suspend fun findUserByName(name: String): Pair<UserDto, String>? = newSuspendedTransaction {
    Users.selectAll().where { Users.username eq name }
        .map {
            UserDto(it[Users.id].value, it[Users.username], it[Users.role]) to it[Users.passwordHash]
        }
        .firstOrNull()
}

suspend fun findUserById(id: Int): UserDto? = newSuspendedTransaction {
    Users.selectAll().where { Users.id eq id }
        .map { UserDto(it[Users.id].value, it[Users.username], it[Users.role]) }
        .firstOrNull()
}

suspend fun loadAllPrizes(): List<PrizeDto> = newSuspendedTransaction {
    val laureatesByPrize = Laureates.selectAll()
        .groupBy { it[Laureates.prizeId].value }
        .mapValues { (_, rows) ->
            rows.map {
                LaureateDto(
                    id = it[Laureates.id].value,
                    fullName = it[Laureates.fullName],
                    portion = it[Laureates.portion],
                    motivation = it[Laureates.motivation],
                    portraitUrl = it[Laureates.portraitUrl]
                )
            }
        }

    Prizes.selectAll().map {
        val pid = it[Prizes.id].value
        PrizeDto(
            id = pid,
            awardYear = it[Prizes.awardYear],
            category = it[Prizes.category],
            fullName = it[Prizes.fullName],
            motivation = it[Prizes.motivation],
            detailLink = it[Prizes.detailLink],
            laureates = laureatesByPrize[pid].orEmpty()
        )
    }
}

suspend fun loadFavoritePrizes(userId: Int): List<PrizeDto> {
    val favIds = newSuspendedTransaction {
        UserPrizes.selectAll().where { UserPrizes.userId eq userId }
            .map { it[UserPrizes.prizeId].value }
            .toSet()
    }
    if (favIds.isEmpty()) return emptyList()
    return loadAllPrizes().filter { it.id in favIds }
}

suspend fun addFavorite(userId: Int, prizeId: Int): Boolean = newSuspendedTransaction {
    // Проверим, что такая премия вообще есть
    val prizeExists = Prizes.selectAll().where { Prizes.id eq prizeId }.any()
    if (!prizeExists) return@newSuspendedTransaction false

    val already = UserPrizes.selectAll()
        .where { (UserPrizes.userId eq userId) and (UserPrizes.prizeId eq prizeId) }
        .any()
    if (already) return@newSuspendedTransaction true

    UserPrizes.insert {
        it[UserPrizes.userId]  = userId
        it[UserPrizes.prizeId] = prizeId
    }
    true
}

suspend fun removeFavorite(userId: Int, prizeId: Int): Boolean = newSuspendedTransaction {
    val n = UserPrizes.deleteWhere {
        with(SqlExpressionBuilder) {
            (UserPrizes.userId eq userId) and (UserPrizes.prizeId eq prizeId)
        }
    }
    n > 0
}

// =====================================================================
// Конфигурация Ktor: плагины + маршруты
// =====================================================================

fun Application.module() {
    install(ContentNegotiation) {
        json(Json {
            prettyPrint = true
            ignoreUnknownKeys = true
        })
    }

    install(CallLogging)

    install(CORS) {
        anyHost()
        allowHeader("Authorization")
        allowHeader("Content-Type")
        allowMethod(io.ktor.http.HttpMethod.Put)
        allowMethod(io.ktor.http.HttpMethod.Delete)
        allowMethod(io.ktor.http.HttpMethod.Patch)
    }

    install(StatusPages) {
        exception<Throwable> { call, cause ->
            log.error("Необработанная ошибка", cause)
            call.respond(HttpStatusCode.InternalServerError, MessageResponse("Internal error: ${cause.message}"))
        }
    }

    install(Authentication) {
        jwt("auth-jwt") {
            realm = Cfg.JWT_REALM
            verifier(jwtVerifier())
            validate { credential ->
                val uid = credential.payload.getClaim("uid").asInt()
                if (uid != null) JWTPrincipal(credential.payload) else null
            }
        }
    }

    routing {
        // ---------- Открытые маршруты ----------

        get("/") {
            call.respondText("Nobel Prize API. См. /docs", ContentType.Text.Plain)
        }

        // Простенькая документация (Swagger UI на CDN + наш openapi.yaml)
        get("/docs") {
            call.respondText(SWAGGER_HTML, ContentType.Text.Html)
        }
        get("/openapi.yaml") {
            call.respondText(OPENAPI_YAML, ContentType.parse("application/yaml"))
        }

        post("/login") {
            val req = call.receive<LoginRequest>()
            val found = findUserByName(req.username)
            if (found == null || !verifyPassword(req.password, found.second)) {
                call.respond(HttpStatusCode.Unauthorized, MessageResponse("Неверный логин или пароль"))
                return@post
            }
            call.respond(TokenResponse(generateToken(found.first)))
        }

        get("/prizes") {
            call.respond(loadAllPrizes())
        }

        // ---------- Защищённые маршруты (нужен JWT) ----------

        authenticate("auth-jwt") {
            get("/users/me") {
                val uid = call.principal<JWTPrincipal>()!!.payload.getClaim("uid").asInt()
                val user = findUserById(uid)
                if (user == null) call.respond(HttpStatusCode.NotFound, MessageResponse("Пользователь не найден"))
                else call.respond(user)
            }

            get("/users/me/prizes") {
                val uid = call.principal<JWTPrincipal>()!!.payload.getClaim("uid").asInt()
                call.respond(loadFavoritePrizes(uid))
            }

            post("/users/me/prizes/{prizeId}") {
                val uid = call.principal<JWTPrincipal>()!!.payload.getClaim("uid").asInt()
                val pid = call.parameters["prizeId"]?.toIntOrNull()
                if (pid == null) {
                    call.respond(HttpStatusCode.BadRequest, MessageResponse("prizeId должен быть числом"))
                    return@post
                }
                val ok = addFavorite(uid, pid)
                if (!ok) call.respond(HttpStatusCode.NotFound, MessageResponse("Премия не найдена"))
                else call.respond(MessageResponse("Добавлено"))
            }

            delete("/users/me/prizes/{prizeId}") {
                val uid = call.principal<JWTPrincipal>()!!.payload.getClaim("uid").asInt()
                val pid = call.parameters["prizeId"]?.toIntOrNull()
                if (pid == null) {
                    call.respond(HttpStatusCode.BadRequest, MessageResponse("prizeId должен быть числом"))
                    return@delete
                }
                val ok = removeFavorite(uid, pid)
                if (!ok) call.respond(HttpStatusCode.NotFound, MessageResponse("Не было в избранном"))
                else call.respond(MessageResponse("Удалено"))
            }
        }
    }
}

// =====================================================================
// main: поднимаем БД, импортируем призы, запускаем сервер на 8080
// =====================================================================

fun main() {
    initDatabase()

    // Импорт делаем в фоне, чтобы сервер успел подняться даже без интернета
    Thread {
        try {
            kotlinx.coroutines.runBlocking { importPrizesFromNobelApi() }
        } catch (e: Exception) {
            log.warn("Импорт упал: {}", e.message)
        }
    }.start()

    embeddedServer(Netty, port = 8080, host = "0.0.0.0") {
        module()
    }.start(wait = true)
}

// =====================================================================
// OpenAPI описание (минимальное) + страничка Swagger UI
// Лежит в коде специально, чтобы не плодить лишних файлов в ресурсах.
// =====================================================================

private val SWAGGER_HTML = """
<!DOCTYPE html>
<html>
<head>
    <title>Nobel Prize API — docs</title>
    <link rel="stylesheet" href="https://unpkg.com/swagger-ui-dist@5/swagger-ui.css">
</head>
<body>
<div id="ui"></div>
<script src="https://unpkg.com/swagger-ui-dist@5/swagger-ui-bundle.js"></script>
<script>
  window.onload = () => SwaggerUIBundle({ url: '/openapi.yaml', dom_id: '#ui' });
</script>
</body>
</html>
""".trimIndent()

private val OPENAPI_YAML = """
openapi: 3.0.0
info:
  title: Nobel Prize API
  version: "1.0.0"
  description: Учебный сервер на Ktor + Exposed + PostgreSQL (neon.tech)
servers:
  - url: http://localhost:8080
paths:
  /login:
    post:
      summary: Авторизация, возвращает JWT
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                username: { type: string }
                password: { type: string }
      responses:
        '200': { description: OK }
        '401': { description: Неверные данные }
  /prizes:
    get:
      summary: Список всех премий из БД
      responses:
        '200': { description: OK }
  /users/me:
    get:
      summary: Профиль текущего пользователя
      security: [{ bearerAuth: [] }]
      responses:
        '200': { description: OK }
  /users/me/prizes:
    get:
      summary: Избранные премии пользователя
      security: [{ bearerAuth: [] }]
      responses:
        '200': { description: OK }
  /users/me/prizes/{prizeId}:
    post:
      summary: Добавить премию в избранное
      security: [{ bearerAuth: [] }]
      parameters:
        - in: path
          name: prizeId
          required: true
          schema: { type: integer }
      responses:
        '200': { description: OK }
        '404': { description: Премия не найдена }
    delete:
      summary: Удалить премию из избранного
      security: [{ bearerAuth: [] }]
      parameters:
        - in: path
          name: prizeId
          required: true
          schema: { type: integer }
      responses:
        '200': { description: OK }
        '404': { description: Не было в избранном }
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
""".trimIndent()
```
