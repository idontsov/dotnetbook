## Шаблон Lifetime

После того как мы так подробно рассмотрели шаблон освобождения ресурсов под названием IDisposable, мы сформулировали ряд проблемных зон этого шаблона, которые усложняют его использование. Чтобы решить эти проблемы, мы должны разработать некие противодействия, которые приведут нас к абсолютно новому и мощному инструменту. Итак, если представить что мы хотим изменить шаблон IDisposable или заменить чем-то другим, избавив при этом его от присущих ему недостатков, то что мы должны сделать? Давайте возьмем группу минусов из предыдущей главы и попробуем где это вообще возможно переделать поведение так, чтобы получить плюсы:

  1. Чтобы избежать не явности объявления разрушаемости объекта, мы должны ввести новый способ уведомления о том, что класс разрушаем. Причем необходимо сделать это так, чтобы пользователь экземпляра не смог бы обойти этот механизм стороной. А какой код экземпляра типа отрабатывает в любом случае? Только конструктор. Получается, конструктор должен обладать такими свойствами, которые подскажут вызвавшему его коду что экземпляр разрушаем. И механизмов тут может быть целых два: либо имя класса должно содержать ключевое слово (например, по аналогии с `Async`, слово `Disposable`) либо должен быть некий параметр, который необходимо передать конструктору;
  2. Проблема необходимости тянуть за собой некий интерфейс чтобы разрушить сущность может быть решена выносом контроля за временем жизни сущности из самой сущности. По сути, реализацией inversion of control процесса разрушения объекта в некий сторонний механизм;
  3. Третья проблема - существование объекта после его разрушения. Её, к сожалению, решить просто не возможно: недетерминированное удаление памяти из-под объекта делает эту задачу нерешаемой;
  4. Вопрос с `explicit` реализацией `IDisposable` уходит сам собой если IDisposable не реализовывать;
  5. Проблематика разделения выделения ресурсов и их освобождения по разным методам может быть решена регистрацией ресурсов в некотором контейнере, освобождение которого будет происходить автоматически;
  6. Если говорить о сложности разрушения графа объектов, разделенных на некие слои, которые разрушаются сами по себе (например, может быть необходимо чтобы был разрушен какой-то один слой графа, а остальные должны корректно существовать дальше), отписываясь от изменений в соседних слоях, то тут можно явно выделить одну общую особенность: о чём бы ни шла речь, логически мы говорим о неких группах объектов, время жизни которых зависит друг от друга. Т.е. введя понятие зависимости времени жизни одной сущности от времени жизни другой сущности мы решим проблему очистки конкретной группы объектов не затрагивая другую группу;

Объединяя все выше сказанное, можно сделать базовый вывод что IDisposable не дает нам полной гибкости в управлении разрушением состояния объектов. Он безусловно полезен в рамках некоторой группы сценариев, но не является универсальным инструментом, который способен решить все задачи.

Давайте же найдем тот механизм, который призван помочь нам в разрушении сложных структур объектов. Итак, основываясь на ранее перечисленных рассуждениях о механизмах улучшения шаблона IDisposable, составим список тезисов:

  - Наш класс либо должен как-то особо называться либо что-либо принимать в качестве параметра конструктора;
  - Процесс разрушения должен быть отдан на сторону;
  - Процесс разрушения должен быть автоматизирован;
  - Также должна существовать некая зависимость времени жизни одних объектов от времени жизни других других;
  - Мы при этом не должны каждый раз писать всю инфраструктуру, как мы делаем это для случая с IDisposable;

Поэтому:

  - Процессом разрушения должна заниматься отдельная сущность. Назовём её `Lifetime`;
  - Конструктор класса должен принимать на вход экземпляр типа `Lifetime` если жизнь экземпляра нашего класса зависит от времени жизни внешнего `Lifetime` либо создавать свой собственный если зависимостей не существует;

Теперь, в новых реалиях, внешний код будет передавать нам специальный экземпляр `Lifetime`, от которого мы будем зависеть и относительно времени жизни которого мы будем существовать. А чтобы нас разрушить, внешний тип не будет разрушать нас напрямую: вместо этого он закончит время жизни экземпляра `Lifetime`, который самостоятельно завершить время жизни всех, кто на него подписан.

С таким подходом мы более не отдаем каждому, кто имеет ссылку на наш объект возможность разрушить нас. Помните что мы имели в шаблоне IDisposable? При проектировании класса мы должны были реализовать интерфейс IDisposable: сделать метод `Dispose()` публичным для всех. Это значит что любой, кто получал ссылку на экземпляр нашего типа получал также средство уничтожения экземпляра нашего типа. Теперь ситуация совершенно иная: конструктор экземпляра типа получает в качестве параметра отрезок собственной жизни как внешнюю зависимость. Не выставляя при этом никаких методов наружу. Получается, что в данном сценарии временем жизни нашего экземпляра типа владеет только тот, кто его создал и никто иной. С шаблоном IDisposable единственный путь обойти публикацию метода - реализовать этот интерфейс неявно, скрыв таким образом метод `Dispose`. Однако, неявная реализация интерфейса будет вызывать много вопросов, т.к. современные IDE позволяют легко и просто увидеть наличие этого метода в структуре типа.

Теперь осталось закрыть еще один вопрос: разделение зон ответственности. Ведь если у нас появляется задача передать кому-либо свой `Lifetime`, не хотелось бы отдавать контроль над возможностью вызова метода запуска разрушения, который закончит жизнь всех, кто подписан на переданный экземпляр `Lifetime`.

Также, чтобы использующие `Lifetime` сущности не могли прервать время его жизни самостоятельно, необходимо ввести разделение зон ответственности между участниками процесса:

  1. `LifetimeDef`. Экземпляр данного типа хранится у того объекта, который будет владеть завершением времени жизни зависящих от него объектов;
  2. `Lifetime` создается экземпляром `LifetimeDef` и будет передан всем тем, кто будет так или иначе зависеть от владельца. Сами они не могут вызвать метод `Terminate`: он доступен только у `LifetimeDef`. Забота экземпляров типов, которые получили `Lifetime` - просто накидать внутрь него действий, которые будут их разрушать.
  3. `OuterLifetime` - инкапсулирует понятие readonly `Lifetime`. Другими словами, это средство защиты от конечного программиста: чтобы он не мог в переданную зависимость добавить действия по уничтожению своей. Этот тип используется, когда вы отдаете кому-либо экземпляр `Lifetime` так, чтобы относительно него можно было бы только создать новый, зависимый `LifetimeDef`. Но добавить туда свои собственные было бы не возможно. Публично, как и `Lifetime`, `OuterLifetime` содержит только свойство `IsTerminated`, однако на основе него можно создать зависимый экземпляр `LifetimeDef`, на основе которого можно осуществлять менеджмент жизни собственного экземпляра.

Рассмотрим самый базовый вариант интерфейса типа:

```csharp
public class Lifetime
{
    public static Lifetime Eternal = Define("Eternal").Lifetime;

    public bool IsTerminated { get; internal set; }

    public void Add(Action action);
}
```

Что мы здесь видим? Есть новая сущность, `Lifetime`. Сущность эта имеет свойство `IsTerminated`, которое призвано помочь в понимании, закончилось ли время жизни Lifetime и всех, кто от него зависит или еще нет. Задать список действий, которые должны произойти в случае смерти экземпляра типа `Lifetime` можно путем их добавления во внутренний список методом `Add`.

Также давайте рассмотрим второй краеугольный камень этого шаблона: класс `LifetimeDef`, который владеет правом на останов срока жизни экземпляра `Lifetime`, которым владеет.

```csharp
public class LifetimeDef : IDisposable
{
    public Lifetime Lifetime { get; }
    public string Name { get; }

    private const string Noname = "Unnamed";

    public LifetimeDef(string name = null)
    {
        Name = name ?? Noname;
        Lifetime = new Lifetime();
    }

    public void Terminate()
    {
        Lifetime.Terminate();
    }

    public void Dispose()
    {
        Terminate();
    }
}
```

Для удобности он содержит необязательное поле `Name`. Это введено чтобы при отладке множественные экземпляры типа `LifetimeDef` не вводили бы в заблужение и позволяли бы легко находить необходимые сущности. Метод `Terminate` обрывает жизнь экземпляру класса Lifetime, а реализация шаблона `IDisposable` позволяет использовать `LifetimeDef` в классических сценариях. Например, в блоке `using`:

```csharp
[Test]
void EntityTest()
{
    Entity entity;
    using(var ldf = Lifetime.Define())
    {
        entity = new Entity(lfd.Lifetime);
        entity.OpenConnection();
        //...
    }

    Assert.Equal(State.Closed, entity.ConnectionState)
}
```