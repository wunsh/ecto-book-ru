## Содержание

- [Вступление](#Вступление)
- [Новшества Ecto 2.X](#Новшества-ecto-2x)
  - [Обновлённый модуль Ecto.Changeset](#Обновлённый-модуль-ectochangeset)
  - [Новая функция Subquery/1 в модуле Ecto.Query](#Новая-функция-subquery1-в-модуле-ectoquery)
  - [Новая функция insert_all/3 в модуле Ecto.Repo](#Новая-функция-insert_all3-в-модуле-ectorepo)
  - [Добавлена Many-to-Many ассоциация](#Добавлена-many-to-many-ассоциация)
  - [Улучшена работа с ассоциациями](#Улучшена-работа-с-ассоциациями)
  - [Новая функция предзагрузки ассоциаций assoc/2 в модуле Ecto](#Новая-функция-предзагрузки-ассоциаций-assoc2-в-модуле-ecto)
  - [Upsert](#upsert)
  - [Новые условия выборки or_where и or_having](#Новые-условия-выборки-or_where-и-or_having)
  - [Добавлена возможность пошагового построения запроса](#Добавлена-возможность-пошагового-построения-запроса)
- [Интерфейс запросов](#Интерфейс-запросов)
  - [Получение одиночной строки](#1-Получение-одиночной-строки)
  - [Получение нескольких строк](#2-Получение-нескольких-строк)
  - [Условия выборки строк](#3-Условия-выборки-строк)
  - [Сортировка строк](#4-Сортировка-строк)
  - [Выбор определенных полей строк](#5-Выбор-определенных-полей-строк)
  - [Группировка строк](#6-Группировка-строк)
  - [Лимитирование и смещение выбираемых строк](#7-Лимитирование-и-смещение-выбираемых-строк)
  - [Соединение таблиц](#8-Соединение-таблиц)
  - [Переопределяющее условие](#9-Переопределяющее-условие)
- [Команды для массовых операций со строками](#Команды-для-массовых-операций-со-строками)
- [Практические примеры](#Практические-примеры)
- [Дополнение из Ecto.Adapters.SQL](#Дополнение-из-ectoadapterssql)

## Вступление

Ecto написанный на Elixir DSL для коммуникации с базами данных. Ecto это не ORM. Почему? Да, потому что Elixir не объектно-ориентированный язык, вот и Ecto не может быть Object-Relational Mapping (объектно-реляционным отображением). Ecto - это абстракция над базами данных состоящая из нескольких больших модулей, которые позволяют создавать миграции, объявлять модели (схемы), добавлять и обновлять данные, а также посылать к ним запросы.

На данный момент актуальная версия Ecto 2, она совместима с PostgreSQL и MySQL. Более ранняя версия дополнительно имеет совместимость с MSSQL, SQLite3 и MongoDB. Независимо от того, какая используется СУБД, формат функций Ecto будет всегда одинаковый. Также Ecto идёт из коробки с Phoenix и является хорошим стандартным решением.

## Новшества Ecto 2.X

- [Обновлённый модуль Ecto.Changeset](#Обновлённый-модуль-ectochangeset)
- [Новая функция Subquery/1 в модуле Ecto.Query](#Новая-функция-subquery1-в-модуле-ectoquery)
- [Новая функция insert_all/3 в модуле Ecto.Repo](#Новая-функция-insert_all3-в-модуле-ectorepo)
- [Добавлена Many-to-Many ассоциация](#Добавлена-many-to-many-ассоциация)
- [Улучшена работа с ассоциациями](#Улучшена-работа-с-ассоциациями)
- [Новая функция предзагрузки ассоциаций assoc/2 в модуле Ecto](#Новая-функция-предзагрузки-ассоциаций-assoc2-в-модуле-ecto)
- [Upsert](#upsert)
- [Новые условия выборки or_where и or_having](#Новые-условия-выборки-or_where-и-or_having)
- [Добавлена возможность пошагового построения запроса](#Добавлена-возможность-пошагового-построения-запроса)

### Обновлённый модуль Ecto.Changeset

1. changeset.model переименована в changeset.data (отныне "models" в Ecto нет).
2. Устаревшим считается передача обязательных полей и опций для них в `cast/4`, отныне следует использовать `cast/3` и `validate_required/3`.
3. Атом `:empty` в `cast(source, :empty, required, optional)` стал устаревшим, желательно вместо этого использовать `empty map` or `:invalid`.

Как итог, вместо этого:

```elixir
def changeset(user, params \\ :empty) do
  user
  |> cast(params, [:name], [:age])
end
```

Рекомендуется делать лучше так:

```elixir
def changeset(user, params \\ %{}) do
  user
  |> cast(params, [:name, :age])
  |> validate_required([:name])
end
```

### Новая функция Subquery/1 в модуле Ecto.Query

Функция `Ecto.Query.subquery/1` даёт возможность конвертировать любые запросы в подзапросы. Для примера, если вы хотите посчитать среднее кол-во просмотров публикаций, вы можете написать:

```elixir
query = from p in Post, select: avg(p.visits)
TestRepo.all(query) #=> [#Decimal<1743>]
```

Однако, если вы хотите посчитать средне кол-во просмотров только для 10 самых популярных постов, вам потребуется подзапрос:

```elixir
query = from p in Post, select: [:visits], order_by: [desc: :visits], limit: 10
TestRepo.all(from p in subquery(query), select: avg(p.visits)) #=> [#Decimal<4682>]
```

Для практического примера, если вы используете для подсчёта агрегированных данных функцию `Repo.aggregate`:

```elixir
# Среднее кол-во просмотров для всех публикаций
TestRepo.aggregate(Post, :avg, :visits) #=> #Decimal<1743>
```

```elixir
# Среднее кол-во просмотров для 10 самых популярных публикаций
query = from Post, order_by: [desc: :visits], limit: 10
TestRepo.aggregate(query, :avg, :visits) #=> #Decimal<4682>
```

В `subquery/1` есть возможность задавать имена полям в подзапросах. Что позволит обрабатывать таблицы с конфликтующими именами:

```elixir
posts_with_private = from p in Post, select: %{title: p.title, public: not p.private}
from p in subquery(posts_with_private), where: p.public, select: p
```

### Новая функция insert_all/3 в модуле Ecto.Repo

Функция `Ecto.Repo.insert_all/3` предназначена для множественной вставки записей внутри одного запроса:

```elixir
Ecto.Repo.insert_all Post, [%{title: "foo"}, %{title: "bar"}]
```

Стоит учесть, что при вставки строк через `insert_all/3` не обрабатываются автогенерируемые поля, такие как `inserted_at` или `updated_at`. Также `insert_all/3` даёт возможность вставлять строки в базу данных в обход `Ecto.Schema`, просто указав имя таблицы:

```elixir
Ecto.Repo.insert_all "some_table", [%{hello: "foo"}, %{hello: "bar"}]
```

### Добавлена Many-to-Many ассоциация

Теперь Ecto поддерживает `many_to_many` ассоциации:

```elixir
defmodule Post do
  use Ecto.Schema

  schema "posts" do
    many_to_many :tags, Tag, join_through: "posts_tags"
  end
end
```

Значение в опции `join_through` может быть именем таблицы, которая содержит в себе колонки `post_id` и `tag_id`, или может быть схемой, такой как `PostTag`, которая содержит внешние ключи и автогенерируемые колонки.

### Улучшена работа с ассоциациями

Отныне Ecto даёт возможность вставлять и изменять строки к `belongs_to` и `many_to_many` ассоциациям через `changeset`. Кроме этого, Ecto поддерживает определение ассоциаций напрямую в структуре данных для вставки. Например:

```elixir
Repo.insert! %Permalink{
  url: "//root",
  post: %Post{
    title: "A permalink belongs to a post which we are inserting",
    comments: [
      %Comment{text: "child 1"},
      %Comment{text: "child 2"},
    ]
  }
}
```

Это улучшение позволяет проще вставлять древовидные структуры в базу данных.

### Новая функция предзагрузки ассоциаций assoc/2 в модуле Ecto

Функция `Ecto.assoc/2` позволяет определять связи второго порядка, которые должны быть загружены к выбираемым записям. Как пример, можно получить авторов и комментарии к выбираемым публикациям:

```elixir
posts = Repo.all from p in Post, where: is_nil(p.published_at)
Repo.all assoc(posts, [:comments, :author])
```

Лучше загружать связи через ассоциации, поскольку это не требует добавления полей к схеме выборки.

### Upsert

Функции `Ecto.Repo.insert/2` и `Ecto.Repo.insert_all/3` начали поддерживать upserts (вставка и обновление) через опции `:on_conflict` и `:conflict_target`.

Опция `:on_conflict` определяет, как база данных должна себя вести в случае совпадения primary key.
Опция `:conflict_target` определяет по каким полям необходимо проверять конфликты при вставке новых строк.

```elixir
# Вставка оригинала
{:ok, inserted} = MyRepo.insert(%Post{title: "inserted"})

# Попытка вставки без вызова ошибки.
{:ok, upserted} = MyRepo.insert(%Post{id: inserted.id, title: "updated"},
                                on_conflict: :nothing)

# Попытка вставки, с обновлением поля title.
on_conflict = [set: [title: "updated"]]
{:ok, updated} = MyRepo.insert(%Post{id: inserted.id, title: "updated"},
                               on_conflict: on_conflict, conflict_target: :id)
```

### Новые условия выборки or_where и or_having

В Ecto добавили выражения `Ecto.Query.or_where/3` и `Ecto.Query.or_having`, которые добавляют новые фильтры к уже существующим условиям через "OR".

```elixir
from(c in City, where: [state: "Sweden"], or_where: [state: "Brazil"])
```

### Добавлена возможность пошагового построения запроса

Данная техника позволяет создавать выражения по крупицам, чтобы впоследствии интерполировать их в общий запрос.

Для примера, у вас есть набор условий, из которых вы хотите построить свой запрос, но нужно выбрать только некоторые из них в зависимости от контекста:

```elixir
dynamic = false

dynamic =
  if params["is_public"] do
    dynamic([p], p.is_public or ^dynamic)
  else
    dynamic
  end

dynamic =
  if params["allow_reviewers"] do
    dynamic([p, a], a.reviewer == true or ^dynamic)
  else
    dynamic
  end

from query, where: ^dynamic
```

В примере выше показано, как можно построить запрос шаг за шагом, учитывая внешние условия, и в конце интерполировать всё вовнутрь одного запроса.

Динамическое выражение всегда можно интерполировать внутри другого динамического выражения или внутри `where`, `having`, `update` или в условии `on` у `join`.

## Интерфейс запросов

Примеры запросов далее в тексте будут относиться к некоторым из этих моделей:

```elixir
Все модели используют id как первичный ключ, если не указано иное.

defmodule Showcase.Client do  
  use Ecto.Schema
  import Ecto.Query

  schema "clients" do
    field :name, :string
    field :age, :integer, default: 0
    field :sex, :integer, default: 0
    field :state, :string, default: "new"

    has_many :orders, Showcase.Order
  end

  # ...
end

defmodule Showcase.Order do  
  use Ecto.Schema
  import Ecto.Query

  schema "orders" do
    field :description, :text
    field :total_cost, :integer
    field :state, :string, default: "pending"

    belongs_to :client, Showcase.Client
    has_many :products, Showcase.Product
  end

  # ...
end
```

- [Получение одиночной строки](#1-Получение-одиночной-строки)
- [Получение нескольких строк](#2-Получение-нескольких-строк)
- [Условия выборки строк](#3-Условия-выборки-строк)
- [Сортировка строк](#4-Сортировка-строк)
- [Выбор определенных полей строк](#5-Выбор-определенных-полей-строк)
- [Группировка строк](#6-Группировка-строк)
- [Лимитирование и смещение выбираемых строк](#7-Лимитирование-и-смещение-выбираемых-строк)
- [Соединение таблиц](#8-Соединение-таблиц)
- [Переопределяющее условие](#9-Переопределяющее-условие)

Для получения объектов из базы данных Ecto предоставляет несколько функций поиска. В каждую функцию поиска можно передавать аргументы для выполнения определенных запросов в базу данных без необходимости писать на чистом SQL. Ecto предусматривает два стиля построения запросов: с использованием ключевого слова и через выражение (функцию/макрос).

Ниже перечислены некоторые выражения, предоставляемые Ecto, они объявлены в двух модулях:

Ecto.Repo:

- get/3
- get_by/3
- one/2
- all/3

Ecto.Query:

- where/3
- or_where/3
- order_by/3
- select/3
- group_by/3
- limit/3
- offset/3
- join/5
- exclude/2

### 1. Получение одиночной строки

#### get/3

Используя функцию `get/3`, можно получить запись, соответствующую определенному первичному ключу (primary key). Например:

```elixir
# Ищет клиента с первичным ключом (id) 10.
client = Ecto.Repo.get(Client, 10)
=> %Client{id: 10, name: "Cain Ramirez", age: 34, sex: 1, state: "good"}
```

Функция `get/3` возвратит `nil`, если ни одной записи не найдено. Если запрос не будет иметь `primary key` или их будет больше одного, то функция вызовет ошибку `argument error`.

Функция `get!/3` ведет себя подобно `get/3`, за исключением того, что она вызовет `Ecto.NoResultsError`, если не найдено ни одной соответствующей записи.

#### get_by/3

Используя функцию `get_by/3`, можно получить запись, соответствующую предоставленным условия выборки. Например:

```elixir
# Ищет клиента с именем (name) "Cain Ramirez".
client = Ecto.Repo.get_by(Client, name: "Cain Ramirez")
=> %Client{id: 10, name: "Cain Ramirez", age: 34, sex: 1, state: "good"}
```

Функция `get_by/3` возвратит `nil`, если ни одной записи не найдено.

Функция `get_by!/3` ведет себя подобно `get_by/3`, за исключением того, что она вызовет ошибку `Ecto.NoResultsError`, если не найдено ни одной соответствующей записи.

#### one/2

Используя функцию `one/2`, можно получить одну запись, соответствующую предоставленным условия выборки. Например:

```elixir
# Выбирает все записи модели Клиент.
query = Ecto.Query.from(c in Client, where: c.name == "Jean Rousey")
client = Ecto.Repo.one(query)
=> %Client{id: 1, name: "Jean Rousey", age: 29, sex: -1, state: "good"}
```

Функция `one/2` возвратит `nil`, если ни одной записи не найдено. И функция вызовет ошибку если по запросу найдётся больше одной записи.

Функция `one!/2` ведет себя подобно `one/2`, за исключением того, что она вызовет ошибку `Ecto.NoResultsError`, если не найдено ни одной соответствующей записи.

### 2. Получение нескольких строк

#### all/3

Используя функцию `all/3`, можно получить все записи, соответствующие предоставленным условиям запроса. Например:

```elixir
# Выбирает все записи модели Клиент.
query = Ecto.Query.from(c in Client)
clients = Ecto.Repo.all(query)
=> [%Client{id: 1, name: "Jean Rousey", age: 29, sex: -1, state: "good"}, ..., %Client{id: 10, name: "Cain Ramirez", age: 34, sex: 1, state: "good"}]
```

Функция `all/3` вернет `Ecto.QueryError`, если запрос не пройдёт валидацию.

### 3. Условия выборки строк

#### where/3

Выражение `where/3` позволяет определить условия для ограничения возвращаемых записей, которые представляет WHERE часть выражения SQL. В случае если передаётся несколько условий выборки, то они комбинируются оператором `AND`.

Вызов по ключевому слову `where:` является неотъемлемой частью макроса `from/2`:

```elixir
from(c in Client, where: c.name == "Cain Ramirez")
from(c in Client, where: [name: "Cain Ramirez"])
```

Есть возможность интерполировать списки с условиями выборки, что позволяет предварительно собрать необходимые ограничения.

```elixir
filters = [name: "Cain Ramirez"]
from(c in Client, where: ^filters)
```

Вызов макросом `where/3`:

```elixir
Client |> where([c], c.name == "Cain Ramirez")
Client |> where(name: "Cain Ramirez")
```

#### or_where/3

Выражение `or_where/3` позволяет определить более гибкие условия для ограничения возвращаемых записей, которые представляет WHERE часть выражения SQL. Разница между `where/3` и `or_where/3` минимальна, но принципиальна. Переданное условие присоединяется к уже существующим через оператор `OR`. В случае если передаётся несколько условий выборки в `or_where/3`, то между собой они комбинируются оператором `AND`.

Вызов по ключевому слову `or_where:` является неотъемлемой частью макроса `from/2`:

```elixir
from(c in Client, where: [name: "Cain Ramirez"], or_where: [name: "Jean Rousey"])
```

Есть возможность интерполировать списки с условиями выборки, что позволяет предварительно собрать необходимые ограничения. Условия в списке между собой объединяются через `AND`, и присоединятся к существующим условиям через `OR`:

```elixir
filters = [sex: 1, state: "good"]
from(c in Client, where: [name: "Cain Ramirez"], or_where: ^filters)
```

... данное выражение эквивалентно:

```elixir
from c in Client, where: (c.name == "Cain Ramirez") or
                         (c.sex == 1 and c.state == "good")
```

Вызов макросом `or_where/3`:

```elixir
Client |> where([c], c.name == "Jean Rousey")
       |> or_where([c], c.name == "Cain Ramirez")
```

### 4. Сортировка строк

Выражение `order_by/3` позволяет определить условие сортировки записей полученных из базы данных. `order_by/3` задаёт `ORDER BY` часть SQL запроса.

Есть возможность сортировать сразу по нескольким полям. Направление сортировки по умолчанию на возрастание (`:asc`), может быть переопределено на убывание (`:desc`). Для каждого поля можно задать своё направление сортировки.

Вызов по ключевому слову `order_by:` является неотъемлемой частью макроса `from/2`:

```elixir
from(c in Client, order_by: c.name, order_by: c.age)
from(c in Client, order_by: [c.name, c.age])
from(c in Client, order_by: [asc: c.name, desc: c.age])

from(c in Client, order_by: [:name, :age])
from(c in Client, order_by: [asc: :name, desc: :age])
```

Есть возможность интерполировать списки с полями сортировки, что позволяет предварительно собрать необходимые условия выборки.

```elixir
values = [asc: :name, desc: :age]
from(c in Client, order_by: ^values)
```

Вызов макросом `order_by/3`:

```elixir
Client |> order_by([c], asc: c.name, desc: c.age)
Client |> order_by(asc: :name)
```

### 5. Выбор определенных полей строк

Выражение `select/3` позволяет определить поля таблиц, которые необходимо вернуть для получаемых записей из базы данных. `select/3` задаёт `SELECT` часть SQL запроса. По умолчанию `Ecto` выбирает все множество полей результата, используя `select *`.

Вызов по ключевому слову `select:` является неотъемлемой частью макроса `from/2`:

```elixir
from(c in Client, select: c)
from(c in Client, select: {c.name, c.age})
from(c in Client, select: [c.name, c.state])
from(c in Client, select: {c.name, ^to_string(40 + 2), 43})
from(c in Client, select: %{name: c.name, order_counts: 42})
```

Вызов макросом `select/3`:

```elixir
Client |> select([c], c)
Client |> select([c], {c.name, c.age})
Client |> select([c], %{"name" => c.name})
Client |> select([:name])
Client |> select([c], struct(c, [:name]))
Client |> select([c], map(c, [:name]))
```

**Важно:** При ограничении полей выборки для ассоциаций важно выбирать внешние ключи связей, иначе `Ecto` не сможет найти связанные объекты.

### 6. Группировка строк

Для определения условия `GROUP BY` в SQL запросе, предназначен макрос `group_by/3`. Все столбцы упомянутые в `SELECT`, должны быть переданы в `group_by/3`. Это общее правило для агрегатных функций.

Вызов по ключевому слову `group_by:` является неотъемлемой частью макроса `from/2`:

```
from(c in Client, group_by: c.age, select: {c.age, count(c.id)})

from(c in Client, group_by: :sex, select: {c.sex, count(c.id)})
```

Вызов макросом `group_by/3`:

```
Client |> group_by([c], c.age) |> select([c], count(c.id))
```

### 7. Лимитирование и смещение выбираемых строк

Для определения LIMIT в SQL запросе используйте выражение `limit/3`, оно определит количество необходимых записей, которые будут получены.

Если `limit/3` передан дважды, то первое значение будет перекрыто вторым.

```
from(c in Client, where: c.age == 29, limit: 1)

Client |> where([c], c.age == 29) |> limit(1)
```

Для определения OFFSET в SQL запросе используйте выражение `offset/3`, оно определит количество записей, которые будут пропущены до начала возвращаемых записей.

Если `offset/3` передан дважды, то первое значение будет перекрыто вторым.

```
from(c in Client, limit: 10, offset: 30)

Client |> limit(10) |> offset(30)
```

### 8. Соединение таблиц

Часто запросы обращаются к нескольким таблицам, такие запросы строятся с использованием конструкции `JOIN`. В Ecto для определения такой конструкции предназначено выражение `join/5`. По умолчанию стратегией присоединения таблицы является INNER JOIN, которую можно переопределить на: `:inner`, `:left`, `:right`, `:cross` or `:full`. В случае построения запроса по ключу, `:join` можно заменить на: `:inner_join`, `:left_join`, `:right_join`, `:cross_join` или `:full_join`.

Вызов по ключевому слову `join:` является неотъемлемой частью макроса `from/2`:

```
from c in Comment,
  join: p in Post, on: p.id == c.post_id,
  select: {p.title, c.text}

from p in Post,
  left_join: c in assoc(p, :comments),
  select: {p, c}

from c in Comment,
  join: p in Post, on: [id: c.post_id],
  select: {p.title, c.text}
```

Все ключи переданные в `on` будут учтены как условия соединения.

Есть возможность интерполировать правую часть относительно `in`. Для примера:

```
posts = Post
from c in Comment,
  join: p in ^posts, on: [id: c.post_id],
  select: {p.title, c.text}
```

Вызов макросом `join/5`:

```
Comment
|> join(:inner, [c], p in Post, c.post_id == p.id)
|> select([c, p], {p.title, c.text})

Post
|> join(:left, [p], c in assoc(p, :comments))
|> select([p, c], {p, c})

Post
|> join(:left, [p], c in Comment, c.post_id == p.id and c.is_visible == true)
|> select([p, c], {p, c})
```

### 9. Переопределяющее условие

Ecto даёт возможность убрать уже определенные условия в запросе или вернуть значения по умолчанию, для этого используйте выражение `exclude/2`.

```
query |> Ecto.Query.exclude(:select)

Ecto.Query.exclude(query, :select)
```

## Команды для массовых операций со строками

### Массовая вставка

Функция `Ecto.Repo.insert_all/3` вставит все переданные записей.

```
Repo.insert_all(Client, [[name: "Cain Ramirez", age: 34], [name: "Jean Rousey", age: 29]])
Repo.insert_all(Client, [%{name: "Cain Ramirez", age: 34}, %{name: "Jean Rousey", age: 29}])
```

Функция `insert_all/3` не обрабатывает автогенерируемые поля, такие как `inserted_at` или `updated_at`.

### Массовое обновление

Функция `Ecto.Repo.update_all/3` обновит все строки подпавшие под условие запроса на переданные значения полей.

```
Repo.update_all(Client, set: [state: "new"])

Repo.update_all(Client, inc: [age: 1])

from(c in Client, where: p.sex < 0)
|> Repo.update_all(set: [state: "new"])

from(c in Client, where: p.sex > 0, update: [set: [state: "new"]])
|> Repo.update_all([])

from(c in Client, where: c.id < 10, update: [set: [state: fragment("?", new)]])
|> Repo.update_all([])
```

### Массовое удаление

Функция `Ecto.Repo.delete_all/2` удаляет все строки, подпавшие под условие запроса.

```
Repo.delete_all(Client)

from(p in Client, where: p.age == 0) |> Repo.delete_all
```

## Практические примеры

### Композиция запросов

```elixir
query = from p in App.Product, select: p

query2 = from p in query, where: p.state == "published"

App.Repo.all(query2)
```

### Функция для постраничного вывода

```elixir
defmodule Finders.Common.Paging do
  import Ecto.Query

  def page(query), do: page(query, 1)

  def page(query, page), do: page(query, page, 10)

  def page(query, page, per_page) do
    offset = per_page * (page-1)

    query |> offset([_], ^offset)
          |> limit([_], ^per_page)
  end
end
```

```
# With Posts: second page, five per page
posts = Post |> Finders.Common.Paging.page(2, 5) |> Repo.all

# With Tags: third page, 10 per page
tags = Tag |> Finders.Common.Paging.page(3) |> Repo.all
```

### Query.API

Операторы сравнения: `==`, `!=`, `<=`, `>=`, `<`, `>`
Булевы операторы: `and`, `or`, `not`
Оператор включения: `in/2`
Функции поиска: `like/2` и `ilike/2`
Проверка на *null*: `is_nil/1`
Агрегаторы: `count/1`, `avg/1`, `sum/1`, `min/1`, `max/1`
Функция для произвольных SQL подзапросов: `fragment/1`

```
from p in Post, where: p.published_at > ago(3, "month")

from p in Post, where: p.id in [1, 2, 3]

from p in Payment, select: avg(p.value)

from p in Post, where: p.published_at > datetime_add(^Ecto.DateTime.utc, -1, "month")

from p in Post, where: is_nil(p.published_at)

from p in Post, where: ilike(p.body, "Chapter%")

from p in Post,
  where: is_nil(p.published_at) and
         fragment("lower(?)", p.title) == "title"
```

## Дополнение из Ecto.Adapters.SQL

### `Ecto.Adapters.SQL.query/4`

Выполняет произвольный SQL запрос в рамках переданного репозитория.

```
Ecto.Adapters.SQL.query(Showcase, "SELECT $1::integer + $2", [40, 2])
=> {:ok, %{rows: [{42}], num_rows: 1}}
```

### `Ecto.Adapters.SQL.to_sql/3`

Конвертирует построенный из выражений запрос в SQL.

```
Ecto.Adapters.SQL.to_sql(:all, repo, Showcase.Client)
=> {"SELECT c.id, c.name, c.age, c.sex, c.state, c.inserted_at, c.created_at FROM clients as c", []}

Ecto.Adapters.SQL.to_sql(:update_all, repo,
                         from(c in Showcase.Client, update: [set: [state: ^"new"]]))
=> {"UPDATE clients AS c SET state = $1", ["new"]}
```
