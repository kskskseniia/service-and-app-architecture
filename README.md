# Архитектура сервисов и приложений

## Сессионное задание

Репозиторий содержит выполнение сессионного задания по дисциплине **«Архитектура сервисов и приложений»**.

---

# Задание 1. Рефакторинг системы управления онлайн-курсами

## 1. Краткий анализ исходной системы

В исходной реализации система управления онлайн-курсами построена вокруг класса `CourseManager`. Этот класс выполняет практически весь функционал.

Основные проблемы исходной реализации:

1. **Слишком много обязанностей в одном классе.**
   `CourseManager` объединяет бизнес-логику, создание объектов, уведомления и работу с внешними сервисами, что нарушает принцип разделения ответственности.

2. **Жёсткое создание курсов.**
   Курсы создаются через условные конструкции и прямые вызовы конструкторов. При добавлении нового типа курса необходимо изменять существующий код.

3. **Негибкий расчёт итоговой оценки.**
   Алгоритм оценивания закреплён внутри `CourseManager` и не может изменяться независимо от основной логики.

4. **Уведомления встроены в бизнес-логику.**
   Вызовы отправки уведомлений находятся внутри методов `CourseManager`, поэтому добавление нового канала уведомлений требует изменения класса.

5. **Прямая зависимость от платёжного API.**
   Система напрямую использует классы внешнего платёжного сервиса, из-за чего смена провайдера оплаты приведёт к изменению бизнес-логики.

6. **Дублирование логики.**
   Похожие действия повторяются в разных методах, что затрудняет сопровождение и повышает риск ошибок.


## 2. Предлагаемая структура системы

Для устранения проблем исходной архитектуры класс `CourseManager` предлагается оставить в системе, но изменить его роль. Теперь он не выполняет всю работу самостоятельно, а выступает как управляющий сервис, который координирует взаимодействие отдельных компонентов.

В новой архитектуре `CourseManager` использует следующие сервисы:

* `CourseService` — сервис управления курсами;
* `UserService` — сервис управления пользователями;
* `EnrollmentService` — сервис учёта связи между студентами и курсами;
* `PaymentService` — сервис оплаты курсов;
* `NotificationService` — сервис рассылки уведомлений.

### 2.1 Основная идея архитектуры

`CourseService` отвечает за создание, изменение, удаление и получение информации о курсах. Для создания объектов курсов он использует `CourseFactory`, а для хранения данных — `CourseRepository`. Благодаря этому `CourseManager` не зависит от конкретных классов курсов.

`UserService` отвечает за регистрацию пользователей, изменение пользовательских данных и получение информации о пользователях. Для работы с данными пользователей используется `UserRepository`.

`EnrollmentService` отвечает за запись студентов на курсы и хранение информации о связи между студентом и курсом. В этой связи могут храниться статус оплаты, прогресс прохождения курса, итоговая оценка и другие метаданные. Для хранения таких данных используется `EnrollmentRepository`.

`PaymentService` отвечает за проведение оплаты. Он получает стоимость курса из `CourseService`, проверяет статус оплаты через `EnrollmentService`, выполняет оплату через внешний платёжный сервис и после успешной оплаты обновляет статус оплаты.

`NotificationService` отвечает за рассылку уведомлений. Он работает по паттерну Observer: хранит список наблюдателей и передаёт им события, происходящие в системе. Например, уведомления могут отправляться при регистрации пользователя, записи на курс, успешной оплате или завершении курса.

Также в системе выделяется абстрактный класс `Course`, от которого наследуются конкретные типы курсов: `OnlineCourse`, `OfflineCourse` и `HybridCourse`. Каждый курс содержит стратегию расчёта итоговой оценки — `GradeStrategy`. Это позволяет использовать разные алгоритмы оценивания для разных типов курсов.

### 2.2 Диаграмма классов

```mermaid
classDiagram
    direction TB

    class CourseManager {
        -CourseService courseService
        -UserService userService
        -EnrollmentService enrollmentService
        -PaymentService paymentService
        -NotificationService notificationService
        +createCourse()
        +registerUser()
        +enrollStudent()
        +payForCourse()
        +completeCourse()
    }
    class UserService {
        -UserRepository userRepository
        +registerUser()
        +updateUserData()
        +getUser()
        +deleteUser()
    }

    class CourseService {
        -CourseFactory courseFactory
        -CourseRepository courseRepository
        +createCourse()
        +updateCourse()
        +deleteCourse()
        +getCourse()
        +getCoursePrice()
    }



    class EnrollmentService {
        -EnrollmentRepository enrollmentRepository
        +enrollStudent()
        +cancelEnrollment()
        +getPaymentStatus()
        +updatePaymentStatus()
        +updateProgress()
        +setFinalGrade()
    }

    class PaymentService {
        -CourseService courseService
        -EnrollmentService enrollmentService
        -PaymentGateway paymentGateway
        +payForCourse()
        +getPaymentStatus()
        +refundPayment()
    }

    class NotificationService {
        -observers
        +subscribe()
        +unsubscribe()
        +notify()
    }

    CourseManager --> UserService
    CourseManager --> CourseService
    CourseManager --> EnrollmentService
    CourseManager --> PaymentService
    CourseManager --> NotificationService

    class Course {
        <<abstract>>
        -GradeStrategy gradeStrategy
        +getTitle()
        +getPrice()
        +calculateFinalGrade()
    }

    class OnlineCourse
    class OfflineCourse
    class HybridCourse

    Course <|-- OnlineCourse
    Course <|-- OfflineCourse
    Course <|-- HybridCourse

    class CourseFactory {
        <<interface>>
        +createCourse()
    }

    class DefaultCourseFactory {
        +createCourse()
    }

    CourseFactory <|.. DefaultCourseFactory
    CourseService --> CourseFactory
    CourseService --> CourseRepository
    CourseService --> Course

    class CourseRepository {
        +save()
        +findById()
        +update()
        +delete()
    }

    class UserRepository {
        +save()
        +findById()
        +update()
        +delete()
    }

    UserService --> UserRepository

    class Enrollment {
        -id
        -studentId
        -courseId
        -paymentStatus
        -progress
        -courseStatus
        -finalGrade
    }

    class EnrollmentRepository {
        +save()
        +findByStudentId()
        +findByCourseId()
        +findByStudentAndCourse()
        +update()
        +delete()
    }

    EnrollmentService --> EnrollmentRepository
    EnrollmentRepository --> Enrollment

    class GradeStrategy {
        <<interface>>
        +calculate()
    }

    class TestGradeStrategy {
        +calculate()
    }

    class ExamGradeStrategy {
        +calculate()
    }

    Course --> GradeStrategy
    GradeStrategy <|.. TestGradeStrategy
    GradeStrategy <|.. ExamGradeStrategy

    class PaymentGateway {
        <<interface>>
        +pay()
        +refund()
        +getStatus()
    }

    class ExternalPaymentAdapter {
        -ExternalPaymentApi externalApi
        +pay()
        +refund()
        +getStatus()
    }

    class ExternalPaymentApi {
        +makePayment()
        +cancelPayment()
        +checkPayment()
    }

    PaymentService --> PaymentGateway
    PaymentGateway <|.. ExternalPaymentAdapter
    ExternalPaymentAdapter --> ExternalPaymentApi

    class NotificationObserver {
        <<interface>>
        +update()
    }

    class EmailNotificationObserver {
        +update()
    }

    class SmsNotificationObserver {
        +update()
    }

    class NotificationEvent {
        -type
        -recipient
        -message
        -metadata
    }

    NotificationService --> NotificationObserver
    NotificationService --> NotificationEvent
    NotificationObserver <|.. EmailNotificationObserver
    NotificationObserver <|.. SmsNotificationObserver

```

## 3. Описание основных классов и их ответственности

| Класс / интерфейс           | Ответственность                                                                                                                                                                                       |
| --------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `CourseManager`             | Управляющий сервис системы. Координирует работу остальных компонентов, но не содержит внутри себя всю бизнес-логику. Использует сервисы курсов, пользователей, записи на курсы, оплаты и уведомлений. |
| `CourseService`             | Отвечает за управление курсами: создание, изменение, удаление и получение информации о курсах. Для создания объектов использует `CourseFactory`, а для хранения данных — `CourseRepository`.          |
| `CourseFactory`             | Интерфейс фабрики для создания курсов разных типов. Позволяет не создавать `OnlineCourse`, `OfflineCourse` и другие классы напрямую в бизнес-логике.                                                  |
| `DefaultCourseFactory`      | Конкретная реализация фабрики курсов. Создаёт нужный объект курса в зависимости от переданного типа и данных.                                                                                         |
| `Course`                    | Абстрактный класс курса. Содержит общие свойства и методы для всех курсов, а также ссылку на стратегию расчёта итоговой оценки `GradeStrategy`.                                                       |
| `OnlineCourse`              | Конкретный тип курса, предназначенный для дистанционного обучения. Наследуется от `Course`.                                                                                                           |
| `OfflineCourse`             | Конкретный тип курса, предназначенный для очного обучения. Наследуется от `Course`.                                                                                                                   |
| `HybridCourse`              | Конкретный тип курса, объединяющий элементы онлайн- и офлайн-обучения. Наследуется от `Course`.                                                                                                       |
| `GradeStrategy`             | Интерфейс стратегии расчёта итоговой оценки. Позволяет задавать разные алгоритмы оценивания для разных курсов.                                                                                        |
| `TestGradeStrategy`         | Реализация стратегии, при которой итоговая оценка рассчитывается на основе тестов и заданий.                                                                                                          |
| `ExamGradeStrategy`         | Реализация стратегии, при которой итоговая оценка рассчитывается на основе экзамена.                                                                                                                  |
| `UserService`               | Отвечает за работу с пользователями: регистрацию, изменение данных, получение информации и удаление пользователей.                                                                                    |
| `UserRepository`            | Отвечает за хранение, поиск, обновление и удаление данных пользователей.                                                                                                                              |
| `EnrollmentService`         | Отвечает за запись студентов на курсы и управление связью между студентом и курсом. Хранит информацию о статусе оплаты, прогрессе прохождения, итоговой оценке и других метаданных.                   |
| `Enrollment`                | Сущность, описывающая связь между студентом и курсом. Содержит идентификаторы студента и курса, статус оплаты, прогресс, статус прохождения и итоговую оценку.                                        |
| `EnrollmentRepository`      | Отвечает за сохранение, поиск, обновление и удаление записей о связях между студентами и курсами.                                                                                                     |
| `PaymentService`            | Отвечает за процесс оплаты курса. Получает стоимость курса из `CourseService`, проверяет статус оплаты через `EnrollmentService`, вызывает платёжный шлюз и обновляет статус оплаты.                  |
| `PaymentGateway`            | Интерфейс для работы с платёжными системами. Определяет общие методы оплаты, возврата и проверки статуса платежа.                                                                                     |
| `ExternalPaymentAdapter`    | Адаптер для внешнего платёжного API. Преобразует вызовы системы к формату конкретного платёжного сервиса.                                                                                             |
| `ExternalPaymentApi`        | Внешний платёжный сервис, с которым система взаимодействует через адаптер.                                                                                                                            |
| `NotificationService`       | Сервис рассылки уведомлений. Хранит список наблюдателей и уведомляет их о событиях в системе.                                                                                                         |
| `NotificationObserver`      | Интерфейс наблюдателя, который получает событие и обрабатывает его.                                                                                                                                   |
| `EmailNotificationObserver` | Наблюдатель для отправки уведомлений по электронной почте.                                                                                                                                            |
| `SmsNotificationObserver`   | Наблюдатель для отправки уведомлений по SMS.                                                                                                                                                          |
| `NotificationEvent`         | Объект события, содержащий тип события, получателя, сообщение и дополнительные данные.                                                                                                                |

## 4. Соответствие проблем, решений и паттернов

| Проблема | Решение | Используемый паттерн | Обоснование |
|---|---|---|---|
| В классе `CourseManager` сосредоточено слишком много обязанностей | Обязанности разделены между `CourseService`, `UserService`, `EnrollmentService`, `PaymentService` и `NotificationService` | Разделение ответственности | Каждый сервис отвечает за свою часть системы, а `CourseManager` только координирует их работу |
| Создание курсов реализовано через условные конструкции и прямые вызовы конструкторов | Создание курсов вынесено в `CourseFactory`, которую использует `CourseService` | `Factory Method` | Добавление нового типа курса не требует изменения `CourseManager`; создание объектов сосредоточено в фабрике |
| Алгоритм расчёта итоговой оценки жёстко зафиксирован в одном методе | Расчёт оценки вынесен в интерфейс `GradeStrategy`, который используется внутри класса `Course` | `Strategy` | Для разных курсов можно применять разные алгоритмы оценивания без изменения основной логики |
| Отправка уведомлений встроена в бизнес-логику | Уведомления вынесены в `NotificationService`, который рассылает события наблюдателям | `Observer` | Бизнес-логика не зависит от конкретного канала уведомлений; можно добавить email, SMS или другой канал |
| Работа с платёжной системой реализована напрямую через внешний API | Введён интерфейс `PaymentGateway` и адаптер `ExternalPaymentAdapter` | `Adapter` | `PaymentService` работает с общим интерфейсом, поэтому внешний платёжный сервис можно заменить без изменения бизнес-логики |
| В коде присутствует дублирование логики | Повторяющиеся операции вынесены в отдельные сервисы и репозитории | DRY, разделение ответственности | Общая логика хранится в одном месте, поэтому её проще сопровождать и изменять |

## 5. Вывод

В результате переработки архитектуры класс `CourseManager` перестаёт быть перегруженным и выполняет роль управляющего сервиса. Основные обязанности системы распределены между отдельными компонентами: `CourseService`, `UserService`, `EnrollmentService`, `PaymentService` и `NotificationService`.

Создание курсов вынесено в фабрику, расчёт итоговой оценки реализован через стратегии, работа с платёжной системой изолирована с помощью адаптера, а уведомления организованы через механизм наблюдателей. Это позволяет добавлять новые типы курсов, алгоритмы оценивания, платёжные сервисы и каналы уведомлений без изменения основной бизнес-логики.

Предложенная структура снижает связанность компонентов, уменьшает дублирование кода и соответствует принципам DRY, KISS и разделения ответственности.
