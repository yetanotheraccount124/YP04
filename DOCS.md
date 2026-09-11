# АРХИТЕКТУРА И СТРУКТУРА БАЗЫ ДАННЫХ ПРОЕКТА "ЭЛЕКТРОННЫЙ ДНЕВНИК"

Проект представляет собой монолитное веб-приложение на языке **Python**, совмещающее асинхронный бэкенд на **Django + Django Ninja** и легковесный динамический фронтенд на базе **Django Templates + Alpine.js + Tailwind CSS**. Такой стек исключает необходимость развертывания Node.js-инфраструктуры, упрощает деплой и позволяет бесшовно упаковать интерфейс в десктопное приложение через **Electron**.

В основе серверной части лежит модифицированная **Луковичная архитектура (Onion Architecture)**, где бизнес-логика изолирована от деталей реализации (HTTP-фреймворка и ORM) через паттерн инверсии зависимостей.

---

## 1. Архитектура проекта и слои приложения

### 1.1. Доменная логика и ядро (Core / Domain)
Чистый Python-код (сервисы, сущности, Use Cases), полностью независимый от Django, баз данных и фронтенд-технологий.

*   **Модуль авторизации и управления пользователями:** Инкапсулирует правила ролевой модели (Администратор, Учитель, Ученик, Родитель), управление профилями и связями между родителями и детьми.
*   **Модуль расписания и занятий:** Содержит бизнес-логику пересечения занятий, занятости кабинетов, проверок доступности учителей и логику замен.
*   **Модуль успеваемости и оценок:** Обрабатывает правила выставления оценок, расчет средневзвешенного балла (с учетом веса работы) и триггеры для вывода итоговых оценок за четверть/полугодие.
*   **Модуль домашних заданий и посещаемости:** Логика дедлайнов, валидация прикрепляемых файлов и фиксация статусов присутствия.

### 1.2. Инфраструктурный слой (Infrastructure)
Реализация интерфейсов ядра с использованием возможностей Django и сторонних библиотек.

*   **Модуль базы данных (Django ORM + PostgreSQL):** Описывает модели данных. Django ORM используется как инструмент персистентности и управления миграциями, скрытый от ядра за паттерном «Репозиторий».
*   **Модуль логирования (Loguru / Django Logging):** Перехватывает системные ошибки Django, логирует входящие HTTP-запросы и действия пользователей (например, изменение оценок).
*   **Модуль уведомлений (Интеграция с Gotify):** Инфраструктурный клиент, отправляющий асинхронные HTTP-запросы на сервер Gotify при возникновении доменных событий (выставлена оценка, изменено расписание).

### 1.3. Слой представления и взаимодействия (Presentation)
Отвечает за отдачу интерфейса пользователю и обработку асинхронных запросов. Бэк и фронт работают на одном порту единого Django-процесса.

*   **REST API (Django Ninja):** Асинхронный слой эндпоинтов, обрабатывающий `GET/POST/PUT/DELETE` запросы от фронтенда. Использует Pydantic-схемы для строгой валидации входящих и исходящих данных.
*   **Интерфейс пользователя (Django Templates + Alpine.js + Tailwind CSS):** Легковесный динамический фронтенд без Node.js:
    *   *Django Templates* отдает базовые HTML-страницы и контекст (например, текущего пользователя).
    *   *Tailwind CSS (via CDN)* отвечает за адаптивную верстку и стили страниц прямо в HTML-классах.
    *   *Alpine.js (via CDN)* обеспечивает реактивность (открытие модальных окон, табы, динамические таблицы) и отправляет асинхронные запросы (Fetch API) к эндпоинтам Django Ninja без перезагрузки страниц.
*   **Панель администратора (Django Admin):** Встроенный интерфейс Django, кастомизированный для администраторов школы с целью оперативного управления пользователями, импорта предметов и глобального заполнения сетки звонков.
*   **Балансировщик нагрузки (Nginx):** Проксирует внешние запросы на ASGI-сервер (например, Uvicorn/Granian), раздающий Django-приложение, а также берет на себя отдачу статических файлов.

---

## 2. Структура таблиц базы данных (Django ORM / PostgreSQL)

Каждая таблица представлена в виде модели Django (`models.Model`). Идентификаторы сущностей используют `UUID` для безопасности API.

### 2.1. Модуль пользователей и прав доступа

#### Модель: User (Таблица: users)
Расширяет стандартную модель `AbstractUser` в Django.
*   id (UUIDField, Primary Key) — Уникальный идентификатор.
*   username (CharField, Unique) — Логин для входа.
*   password (CharField) — Хэшированный пароль (управляется Django).
*   email (EmailField, Optional) — Электронная почта.
*   phone (CharField) — Номер телефона.
*   role (CharField) — Выбор из списка (ADMIN, TEACHER, STUDENT, PARENT).
*   is_active (BooleanField) — Статус блокировки.
*   date_joined (DateTimeField) — Дата регистрации.

#### Модель: Profile (Таблица: profiles)
*   user (OneToOneField -> User) — Связь 1:1 с учетной записью.
*   first_name (CharField) — Имя.
*   last_name (CharField) — Фамилия.
*   middle_name (CharField, Optional) — Отчество.
*   birth_date (DateField) — Дата рождения.

#### Модель: ParentStudent (Таблица: parent_student)
Связующая таблица для отношения Many-to-Many между родителями и детьми.
*   parent (ForeignKey -> User, related_name='children_set')
*   student (ForeignKey -> User, related_name='parents_set')

### 2.2. Модуль организационной структуры школы

#### Модель: SchoolClass (Таблица: school_classes)
*   id (AutoField, Primary Key) — Идентификатор класса.
*   grade_number (PositiveIntegerField) — Параллель (например, 5, 11).
*   grade_letter (CharField) — Буква (например, А, Б).
*   homeroom_teacher (ForeignKey -> User, Optional) — Классный руководитель.

#### Модель: Subject (Таблица: subjects)
*   id (AutoField, Primary Key) — Идентификатор предмета.
*   name (CharField, Unique) — Название (например, Физика).

#### Модель: StudentClass (Таблица: student_classes)
*   student (ForeignKey -> User) — Ученик.
*   school_class (ForeignKey -> SchoolClass) — Класс.
*   academic_year (CharField) — Учебный год (например, "2026-2027").

### 2.3. Модуль расписания и занятий

#### Модель: TimeSlot (Таблица: time_slots)
*   id (PositiveIntegerField, Primary Key) — Номер урока.
*   start_time (TimeField) — Начало.
*   end_time (TimeField) — Конец.

#### Модель: Timetable (Таблица: timetable)
Шаблон недели (базовое расписание).
*   id (UUIDField, Primary Key)
*   school_class (ForeignKey -> SchoolClass) — Для какого класса.
*   subject (ForeignKey -> Subject) — Предмет.
*   teacher (ForeignKey -> User) — Преподаватель.
*   day_of_week (PositiveIntegerField) — День недели (1-7).
*   time_slot (ForeignKey -> TimeSlot) — Время урока.
*   classroom (CharField) — Кабинет.

#### Модель: Lesson (Таблица: lessons)
Фактический календарный урок. Генерируется автоматически на основе шаблонов Timetable.
*   id (UUIDField, Primary Key)
*   timetable (ForeignKey -> Timetable, Optional) — Ссылка на шаблон.
*   school_class (ForeignKey -> SchoolClass) — Класс.
*   subject (ForeignKey -> Subject) — Предмет.
*   teacher (ForeignKey -> User) — Фактический учитель (с учетом замен).
*   date (DateField) — Дата проведения урока.
*   time_slot (ForeignKey -> TimeSlot) — Время урока.
*   topic (CharField, Optional) — Тема урока (заполняет учитель).

### 2.4. Модуль успеваемости, домашних заданий и посещаемости

#### Модель: Homework (Таблица: homeworks)
*   id (UUIDField, Primary Key)
*   lesson (ForeignKey -> Lesson) — К какому уроку выдано.
*   description (TextField) — Текст задания.
*   file (FileField, Optional) — Ссылка на файл (обрабатывается Django Storage).
*   due_date (DateField) — Срок сдачи.

#### Модель: Attendance (Таблица: attendance)
*   id (UUIDField, Primary Key)
*   lesson (ForeignKey -> Lesson) — Урок.
*   student (ForeignKey -> User) — Ученик.
*   status (CharField) — Статус (Н — отс., У — уваж., О — опоздание).

#### Модель: Grade (Таблица: grades)
*   id (UUIDField, Primary Key)
*   lesson (ForeignKey -> Lesson) — Урок.
*   student (ForeignKey -> User) — Ученик.
*   teacher (ForeignKey -> User) — Кто выставил.
*   value (CharField) — Оценка (5, 4, 3, 2, НЗ).
*   weight (PositiveIntegerField) — Вес оценки (1 — ответ, 2 — контрольная).
*   comment (CharField, Optional) — Комментарий.
*   created_at (DateTimeField, auto_now_add=True) — Время выставления.
*   updated_at (DateTimeField, auto_now=True) — Время изменения.

### 2.5. Модуль системных логов

#### Модель: NotificationLog (Таблица: notification_logs)
*   id (UUIDField, Primary Key)
*   user (ForeignKey -> User) — Получатель.
*   title (CharField) — Заголовок.
*   body (TextField) — Текст сообщения.
*   is_sent (BooleanField) — Статус отправки в Gotify.
*   created_at (DateTimeField, auto_now_add=True)

#### Модель: AuditLog (Таблица: audit_logs)
*   id (BigAutoField, Primary Key)
*   user (ForeignKey -> User) — Инициатор действия.
*   action (CharField) — Тип действия (UPDATE_GRADE, etc.).
*   entity_name (CharField) — Имя таблицы.
*   entity_id (UUIDField) — ID измененной записи.
*   old_value (JSONField) — Состояние до изменения.
*   new_value (JSONField) — Состояние после изменения.
*   ip_address (GenericIPAddressField) — IP-адрес клиента.
*   created_at (DateTimeField, auto_now_add=True)
