# Отношения в JPA. Ленивая загрузка. Проблема N+1

Сегодня постараемся разобраться, какие возможности JPA предоставляет для работы со связанными сущностями -
имплементацией на уровне виртуальной БД концепции PK-FK. А также с тем, какие ограничения и недостатки это несет.

## Общая идея

В реализации связей между сущностями JPA продолжает исходить из тех же вводных, что и в самой концепции сущностей -
манипуляция данными максимально абстрагирована от явных запросов к БД и строится вокруг взаимодействия с entity-классами
как с обычными кассами моделей.

## Java API

Связи между сущностями в JPA регламентируются через набор соответствующих аннотаций для полей Entity-классов. Ниже 
разберем как основные аннотации, так и некоторую специфику по работе с ними.

При этом заранее стоит понимать, что именно механизм работы со связанными сущностями - ключевой для любой ORM. Ведь 
именно здесь раскрывается переход между объектными представлениями данных и реляционной природой этих данных. 
Качество и удобство этого механизма - визитная карточка ORM. Ведь без него любая ORM по сути сводится к 
динамическому маппингу данных и функционально мало чем отличается от не-ORM библиотек для работы с БД.

### @ManyToOne

Начнем с простого. Наиболее распространенное отношение между таблицами БД - Many-to-One. В классическом представлении, 
несколько строк одной таблицы могут ссылаться на одну и ту же строку в другой таблице.

Ключевые аннотации JPA для реализации связей между сущностями повторяют название конкретной связи, что видно из 
заголовка пункта.

В качестве примера определим таблицу `car`, которую будем связывать с таблицей `person`. В примерах ниже 
используются эти же таблицы, меняется лишь организация связей.

```sql
create table car (
    id          bigserial       primary key,
    number      varchar         not null,
    fk_person   bigint          not null references person(id)
);
```

Entity-класс:

```java
@Entity
@Table(name = "car")
public class CarEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    @Column(nullable = false)
    private String number;

    @ManyToOne
    @JoinColumn(name = "fk_person")
    private PersonEntity owner;
    
    // Getter'ы и Setter'ы
}
```

Здесь нас интересуют сразу две аннотации: `@ManyToOne` и `JoinColumn`.

`@ManyToOne` - аннотация для непосредственного указания типа связи над полем, которое с этой связью ассоциируется. 
Для этого типа связи классическим можно считать подход, в котором вместо поля, аналогичного колонке с FK мы 
используем поле entity-типа, ассоциируемое со строкой (точнее, с ее отражением в виде entity-объекта) в связанной 
таблице. Так, вместо поля для колонки `fk_person` мы сразу объявляем поле для `PersonEntity`.

> Описанный выше подход не единственный возможный. Как минимум, у него есть очевидный недостаток: получение значения 
> `fk_person` становится возможным только через обращение к дополнительному объекту:
> 
> ```java
> var ownerId = car.getOwner().getId();
> ```
> 
> В более глобальном смысле поле `owner` не является отражением колонки `fk_person` как таковой - это не очевидно, но 
> станет заметно при разборе следующих типов связей, в которых похожие поля будут, а вот аналогичных колонок 
> может и не быть.
> 
> Я предпочитаю следующий формат записи M2O:
> 
> ```java
> @JoinColumn(name = "fk_person")
> @ManyToOne(fetch = FetchType.EAGER)
> private PersonEntity owner;
> 
> @Column(name = "fk_person", nullable = false, updatable = false, insertable = false)
> private Long ownerId;
> 
> public void setOwner(PersonEntity owner) {
>     this.owner = owner;
>     this.ownerId = owner == null ? null : owner.getId();
> }
> 
> // Getter'ы для указанных полей
> ```
> 
> Тогда получение `ownerId` становится намного проще:
> ```java
> var ownerId = car().getOwnerId();
> ```
> 
> Единственный нюанс - setter для поля `ownerId` лучше не добавлять - это может сломать логику виртуальной БД, 
> поскольку появляется возможность добавления `id` без связи. И в зависимости от ситуации, проявится эта проблема 
> может по-разному, не всегда в виде очевидной ошибки. 

Также в примере выше мы видим еще одну новую для нас аннотацию - `@JoinColumn`. Она во многом повторяет семантику 
`@Column`, но привносит ряд новых атрибутов, актуальных именно для описания различных типов связей. В данном случае 
она необходима только для указания имени колонки, по которой должен производиться маппинг.

Теперь, когда с содержимым нового Entity-класса стало чуть понятнее, можно рассмотреть, в каком виде возможно 
взаимодействие со связанными сущностями в коде:

```java
void updateRelatedEntity(EntityManager em) {
    em.getTransaction().begin();
    var car = em.find(CarEntity.class, 1L);

    car.getOwner().setAge(20);

    em.getTransaction().commit();
}
```

Так, мы, например, можем получать доступ к связанной сущности через геттер. Более того, в пределах транзакции мы 
можем обновлять связанную сущность, не запрашивая ее из БД напрямую - запрос к БД, конечно, произойдет, но это будет 
скрыто внутри JPA и не потребует какого-то дополнительного кода с нашей стороны. 

Также мы можем и заменить связанную сущность на другую:

```java
void replaceRelatedEntity(EntityManager em) {
    em.getTransaction().begin();
    var car = em.find(CarEntity.class, 1L);
    var person = em.find(PersonEntity.class, 2L);

    car.setOwner(person);

    em.getTransaction().commit();
}
```

Несколько интересней обстоит ситуация, если мы хотим создать нового владельца для машины, которого еще нет в БД. 
Если попытаться это сделать на текущем этапе примерно в таком виде:

```java
var car = em.find(CarEntity.class, 1L);
var person = new PersonEntity();
// Установка значений полей для person

car.setOwner(person);
```

Мы получим заслуженную ошибку - `person` не находится под управлением EM и не может быть сохранен в БД. В целом, 
разумным шагом будет вызвать `EntityManager#persist()` для нового объекта и все заработает. Но JPA предоставляет и 
другие решения для описанной проблемы. Пользоваться ими без особой необходимости (или достаточной осознанности) не 
стоит из-за ряда побочных эффектов, которые с ними связаны, но познакомиться можно уже сейчас, вместе с разбором
атрибутов `@ManyToOne`:

- `cascade`. Собственно, решение проблемы выше. Этот атрибут регламентирует для EM правила работы со связанным 
  entity-объектом. С помощью `cascade` можно указать, какие действия с основной сущностью (в нашем случае - `car`) 
  должны применяться по отношению к связанной (`person`). По умолчанию - EM воспринимает связанную сущность как
  полностью самостоятельную. Соответственно, требует для нее явных вызовов `persist()`, `remove()` и иных методов, 
  связанных с управлением жизненным циклом. Если же в `cascade` указаны какие-либо значения (см. `CascadeType`), EM 
  будет применять указанные операции к связанному entity-объекту автоматически, если они выполняются для основного 
  entity-объекта; 
- `targetEntity`. Позволяет указать класс, который должен быть использован для маппинга связанной сущности. По 
  умолчанию используется тип поля, что достаточно для большинства ситуаций. Но в специфичных сценариях, например, 
  при сложных иерархиях наследования entity поведения по умолчанию может быть недостаточно и могут возникать
  ситуации, при которых в типе поля указан суперкласс, но маппинг должен производиться в класс-наследник. Подробнее 
  разберем эту идею при знакомстве с наследованием в JPA; 
- `fetch`. Способ получения связанной сущности из БД. Вероятно, самый востребованный атрибут для любого типа связи. 
  Позволяет выбрать, загружать ли связанную сущность из БД одновременно с основной (`FetchType.EAGER`) или же только 
  при явном обращении к связанной сущности (`FetchType.LAZY`). Поведение по умолчанию для `@ManyToOne` - `EAGER`. 
  Более подробно с механизмом ленивой загрузки познакомимся в одном из следующих пунктов статьи; 
- `optional`. Указывает, может ли связь отсутствовать. Фактически эквивалентно наличию или отсутствию `NOT NULL` для 
  колонки с FK. Значение по умолчанию `true` - связь опциональна, т.е. не обязательна. 

Заодно стоит подробнее рассмотреть и набор атрибутов для `@JoinColumn`. Большая часть из них нам знакома из 
`@Column` и поэтому опущена, остаются лишь три:

- `referencedColumnName`. Позволяет указать имя колонки, на базе которой должен происходить поиск связанных 
  сущностей. Актуально для ситуаций, когда внешним ключом связанной таблицы выступает не PK текущей. По умолчанию в 
  качестве значения этого атрибута JPA использует колонку, ассоциированную с полем, аннотированное `@Id`;
- `table`. Позволяет указать целевую таблицу для связи. В большинстве случаев связь очевидна из типа поля, однако 
  для специфических сценариев наследования или при использовании **secondary table** (еще более специфичный 
  инструмент в рамках JPA, не затрагиваемый в курсе) может потребоваться явное указание имени таблицы;
- `foreignKey`. Позволяет описать особенности оформления FK для кодогенерации. В первую очередь - указать, 
  необходимо ли создание constraint'а на уровне БД. Или же связь между сущностями должна остаться исключительно 
  логической, без дополнительной валидации со стороны СУБД. Обычно этот атрибут не требуется.

> Если по какой-то причине нужно построить связь между сущностями, в которой JOIN-условие содержит несколько колонок -
> стоит обратиться к аннотации-контейнеру `@JoinColumns`, которая позволяет описать внутри себя несколько отдельных 
> `@JoinColumn`.

Также стоит отметить, что хоть классический сценарий подразумевает определение связей через поля сущности, 
технически аннотировать разбираемыми в статье аннотации можно и методы - чаще всего, getter'ы. В большинстве 
сценариев разницы в поведении не будет, а выбор в пользу такого подхода объясняют в основном идеологическими причинами.

### @OneToOne

Связь, во многом похожая с предыдущей. С точки зрения СУБД отличие между M2O и O2O заключается только в 
использовании уникального индекса на FK в последнем случае. Чтобы гарантировать, что со строкой в одной таблице 
будет связана лишь одна строка в другой.

С точки зрения JPA разница между `@ManyToOne` и `@OneToMany` тоже невелика. Единственное существенное отличие - 
атрибут `mappedBy`, который позволяет сущности получать доступ к связанной в условии, когда FK физически находится 
именно в связанной таблице.

> Разные аннотации имеют различное значение по умолчанию для атрибута `fetch`. Каждый раз дублировать назначение 
> атрибута не имеет смысла, но стандартные значения лучше помнить. Здесь - все еще `EAGER`. 

Рассмотрим на примере машины и владельца. Допустим, мы ввели ограничение, по которому у человек может быть максимум 
одна машина (коммунизм через пятилетку, а пока - уравниловка):

```sql
create unique index ix_unq_car_fk_person on car(fk_person);
```

Необходимо адаптировать сущность `CarEntity` к новым вводным:

```java
@Entity
@Table(name = "car")
public class CarEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    @Column(nullable = false)
    private String number;

    @OneToOne
    @JoinColumn(name = "fk_person")
    private PersonEntity owner;
    
    // Getter'ы и Setter'ы
}
```

Также мы можем отобразить эту связь и в `PersonEntity`:

```java
@Entity
@Table(name = "person")
public class PersonEntity {
    // Ранее добавленные поля

    @OneToOne(mappedBy = "fk_person")
    private CarEntity car;

    // Getter'ы и Setter'ы
}
```

Также эта связь примечательна тем, что на базе нее можно продемонстрировать одно из уместных использований 
`EntityManager#flush()` в явном виде:

```java
void replaceCar(EntityManager em) {
    em.getTransaction().begin();
    var car = em.find(CarEntity.class, 1L);
    var person = car.getOwner();
    person.setCar(null); // Из-за особенностей реализации односторонних One-to-one приходится явно обнулять связь 

    em.remove(car);

    var newCar = new CarEntity();
    newCar.setNumber("NEW-CAR-NUMBER");
    newCar.setOwner(person);

    em.persist(newCar); // Ошибка уникального индекса: запись по такому ключу уже существует 

    em.getTransaction().commit();
    }
```

В коде выше все кажется логичным, пока не появляется исключение. Почему? Потому что `EntityManager#remove()` не 
удаляет сущность в БД сразу при вызове. Зато `EntityManager#persist()` для указанной стратегии генерации `id` 
производит `INSERT` в БД.

Впрочем, ошибка произошла бы при любой стратегии. Разница лишь в том, что произошла бы при фиксации транзакции, а не 
на вызове `persist()`. Как упоминалось в одной из предыдущих статей, JPA в своей внутренней логике сначала добавляет 
новые строки, а лишь затем удаляет старые.

Решение может состоять в добавлении `flush()` после вызова `remove()`. Ведь тогда JPA сообщит БД все изменения, 
которые существуют в текущем EM к этому моменту:

```java
void replaceCar(EntityManager em) {
    em.getTransaction().begin();
    var car = em.find(CarEntity.class, 1L);
    var person = car.getOwner();
    person.setCar(null); 

    em.remove(car);
    em.flush();

    var newCar = new CarEntity();
    newCar.setNumber("NEW-CAR-NUMBER");
    newCar.setOwner(person);

    em.persist(newCar); 

    em.getTransaction().commit();
    }
```

В таком виде все работает корректно, старая машина удаляется, новая добавляется и связывается с пользователем. 
Единственный нюанс - объект `person` так и не узнает о новой привязанной машине. Для исправления ситуации придется 
вызвать `refresh()` после фиксации транзакции или же (если транзакция должна продолжить выполнение) сделать еще один 
явный `flush()` после добавления новой машины и так же `refresh()` для `person`. Это одна из 
относительно распространенных ситуаций, когда может потребоваться явный вызов `refresh()`. 

### @OneToMany

One-to-Many - по сути является тем же Many-to-One, но с точки зрения "one"-сущности. Соответственно, для примера 
откатим изменения для предыдущего пункта и вновь позволим заводить несколько машин одному человеку. Иными словами, 
долой уравниловку, да и коммунизм - как-то не по-нашему.

Тем не менее мы хотим получить возможность через человека просматривать все его машины. Поэтому добавляем связь для 
`PersonEntity`:

```java
@OneToMany(mappedBy = "owner")
private List<CarEntity> cars = new ArrayList<>();

public List<CarEntity> getCars() {
    return cars;
}
```

Чтобы не переживать о возможном `NullPointerException`, рекомендую сразу прописывать для поля с O2M инициализацию по 
умолчанию. А вот сеттер для него может оказаться неудачной затеей - если у сущности уже есть связи, явный сет коллекции
может иметь слишком много побочных эффектов, в зависимости от бизнес-логики и настроек самой связи. Лучше использовать 
`List#add()` или `List#addAll()`.

> В целом, не обязательно One-to-Many должно выражаться в виде поля типа `List`. Вполне подойдет и `Set`, если это 
> укладывается в логику обработки.

При стандартных настройках O2M-поле позволяет лишь получать информацию о существующих связанных entity-объектах и 
обновлять их. Если же нам нужно позволить работать с добавлением связанных объектов - необходимо указывать это через 
настройки `cascade` у `@OneToMany` или же явно помещать добавляемые объекты под управление EM. Тогда станет актуален 
упомянутые выше методы для добавления в коллекцию.

Но удалить связанную сущность из БД, лишь удалив ее из коллекции, не получится даже с настройкой `cascade`. Это 
достаточно специфическая операция для поведения по умолчанию, ибо существует масса ситуаций, когда логика такого 
удаления ограничена или конфликтует с другими частями JPA. Однако если очень хочется, то можно* - достаточно 
установить атрибут `orphanRemoval` у `@OneToMany` в значение `true`

> *Несмотря на достаточно широкие возможности JPA по манипуляции связанными сущностями через основные, я рекомендую 
> избегать подобного подхода, где это возможно. Как минимум на начальных этапах.
> 
> Для того чтобы использовать его эффективно (или хотя бы не стрелять себе в ногу), требуется достаточно хорошо 
> понимать, как именно JPA взаимодействует с entity-объектами, какие ограничения и side-эффекты влекут те или иные
> продвинутые операции. Не зря по умолчанию `orphanRemoval` установлен в `false`, а каскадные операции выключены.

Пример удаления старых и добавления новых машин через связь (считаем, что для `cascade` в `@OneToMany` установлено 
значением `CascadeType.ALL` или хотя бы `CascadeType.PERSIST`, `orphanRemoval` - `true`):

```java
void addCars(EntityManager em) {
    em.getTransaction().begin();

    var person = em.find(PersonEntity.class, 1L);

    // Удаляем все машины, связанные с текущей персоной
    person.getCars().clear();
    
    var newCar1 = new CarEntity();
    newCar1.setNumber("NEW-CAR-NUMBER1");
    // Указывать связь нужно явно, даже если дальнейшее управление сущностью EM 
    //  получит через каскад для PersonEntity#cars
    newCar1.setOwner(person);

    // Если бы не настройка cascade для связи, здесь пришлось бы явно вызывать em.persist(newCar1) и лишь затем 
    //  добавлять сущность в коллекцию
    person.getCars().add(newCar1);

    var newCar2 = new CarEntity();
    newCar2.setNumber("NEW-CAR-NUMBER2");
    newCar2.setOwner(person);

    person.getCars().add(newCar2);

    em.getTransaction().commit();
}
```

> Атрибут `fetch` для `@OneToMany` по умолчанию имеет значение `LAZY`. То есть связанные сущности не запрашиваются 
> из БД без явного обращения к полю.

### @ManyToMany

Оставшаяся классическая связь - Many-to-Many. Она потребует введение дополнительной промежуточной таблицы.

Допустим, что мы считаем возможным наличие нескольких владельцев у машины. Также и у одного человека может быть 
несколько машин:

```sql
create table person_car(
   fk_person        bigint not null references person (id),
   fk_car           bigint not null references car (id),
   unique (fk_person, fk_car)
);
```

```java
@Entity
@Table(name = "car")
public class CarEntity {
    // Ранее добавленные поля

    @ManyToMany
    @JoinTable(name = "person_car",
            joinColumns = @JoinColumn(name = "fk_car"),
            inverseJoinColumns = @JoinColumn(name = "fk_person"))
    private List<PersonEntity> owners = new ArrayList<>();

    // Getter'ы и Setter'ы
}
```

```java
@Entity
@Table(name = "person")
public class PersonEntity {
    // Ранее добавленные поля

    @ManyToMany
    @JoinTable(name = "person_car",
            joinColumns = @JoinColumn(name = "fk_person"),
            inverseJoinColumns = @JoinColumn(name = "fk_car"))
    private List<CarEntity> cars = new ArrayList<>();

    // Getter'ы и Setter'ы
}
```

В данном случае мы реализовали двустороннюю связь, при которой добавление новых и удаление старых связей возможно с 
каждой из сторон Many-to-Many. Взаимодействие в части добавления и удаление связей аналогично добавлению и удалению в
One-to-Many. За исключением атрибута `orphanRemoval` - здесь он отсутствует и удаление доступно по умолчанию. Ведь 
мы удаляем не сущность, а лишь запись в промежуточной таблице со связями.

Тем не менее, описанный выше подход нельзя назвать распространенным. Чаще всего M2M связь реализуется как 
односторонняя - лишь у одной из сущностей есть необходимость знать о связи. Это не означает, что для второй сущности 
эту связь нельзя сделать, просто часто в этом нет необходимости в силу внутренней логики приложения.

Однако если связь действительно необходима с двух сторон, стоит задуматься о владельце этой связи. Ведь если связь 
можно изменять (добавлять новые или удалять старые связи между строками) с каждой стороны - всегда есть риск, что 
эти изменения окажутся несогласованными. Поэтому зачастую в таких ситуациях для `@ManyToMany` определяют атрибут 
`mappedBy`. Его наличие означает, что текущая сущность использует связь из связанного entity-класса и связь доступна 
ей только на чтение - изменения коллекции не будут иметь значения для БД.

Разберем на примере:

```java
void replaceCars(EntityManager em) {
    em.getTransaction().begin();

    var person = em.find(PersonEntity.class, 1L);

    // Действие 1 - удаляем владение первой машиной у выбранной персоны
    person.getCars().removeFirst();

    // Действие 2 - получаем машину и добавляем ей второго владельца
    var existsCar = person.getCars().getFirst();
    var person2 = em.find(PersonEntity.class, 2L);
    existsCar.getOwners().add(person2);

    var car = new CarEntity();
    car.setNumber("MY-CUSTOM-NUMBER");
    
    em.persist(car);
    // Действие 3 - добавляем новую машину
    person.getCars().add(car);

    em.getTransaction().commit();
    }
```

При текущем описании связей будут выполнены все три действия. Но если мы решим, что "владелец" связи только 
`PersonEntity`, то необходимо доработать описание `CarEntity`:

```java
@Entity
@Table(name = "car")
public class CarEntity {
    // Ранее добавленные поля

    @ManyToMany(mappedBy = "cars")
    private List<PersonEntity> owners = new ArrayList<>();

    // Getter'ы и Setter'ы
}
```

Теперь `CarEntity` может использовать связь только для чтения. Все изменения связей через объекты `CarEntity` будут 
проигнорированы. А значит в примере выше будут применены только действия 1 и 3.

Несложно заметить, что кроме `@ManyToMany` появилась еще одна новая аннотация - `@JoinTable`. Как понятно из 
названия, она необходима для описания промежуточной таблицы, которая не является самостоятельной сущностью, но 
используется в качестве хранилища связей. Чаще всего используется именно для M2M, но технически никто не запрещает 
использовать ее для O2M или M2O связей, если по каким-то причинам недопустимо добавление FK в одну из сущностей.

Поскольку семантически `@JoinTable` отвечает и за связи, и за таблицу, ее атрибуты выглядят как причудливая 
композиция из атрибутов, характерных для `@JoinColumn` и `@Table`:

- `name`. Имя промежуточной таблицы;
- `catalog`. Каталог, в котором хранится промежуточная таблица. Неактуально для PostgreSQL;
- `schema`. Схема, в которой хранится промежуточная таблица;
- `joinColumns`. Массив колонок, обеспечивающих FK для текущей сущности. В нашем случае колонка лишь одна - 
  `fk_person`. Если бы связь обеспечивалась не по PK или сам PK был композитным - было бы актуально указать 
  несколько колонок;
- `inverseJoinColumns`. То же, что и `joinColumns`, то для связанной сущности;
- `foreignKey`. Играет ту же роль, что и одноименный атрибут в `@JoinColumn`;
- `inverseForeignKey`. Аналогичен `foreignKey`, но применяется для FK для связанной (по отношению с владельцу связи) 
  сущности. Например, если владельцем связи является `PersonEntity`, в этом атрибуте можно описать FK для `CarEntity`;
- `uniqueConstraints`. Играет ту же роль, что и одноименный атрибут в `@Table`. При выключенной генерации DDL 
  является исключительно информационным;
- `indexes`. Играет ту же роль, что и одноименный атрибут в `@Table`;

Сама по себе связь M2M достаточно сложная и в восприятии, и в поддержке. Это находит выражение и в JPA. Однако при 
четком понимании бизнес-необходимости и требуемых для управления связью возможностей, все может быть реализовано 
достаточно эффективно.

### @ElementCollection и @CollectionTable

Указанные в заголовке аннотации обеспечивают более специфические связи между таблицами. С точки зрения отношений 
между таблицами, обычно это будет описание O2M или M2M связей, где связанный объект не является JPA Entity.

В общем смысле это может означать, что участвующая в связи таблица не находится под контролем JPA, но это очень 
узкая область, которая не имеет смысла за пределами специфики конкретного проекта.

С точки зрения более широго использования, эти аннотации могут быть полезны в случаях, когда нам нет необходимости 
получать всю связанную сущность целиком, а достаточно получить лишь отдельный атрибут или набор атрибутов. Чаще 
всего имеет смысл получение идентификаторов связанных entity-объектов без получения объектов целиком.

Например, у меня есть M2M между `CarEntity` и `PersonEntity`. При этом в пределах задач для `CarEntity` мне не нужны 
данные о владельцах - достаточно лишь знать идентификаторы владельцев. Тогда вместо использования `@ManyToMany` я 
могу описать нечто подобное:

```java
@Entity
@Table(name = "car")
public class CarEntity {
    // Ранее добавленные поля

    @ElementCollection
    @CollectionTable(name = "person_car",
            joinColumns = @JoinColumn(name = "fk_car"))
    @Column(name = "fk_person", insertable = false, updatable = false)
    private List<String> ownerIds = new ArrayList<>();

    // Getter'ы и Setter'ы
}
```

Аннотация `@ElementCollection` указывает на необходимость отображения связанных элементов в виде коллекции. Обычно 
используется `Set` или `List`.

`@CollectionTable` описывает правила определения связи. Атрибутивный состав напоминает `@JoinTable`, но без 
inverse-атрибутов - так как связанная таблица не рассматривается как промежуточная, в них нет смысла.

Наконец, `@Column` описывает колонку из связанной таблицы, а не из текущей. Это актуально для ситуаций, когда через 
связь извлекается лишь одна колонка. Если извлекается набор атрибутов* - все несколько сложнее, но для более 
подробного описания стоит сначала познакомиться с концепцией наследования в JPA.

> *Учитывая редкость подобных сценариев, связь наследования в JPA и `@ElementCollection` лучше отложить до времен, 
> когда в подобной реализации действительно появится необходимость.

На этом мы завершаем знакомство со связями между сущностями. И переходим к смежным концепциям, которые не менее 
важны для эффективного использования JPA.

## Ленивая загрузка

Выше был многократно упомянут атрибут `fetch` для аннотаций, декларирующих связи между сущностями. Для нас этот атрибут
открывает обширную тему ленивой загрузки в JPA.

Ленивая загрузка - механизм отложенного получения данных о сущностях, при котором связанные объекты загружаются не 
сразу при получении основной сущности (как это происходит при `EAGER`-загрузке), а лишь при явном обращении к полю 
со связью.

Актуальность механизма достаточно очевидна - сущность может иметь множество связей, сами связи (особенно O2M и M2M) 
могут содержать множество объектов. И вполне логично, что далеко не всегда все связанные сущности нужны. А если они 
не нужны - зачем тратить ресурсы на их получение и хранение в памяти?

Ранее также упоминалось, что для Entity-классов Hibernate и другие JPA-провайдеры используют прокси-объекты. Одна из 
основных причин, по которым нужно проксирование - как раз обеспечение ленивой загрузки. Ведь фактически необходимо 
при вызове getter'а (или ином обращении к полю) производить запрос в базу данных, что не реализуемо без 
прокси-объекта,
[декоратора](https://ru.wikipedia.org/wiki/%D0%94%D0%B5%D0%BA%D0%BE%D1%80%D0%B0%D1%82%D0%BE%D1%80_(%D1%88%D0%B0%D0%B1%D0%BB%D0%BE%D0%BD_%D0%BF%D1%80%D0%BE%D0%B5%D0%BA%D1%82%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D1%8F))
или иных синтаксически родственных подходов.

На данном этапе синтаксисом для ленивой загрузки для нас остается только уже упомянутый атрибут аннотаций. В 
дальнейшем мы познакомимся и с другими способами настройки этого инструмента - через `EntityGraph`, JPQL и т.д.

В целом, ленивая загрузка - решение по умолчанию для большинства сценариев использования. И этот подход часто 
выбирают даже для O2O и O2M связей, где JPA по умолчанию предлагает `EAGER`-загрузку.

### LazyInitializationException

Но стоит понимать, что `LAZY`-загрузка, как и любые другие сложные механизмы, имеет свои ограничения и недостатки.

Наиболее очевидный из них - необходимость поддерживать соединение с БД. Ведь если нет связи с базой данных - нет 
возможности и получить из нее связанные сущности. Так, при попытке обратиться к ранее не инициализированному полю с 
`LAZY`-загрузкой после закрытия `EM`, Hibernate выбросит исключение `LazyInitializationException`. Ведь нам 
необходимо сделать запрос к БД, но у нас уже нет доступа к БД.

> В этом разделе я стараюсь не углубляться в детали реализации отдельных JPA-провайдеров. Но для описываемого 
> механизма поведение может разниться от реализации. Например, `LazyInitializationException` - исключение, 
> определяемое именно Hibernate, а не JPA.

Эта проблема также имеет решение - например, через настройку параметра `hibernate.enable_lazy_load_no_trans`. При 
установке значения в `true` Hibernate будет открывать новую временную сессию (`Session` - имя реализации EM в 
Hibernate) для получения данных, если активной сессии не найдено. Но каждая открытая сессия - это получение 
Connection'а, что является дополнительными накладными расходами. Особенно, если после закрытия EM приходится 
загружать данные из множества связей - каждый такой запрос будет требовать получения нового Connection'а. Отчасти 
это нивелируется использованием Connection Pool, но все еще является более дорогим (и менее управляемым) решением, 
нежели загрузка в пределах основной сессии.

## Проблема N+1

Проблема N+1 - обобщенное описание для ряда сценариев, при которых получение данных разбивается на множество 
маленьких запросов вместо одного большого.

Обычно эта проблема связана с раздельной загрузкой основной сущности и ее связей. Получается, что вместо одного 
общего запроса с JOIN'ами, выполняется несколько более мелких и простых обращений к БД. Но по совокупности множество 
запросов тратит больше ресурсов, нежели один тяжелый запрос.

Строго говоря, эта проблема актуальна независимо от способа взаимодействия с БД и может проявляться даже при 
использовании JDBC. Но традиционно ее рассматривают с точки зрения работы через JPA, поскольку непрозрачная 
коммуникация с БД делает непрозрачной и саму проблему - в отличие от того же JDBC, далеко не всегда то, что нам 
кажется одним запросом, является одним запросом на самом деле.

Простой пример, в котором проявляется N+1 и проясняется название термина: допустим, у нас есть сущность "Машина" и 
связанная с ней по M2O (как в начале статьи) сущность "Персона". Мы хотим получить список имен владельцев машин без 
дубликатов.

Для большей наглядности будем считать, что `fetch` у связи установлен в значение `LAZY`. Для `EAGER` существует 
ровно та же проблема, но проявляется она менее наглядно.

Итак, решение поставленной задачи может выглядеть примерно так:

```java
List<String> getCarOwnerNames(EntityManager em) {
    List<CarEntity> cars = em.createNativeQuery("select * from car", CarEntity.class)
            .getResultList();
    
    return cars.stream()
            .map(CarEntity::getOwner) // Запрос к БД для получения связанной сущности
            .map(PersonEntity::getName)
            .distinct()
            .toList();
}
```

Получается, что для получения итоговой информации потребовалось:

- Сделать один запрос для получения данных по основной сущности;
- Сделать N запросов для получения связанных сущностей, где N - количество объектов, полученных в результате первого 
  запроса.

Итого потребовалось N+1 запросов, чтобы собрать всю информацию.

При этом стоит понимать, что `EAGER`-загрузка проблему никак не исправит - запросов будет ровно столько же, только 
получение связанных сущностей будет выполняться неявно, сразу после получения основных. А ведь у связанных сущностей 
могут быть свои связи. И если `fetch` для них установлен в `EAGER` - они тоже будут сразу запрашиваться из БД. В 
результате банальное получение записей даже для одной таблицы может превратиться в целую плеяду запросов.

Ситуация усугубляется тем, что внешнему наблюдателю сложно отловить проблему штатными средствами мониторинга.
Мы можем наблюдать за нагрузкой на базу данных, можем отслеживать, что она избыточно высокая. Но отловить проблему не
сможем. Просто потому что нагрузка производится не какими-то отдельными тяжелыми запросами (классическая проблема, с
которой к разработчику может прийти DBA или другой специалист, отвечающий за мониторинг), а массой маленьких, быстро
выполняемых (по отдельности) запросов.

Глобальное решение N+1 состоит в ответственном подходе к взаимодействию с БД как в разрезе кода приложения, так и в 
части корректной настройки JPA.

Отдельные проявления можно решать различными инструментами - по мере дальнейшего знакомства с JPA я буду их 
подсвечивать отдельно. На данном этапе наиболее простым вариантом выглядит следующий:

```java
List<String> getCarOwnerNames(EntityManager em) {
    List<CarEntity> cars = em.createNativeQuery("select * from car", CarEntity.class)
            .getResultList();

    var ownerIds = cars.stream()
            // Одна из причин, почему я предпочитаю выделять сам FK в отдельное поле
            .map(CarEntity::getOwnerId)
            .toList();

    return em.createNativeQuery("select full_name from person where id in (:ids)", String.class)
            .setParameter("ids", ownerIds)
            .getResultList()
            .stream()
            .distinct()
            .toList();
    }
```

В текущем виде мы сократили количество запросов до константного числа - до двух. Технически, именно в данном случае 
можно обойтись и одним запросом с `JOIN`, но тогда в решении мало что осталось бы от JPA. В большинстве случаев не 
требуется сводить N+1 к одному запросу - такой запрос может оказаться избыточно громоздким и тяжелым в поддержке. 
Достаточно свести количество запросов до константного значения, не зависящего от числа извлекаемых из БД строк.

Подводя итог, стоит отметить, что далеко не всегда проблема наглядно видна из кода, в котором происходит обращение к 
БД. Иногда проблема лежит на уровне Entity-классов и их некорректной конфигурации, иногда находится под капотом JPA 
и не очевидна до углубленного изучения отдельного блока логики. В результате N+1 становится одной из наиболее 
распространенных проблем с производительностью у приложений, использующих или иные непрозрачные инструменты 
коммуникации с БД. 

#### С теорией на сегодня все!

![img.png](../../../commonmedia/defaultFooter.jpg)

Переходим к практике:

## Задача

Доработайте приложение, разработанное на базе практики к
[статье 156](https://github.com/KFalcon2022/lessons/blob/master/lessons/orm-and-jpa/156/EntityManagerFactory.%20EntityManager.%20Entity%20lifecycle.md).
Необходимо:

1. Добавить логику, реализованную в практике к статьям 148-152, переписав взаимодействие с пользователями через JPA;
2. Добавить атрибут "Владельцы" для машин, связав машины и пользователей. Пользователю может принадлежать несколько
   машин, а каждая машина может принадлежать нескольким пользователям;
3. Добавить сущность "Модель автомобиля", связав машину с моделью. Разные машины могут быть одной и той же модели;
4. Добавить сущность "Марка автомобиля", связав модели и марки. Марка должна знать обо всем модельном ряде, модель
   должна ссылаться на марку. Таким образом нужно описать двустороннюю связь: Марка - модели, модель - марка;
5. Реализовать HTTP API для получения всех моделей автомобиля по марке, получения всех автомобилей текущего
   пользователя;
6. Реализовать HTTP API для добавления или редактирования информации о машинах пользователя, о возможных марках и
   моделях автомобилей.

Ветка для PR: [for-pr](https://github.com/KFalcon2022/car-jpa-practical-task/tree/for-pr/entity-relationship).

> Если что-то непонятно или не получается – welcome в комменты к посту или в лс:)
>
> Канал: https://t.me/ViamSupervadetVadens
>
> Мой тг: https://t.me/ironicMotherfucker
>
> **Дорогу осилит идущий!**