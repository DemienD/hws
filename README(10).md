# ORM: объектно-реляционное отображение для Java

В прошлой лекции вы с одногруппниками написали и выполнили простые SQL-запросы с помощью `JdbcTemplate`, а в рамках
проекта написали простой REST API, который поддерживал работу с базой.

Такой подход к написанию кода взаимодействия с базой является достаточно простым и имеет преимущества. Однако в больших
проектах всё более явно проявляются его недостатки. Разберём их внимательней.

## Задания

- [Задание 1](#задание-1)
- [Задание 2](#задание-2)
- [Задание 3](#задание-3)

## Проблемы ручного написания SQL-запросов

При работе с базами данных через `JdbcTemplate` приходится писать SQL-запросы вручную. Это может показаться просто:
нужно описать запрос, выполнить его и обработать результат. Однако на практике такой подход имеет множество недостатков.

### Шаблонность SQL-запросов

Для каждой сущности в коде (например, класса `User` или `Order`) пишутся однотипные запросы: `SELECT`, `INSERT`,
`UPDATE`, `DELETE`. Эти запросы практически не меняются от сущности к сущности, но всё равно приходится их составлять
заново для каждого случая.

Пример:

```java
// Для User
String selectUser = "SELECT * FROM users WHERE id = ?";
String insertUser = "INSERT INTO users (name, email) VALUES (?, ?)";

// Для Order
String selectOrder = "SELECT * FROM orders WHERE id = ?";
String insertOrder = "INSERT INTO orders (user_id, product_id) VALUES (?, ?)";
```

### Маппинг ResultSet в объект Java

После выполнения запроса получаем `ResultSet`, который нужно преобразовать в Java-объект. Этот процесс тоже требует
много шаблонного кода:

```java
public User mapRow(ResultSet rs) throws SQLException {
    User user = new User();
    user.setId(rs.getInt("id"));
    user.setName(rs.getString("name"));
    user.setEmail(rs.getString("email"));
    return user;
}
```

Если полей много или структура данных сложная, то этот код может стать громоздким и трудно поддерживаемым. Кроме того,
при изменении структуры базы необходимо будет пройтись по всем таким методам и не забыть отразить в них изменения.

### Динамические запросы с фильтрами

Часто нужно создавать запросы динамически, например, когда пользователь выбирает несколько параметров для фильтрации.
Если какое-то поле не заполнено, его не должно быть в запросе. Это приводит к необходимости строить запрос программно:

```java
String query = "SELECT * FROM users WHERE 1=1";
if (filter.getName() != null) {
    query += " AND name = ?";
}
if (filter.getEmail() != null) {
    query += " AND email = ?";
}
```

Такой подход усложняет чтение и поддержку кода, а также увеличивает вероятность ошибок.

### SQL-инъекции

Одна из самых опасных проблем — SQL-инъекции. Если неправильно обрабатывать параметры запроса, злоумышленник может
внедрить вредоносный код.

Пример уязвимого запроса:

```java
String query = "SELECT * FROM users WHERE email = '" + userInputEmail + "'";
```

Если пользователь введёт `'; DROP TABLE users; --`, запрос станет:

```sql
SELECT *
FROM users
WHERE email = '';
DROP TABLE users; --
```

Это приведёт к удалению таблицы `users`.

Даже при использовании `JdbcTemplate` важно правильно использовать параметризованные запросы:

```java
String query = "SELECT * FROM users WHERE email = ?";
jdbcTemplate.query(query, new Object[]{userInputEmail}, ...);
```

### Прочие проблемы

Каждая база данных имеет свой диалект SQL. Например, в PostgreSQL и в MySQL разный набор типов. Это означает, что если
нужно будет перенести проект на другую СУБД, придётся изменять некоторые запросы.

Также при ручном написании нетривиальных запросов легко допустить ошибки, которые могут негативно повлиять на
производительность.

Рассмотрим, как эти проблемы решаются с помощью ORM (англ. Object-Relational Mapping) и Hibernate в частности.

## Введение в ORM

ORM (Object-Relational Mapping) — это концепция, которая позволяет работать с базами данных через объекты Java, не
задумываясь о написании SQL. Вместо того чтобы писать запросы вручную, нужно просто взаимодействовать с объектами
приложения, а ORM сам преобразует эти действия в соответствующие SQL-запросы.

Основная идея ORM заключается в том, что таблицы базы данных отображаются на классы Java, а строки таблиц — на объекты
этих классов. Например, если есть таблица `users`, то она будет представлена классом `User`, а каждая строка этой
таблицы — объектом класса `User`. Это значительно упрощает работу с данными, так как не нужно заботиться о маппинге
между объектами и таблицами вручную.

## Настройка проекта для работы со Spring Data

Для того чтобы подключить ORM в проект, необходимо использовать специальную интеграцию для спринга — `Spring Data`. Она
подключает ORM Hibernate и различные вспомогательные библиотеки для удобной работы с ним из `Spring`.

### **Добавление зависимостей**

В файле `build.gradle.kts` добавь следующую зависимость:

```kotlin

dependencies {
    // Библиотека для работы ORM
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
}
```

### **Настройка подключения к PostgreSQL**

Используем PostgreSQL, запущенный в Docker. Например, мы запустили PostgreSQL в Docker на порте 5432 командой:

```bash
docker run --name postgres -e POSTGRES_PASSWORD=secret -p 5432:5432 -d postgres
```

Теперь нужно настроить подключение к базе данных в файле `application.properties`:

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/postgres
spring.datasource.username=postgres
spring.datasource.password=secret
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
```

- `spring.datasource.url`: URL для подключения к базе данных.
- `spring.datasource.username` и `spring.datasource.password`: учётные данные.
- `spring.jpa.hibernate.ddl-auto=update`: автоматическое обновление структуры базы данных.
- `spring.jpa.show-sql=true`: показывать SQL-запросы в логах.
- `spring.jpa.properties.hibernate.dialect`: диалект SQL для PostgreSQL.

Обрати внимание, что, помимо уже знакомых свойств `spring.datasource`, используется и конфигурация `ORM`: эти свойства
мы указываем в секции `spring.jpa`.

Мы включим показ тех SQL-запросов, которые ORM будет генерировать из кода, скажем `ORM`, к какой базе мы подключаемся и
какой диалект SQL нужно использовать (в нашем случае — `PostgreSQL`). Также попросим автоматически генерировать при
старте DDL-операции — это операции, которые создают или изменяют структуру таблиц. Например, `CREATE TABLE` или
`ALTER TABLE`. Такой подход упросит разработку миниприложений. Но в серьёзных продакшен-решениях `ddl-auto` не
используется и все миграции структуры базы (создание, изменение и удаление таблиц и прочего) выполняются с помощью
специальных библиотек миграции, например, `Liquibase`.

## Базовые принципы Spring Data

Spring Data предоставляет удобный интерфейс для работы с базами данных через репозитории. Рассмотрим основные концепции.

### **Сущности**

Создай класс `User`, который будет представлять таблицу `users`. Используй аннотации JPA и Lombok для минимизации кода:

```java
import lombok.Getter;
import lombok.Setter;
import javax.persistence.*;

@Entity // Помечает класс как сущность, теперь можно её использовать в ORM
@Table(name = "users") // Указывает имя таблицы
@Getter
@Setter
public class User {

    @Id // Помечает поле как первичный ключ
    @GeneratedValue(strategy = GenerationType.IDENTITY) // Автоматическая генерация ID
    private Long id;

    @Column(nullable = false) // Поле не может быть NULL
    private String name;

    @Column(nullable = false, unique = true) // Поле уникально
    private String email;
}
```

Описывая сущность в коде, мы соотносим имя таблицы с именем класса, а также имя колонки с именем поля в классе с помощью
аннотаций `@Table` и `@Column`. Выбор типа колонки происходит исходя из того `Java`-типа, который заиспользован при
описании поля.

В каждой сущности должно быть одно поле, которое является первичным ключом, — оно отмечается аннотацией `@Id`. Если
значение поля автоматически генерируются базой при вставке объекта, то необходимо сообщить об этом ORM, проставив
аннотацию: `@GeneratedValue(strategy = GenerationType.IDENTITY)`

Так как мы ранее включили автоматическое управление структурой таблиц через свойство
`spring.jpa.hibernate.ddl-auto=update`, то при старте приложения, если в базе нет таблицы `users`, ORM Hibernate
автоматически создаст её, сгенерировав следующий SQL:

```sql
CREATE TABLE users
(
    id    BIGSERIAL PRIMARY KEY,       -- AUTO_INCREMENT
    name  VARCHAR(255) NOT NULL,       -- NOT NULL
    email VARCHAR(255) NOT NULL UNIQUE -- NOT NULL и UNIQUE
);
```

#### Задание 1

Опиши сущность `Book` с использованием Hibernate и Spring Data JPA. Сущность должна представлять таблицу `books` в базе
данных. Каждая книга имеет следующие характеристики:

1) **id** — уникальный идентификатор книги (автоинкремент);

2) **title** — название книги. Не может быть пустым;

3) **author** — автор книги. Не может быть пустым;

4) **year** — год издания книги. Целое число, необязательное поле;

5) **available** — флаг доступности книги в библиотеке. `true` — книга доступна, `false` — недоступна.

### Репозитории

Репозиторий — это интерфейс, который предоставляет методы для работы с данными. Чтобы его объявить, нужно создать свой
интерфейс, унаследовав его от интерфейса `JpaRepository`:

```java
import org.springframework.data.jpa.repository.JpaRepository;

@Repository
public interface UserRepository extends JpaRepository<User, Long> {
}
```

`JpaRepository` — это интерфейс из Spring Data JPA, который предоставляет стандартные CRUD-методы для работы с базой
данных.

Вот основные методы, которые доступны в `JpaRepository` из коробки:

| Метод               | Описание                                         |
|---------------------|--------------------------------------------------|
| `save(T entity)`    | Сохраняет или обновляет сущность в базе данных   |
| `findById(ID id)`   | Возвращает сущность по её ID                     |
| `findAll()`         | Возвращает все сущности из таблицы               |
| `deleteById(ID id)` | Удаляет сущность по её ID                        |
| `existsById(ID id)` | Проверяет, существует ли сущность с указанным ID |
| `count()`           | Возвращает количество записей в таблице          |
|                     |                                                  |

Главная киллер-фича Spring Data — он сам реализует интерфейс `UserRepository` и определяет все `CRUD`-методы для
сущности пользователь.

Посмотрим на базовый пример использования `UserRepository`.

#### Дефолтные методы `JpaRepository`

##### 1. Создание пользователя.

```java
User user = new User();
user.setName("John Doe");
user.setEmail("john.doe@example.com");

// Сохраняем пользователя в базу данных
userRepository.save(user);
```

Здесь Hibernate автоматически сгенерирует SQL-запрос:

```sql
INSERT INTO users (name, email)
VALUES ('John Doe', 'john.doe@example.com');
```

##### 2. Чтение пользователя.

```java
// Находим пользователя по ID
Optional<User> optionalUser = userRepository.findById(1L);

if (optionalUser.isPresent()) {
    User user = optionalUser.get();
    System.out.println("User found: " + user.getName());
} else {
    System.out.println("User not found");
}
```

Hibernate выполнит запрос:

```sql
SELECT *
FROM users
WHERE id = 1;
```

##### 3. Обновление пользователя.

```java
User user = userRepository.findById(1L).orElseThrow(() -> new RuntimeException("User not found"));
user.setName("John Updated");

// Сохраняем изменения
userRepository.save(user);
```

Hibernate выполнит запросы:

```sql
SELECT *
FROM users
WHERE id = 1;
UPDATE users
SET name  = 'John Updated',
    email = 'john.doe@example.com'
WHERE id = 1;
```

> Hibernate обновил все поля сущности, хотя обновлено было только имя. Дело в том, что Hibernate под капотом кэширует
`PreparedStatement` и не генерирует `SQL` в рантейме, это позволяет получать лучший перфоманс в большинстве случаев.

##### 4. Удаление пользователя.

```java
userRepository.deleteById(1L);
```

Hibernate выполнит запрос:

```sql
DELETE
FROM users
WHERE id = 1;
```

---

При вызове методов репозитория Hibernate автоматически генерирует соответствующие SQL-запросы. Больше не нужно писать
SQL вручную, нужно работать только с объектами Java.

#### Дополнительная функциональность репозиториев

Spring Data позволяет создавать кастомные методы для поиска данных без написания явных SQL-запросов (например, для
поиска пользователя по email, а не только по ID). Для этого нужно следовать определённому паттерну при названии методов
и описать их внутри интерфейса репозитория — реализация для таких методов по-прежнему будет сгенерирована:

```java
public interface UserRepository extends JpaRepository<User, Long> {
    // Можно добавить дополнительные методы, например:
    User findByEmail(String email); // Hibernate сгенерирует корректную реализацю сам
}
```

##### Правила именования методов

- Начинать название метода с глагола (`find`, `get`, `delete` и так далее).
- Использовать имя поля сущности (например, `name`, `email`).
- Добавлять логические операторы (`And`, `Or`, `Between`, `Like` и так далее).

  Например: `findByAgeBetween(int min, int max)` → `SELECT * FROM users WHERE age BETWEEN ? AND ?`.

##### Кастомные SQL-методы на HQL

Если стандартные возможности не подходят, можно использовать HQL (Hibernate Query Language). HQL очень похож на SQL, но
вместо имён таблиц и столбцов используется имя сущности и её полей. С его помощью можно описать более сложную логику.

Например, создадим запрос для поиска пользователей, у которых:

1) имя начинается на букву `'Д'`;

2) возраст больше указанного значения `age` (сначала добавим поле `age` в класс `User`).

```java
public interface UserRepository extends JpaRepository<User, Long> {

    // Кастомный HQL-запрос
    @Query("SELECT u FROM User u WHERE u.name LIKE 'Д%' AND u.age > :age")
    List<User> findUsersByNameStartsWithDAndAgeGreaterThan(@Param("age") int age);
}
```

HQL-запрос `SELECT u FROM User u WHERE u.name LIKE 'Д%' AND u.age > :age` работает следующим образом: он выбирает все
объекты сущности User, где поле name начинается на букву «Д» (благодаря оператору LIKE с шаблоном 'Д%') и при этом
значение поля age больше указанного параметра age. Hibernate автоматически преобразует этот запрос в соответствующий
SQL, заменяя :age на переданное значение и добавляя необходимые кавычки в условии LIKE в случае строковых аргументов (к
примеру, если фильтрация происходит по имени: `@Param(“name“) String name`). Такой подход позволяет фильтровать данные
точно и безопасно, избегая рисков SQL-инъекций.

Hibernate преобразует HQL-запрос в следующий SQL-запрос:

```sql
SELECT *
FROM users
WHERE name LIKE 'Д%'
  AND age > ?;
```

Здесь:

- `name LIKE 'Д%'` — фильтр по имени;
- `age > ?` — фильтр по возрасту (Hibernate автоматически подставит значение параметра `age`).

#### Задание 2

Опиши репозиторий, который предоставит методы `CRUD` для работы с ранее описанной сущностью `Book`. Напиши кастомный
метод для поиска книг по названию или автору (через совпадение подстроки), а также кастомный HQL-запрос для поиска
доступных книг (available = true) после указанного года.

#### Задание 3

Напиши REST-контроллер, который реализует CRUD-операции над сущностью Book.

- Получение всех книг.
- Получение книги по ID.
- Обновление книги по ID.
- Создание книги.
- Удаление книги по ID.

### Недостатки ORM

Несмотря на то что ORM (например, Hibernate) значительно упрощает работу с базами данных, это не универсальное решение.
У него есть свои недостатки и ограничения.

Наиболее известна проблема «N + 1», которая связана с загрузкой связанных сущностей (мы их не рассматривали в рамках
этого короткого курса). Когда появляются внешние ключи в таблицах, нужно осторожно использовать ORM и понимать, что
будет происходить под капотом.

Если тебя заинтересовала тема использования ORM, можешь прочитать книгу *Java Persistence with Hibernate*.

## Интеграционное тестирование

В современной разработке программного обеспечения интеграционное тестирование играет ключевую роль в обеспечении
устойчивости и надёжности приложений.

**Интеграционное тестирование** — это процесс проверки взаимодействия различных компонентов системы. В отличие от
юнит-тестов, которые фокусируются на изолированном тестировании конкретных функций или методов, интеграционные тесты
направлены на выявление проблем в работе нескольких компонентов вместе. Например, если приложение использует базу
данных, сервисы и контроллеры, то именно интеграционное тестирование поможет обнаружить ошибки при взаимодействии этих
частей системы.

**Отличия юнит-тестов и интеграционных тестов**

- **Юнит-тесты** проверяют внутреннюю логику одного модуля или класса без учёта внешних зависимостей.
- **Интеграционные тесты**, напротив, проверяют, как различные компоненты работают вместе, что особенно важно для
  сложных систем, где данные проходят через несколько уровней (например, от контроллера к базе данных).

Когда мы ранее писали юнит-тесты с логикой, которая работала с базой, то использовали `Mockito`, чтобы замокать
обращения к базе. В случае интеграционного тестирования мы поднимем базу внутри докера во время тестирования и будем
работать с ней.

### TestContainers

Одним из эффективных инструментов для выполнения интеграционных тестов является библиотека **Testcontainers**. Она
позволяет запускать временные контейнеры для тестирования, например, баз данных, без необходимости настройки реальных
экземпляров.

Для работы с PostgreSQL нужно добавить зависимость в `build.gradle.kts`:

```kotlin
	// Для запуска постгреса
    testImplementation("org.testcontainers:postgresql:1.17.6")
    // Для интеграции с Junit5
    testImplementation("org.testcontainers:junit-jupiter:1.17.6")
```

Посмотрим на примере, как использовать эту библиотеку для тестирования кода.

Код для тестирования:

```java
@RestController
@RequestMapping("/users")
public class UserController {

    // Внутри него обращается к Hibernate-репозиторию UserRepository
    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @PostMapping
    public ResponseEntity<User> createUser(@RequestBody User user) {
        User createdUser = userService.createUser(user);
        return ResponseEntity.status(HttpStatus.CREATED).body(createdUser);
    }
}
```

Комментарий к `UserController`

В примере выше метод контроллера принимает на вход и возвращает сущность Hibernate – объект класса `User`, помеченный
аннотацией `@Entity`. Но в больших приложениях лучше использовать DTO (Data Transfer Object), с которым вы познакомились
на предыдущих занятиях, например `UserCreateDto` в качестве RequestBody и `UserDto` в качестве возвращаемого значения.

Некоторые плюсы такого подхода:

1. В классе `User` есть поля, которые не нужно передавать при создании сущности, например `id`. Если использовать
   отдельный класс `UserCreateDto` без поля `id`, а только с необходимыми для создания сущности полями, то работа с API
   становится более явной.
2. В классе `User` могут быть чувствительные поля, которые не нужно возвращать из метода API, например номер паспорта
   или зашифрованный пароль. `UserDto` может содержать только те данные из класса `User`, которые можно получить
   пользователю API.

Определим «скелет» теста — в нём нужно подключить тестовый контейнер `postgresql`, а также настроить подключение к этому
тестовому контейнеру внутри кода:

```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
@Testcontainers
public class UserControllerIntegrationTest {

    @Container
    public static PostgreSQLContainer<?> postgreSQLContainer =
            new PostgreSQLContainer<>("postgres:14.1")
                    .withDatabaseName("testdb")
                    .withUsername("testuser")
                    .withPassword("testpass");

    @DynamicPropertySource
    static void postgresqlProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgreSQLContainer::getJdbcUrl);
        registry.add("spring.datasource.username", postgreSQLContainer::getUsername);
        registry.add("spring.datasource.password", postgreSQLContainer::getPassword);
    }
}
```

Как и всегда — снова куча магических аннотаций, которые делают всю работу. Разберём, что из них за что отвечает:

- `@SpringBootTest` — запускает полное приложение Spring Boot в тестовом контексте. Это означает, что все компоненты
  приложения (контроллеры, сервисы, репозитории и так далее) будут загружены так же, как при запуске приложения в
  обычном режиме.
- `@Testcontainers` — аннотация из библиотеки Testcontainers, которая сообщает Junit5, что тест зависит от временных
  контейнеров (например, PostgreSQL). Она автоматически управляет жизненным циклом контейнера: запускает его перед
  тестами и останавливает после их завершения.
- `@Container` — помечает поле как контейнер, который должен быть автоматически запущен перед выполнением тестов. В
  нашем случае это контейнер PostgreSQL. **Контейнер будет запущен заново для каждого метода.** Если нужно запустить
  один контейнер для всех методов, то нужно объявить это поле как `static`.
- `@DynamicPropertySource` — регистрирует динамические свойства (например, URL базы данных, имя пользователя и пароль)
  из контейнера PostgreSQL в конфигурацию Spring. Эти свойства используются для подключения к временной базе данных.

Такая заготовка является универсальной и может жить в абстрактном классе, от которого в свою очередь уже будут
наследоваться классы-наследники.

Напишем тест-метод для тестирования логики сохранения пользователей:

```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
@Testcontainers
public class UserControllerIntegrationTest {

    @Container
    public static PostgreSQLContainer<?> postgreSQLContainer =
            new PostgreSQLContainer<>("postgres:14.1")
                    .withDatabaseName("testdb")
                    .withUsername("testuser")
                    .withPassword("testpass");

    @DynamicPropertySource
    static void postgresqlProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgreSQLContainer::getJdbcUrl);
        registry.add("spring.datasource.username", postgreSQLContainer::getUsername);
        registry.add("spring.datasource.password", postgreSQLContainer::getPassword);
    }

    @Autowired
    private TestRestTemplate testRestTemplate;

    @Autowired
    private UserRepository userRepository; // Репозиторий для работы с базой данных

    @Test
    public void testCreateUser() {
        // Создаём пользователя для отправки
        User user = new User();
        user.setName("John Doe");
        user.setEmail("john.doe@example.com");

        // Отправляем POST-запрос через контроллер
        ResponseEntity<User> response = testRestTemplate.postForEntity("/users", user, User.class);

        // Проверяем HTTP-статус и содержимое ответа
        assertEquals(HttpStatus.CREATED, response.getStatusCode());
       
        String email = response.getBody().getEmail(); // Получаем email из ответа
        User savedUser = userRepository.findByEmail(email); // Ищем пользователя по email в базе

        assertNotNull(savedUser, "Пользователь не найден в базе данных!");
        assertEquals("John Doe", savedUser.getName(), "Имя пользователя не совпадает!");
        assertEquals(email, savedUser.getEmail(), "Email пользователя не совпадает!");
    }
}
```

Мы используем `TestRestTemplate`, чтобы делать запросы к реально запущенному Spring-приложению на рандомном порте. После
проверяем, что пользователь действительно был сохранён в базе данных.

Когда мы поднимаем тесты с помощью `@SpringBootTest`, то можем использовать любые бины из контекста. В том числе и те,
которые наследуются от `JpaRepository`. Это позволяет тестировать логику, которая работает с базой данных. 