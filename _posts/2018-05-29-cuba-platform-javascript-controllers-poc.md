---
layout: article
title: CUBA.platform приложение на JS. PoC
tags: CUBA.platform js javascript nashorn 
mathjax: false
---

## Мотивация
Как и многие другие, данная история началась в баре. Разговор зашел о возможностях CUBA.platform и людях которые ее используют. Содержание разговора было бы нам совершенно не интересно, если бы в него не вступил Frontend разработчик. Ему было интересно может ли он разрабатывать CUBA.platform приложения зная исключительно JavaScript. Я ответил: "Нет",- но тут же вспомнил, что я уже неоднократно слышал о том, что есть положительный опыт запуска JavaScript под JVM. Далее, дискуссия пошла в каком-то ином направлении, но вопрос я запомнил. 

И вот, в один прекрасный день, я вспомнил вопрос и решил, что нужно попробовать, что из этого получилось вы можете прочитать далее в текущей и будущих статьях.

## Описание задачи
Разработать CUBA.platform компонент предоставляющий возможность программирования CUBA.platform Application на языке JavaScript.

<!--more-->

## Постановка
В рамках PoC имеет смысл реализовать некоторый простой для CUBA.platform кейс, позволяющий оценить жизнеспособность и сложность реализации идеи. Постановка вида: "Вывести на экран пользователя сообщение `Hello World!`",- кажется неплохой, но требует уточнений, определяющих, в каком виде и в какой момент должно показываться сообщение. Допустим, что показывать сообщение можно в качестве стандартной нотификации, когда пользователь нажал на экране кнопку. Назовем кнопку: "clickMe".

## Экран с кнопкой
Для начала, создадим стандартный CUBA.platform экран с кнопкой (имя экрана: `js-screen`). Для этого можно воспользоваться функционалом CUBA Studio.

В результате у нас будет два файла. XML описание экрана (`js-screen.xml`):

```xml
<window xmlns="http://schemas.haulmont.com/cuba/window.xsd"
        caption="msg://caption"
        class="com.company.jscontroller.web.screens.JsScreen"
        messagesPack="com.company.jscontroller.web.screens">
    <dialogMode height="600"
                width="800"/>
    <layout>
        <button id="clickMe"
                caption="msg://clickMe"/>

    </layout>
</window>
```

И Java контроллер (`JsScreen.java`):

```java
public class JsScreen extends AbstractWindow {}
```

## Nashorn
Для того, чтобы получить возможность исполнения JS внутри JVM нам необходим соответствующий Runtime. И в составе JVM уже имеется такой, имя ему Nashorn[1].
После непродолжительного чтения документации [2][3], мы находим все необходимое для исполнения JS кода из Java:

```java
ScriptEngine engine = new ScriptEngineManager().getEngineByName("nashorn"); (1)
engine.eval("print('Hello World!');"); (2)
```

* (1) создаем `ScriptEngine`, он необходим для исполнения JS кода
* (2) выполняем JS код.

Стоит обратить внимание на функцию `print`. Знакомые с JS, скажут, что такой функции нет. Но они будут не до конца правы. Действительно, в браузерах данной функции нет, но Nashorn и не является браузером. `print` - это как раз пример такого различия. Далее мы увидим, что Nashorn предоставляет и другие, специфичные только для него, функции.

## JS Controller
После того, как мы выяснили, что необходимо сделать для того чтобы исполнить JS код в Java, напишем код JS контроллера:

```js
// getting ui component
var button = frame.getComponent("clickMe"); (1)

// getting action Java type
var BaseAction = Java.type("com.haulmont.cuba.gui.components.actions.BaseAction"); (2)

var handler = function (e) { (3)
    frame.showNotification("Hello World!");
};

var action = new BaseAction("clickMe").withHandler(handler); (4)
button.setAction(action);
```

Сохраним JS код выше в файл `js-screen.js` (рядом с `js-screen.xml`) и рассмотрим подробнее, что происходит на каждом из шагов:

* (1) получаем элемент экрана. В CUBA.platform контроллерах, элементы экрана наиболее часто получаются через инжектирование (использования `@Inject`)
* (2) получаем описание Java type `BaseAction`. В CUBA.platform элемент `Button` в качестве действия принимает экземпляр класса `Action`. Так как `Action` является интерфейсом, нам необходимо подобрать подходящую реализацию. `BaseAction` хорошо подходит на эту роль.

	Метод `Java.type("...")` является специфичным для `Nashorn` и позволяет получить описание Java типа из JS. В дальнейшем можно создать экземпляр такого типа воспользовавшись конструкцией вида `new BaseAction("clickMe")`.
* (3) описываем функцию-обработчик, которая будет вызываться пользователем по нажатию на экранную кнопку. `Nashorn` не поддерживает стрелочные функции
* (4) создаем экземпляр `BaseAction`, указываем обработчик

Последним, что требует разъяснения, это объект `frame`, который встречается в (1) и (3). Данный переменная отображает Java объект `Window` экрана в котором была нажата кнопка. Самым простым и очевидным способом передать данный объект в JS является использование контекстов.

## Nashorn. Context
Nashorn поддерживает два варианта контекстов Global и Engine. В нашем случае будет достаточно использовать Engine. В результате код будет выглядеть следующим образом:

```java
ScriptEngine nashornScriptEngine = ...;

Bindings bindings = nashornScriptEngine.getBindings(ScriptContext.ENGINE_SCOPE);
bindings.put("frame", getFrame());

nashornScriptEngine.eval("", bindings);
``` 

## Собираем все воедино
Так как на предыдущих шагах мы однозначно не определяли Java package, будем считать что все файлы которые мы создали, располагаются в пакете `com.company.jscontroller.web.screens`. В результате в модуле `web` мы имеем следующий набор файлов:

- `com/company/jscontroller/web/screens/js-screen.xml` - декларативное описание экрана
- `com/company/jscontroller/web/screens/JsScreen.java` - Java контроллер
- `com/company/jscontroller/web/screens/js-screen.js` - JS контроллер

JS контроллер и декларативное описание экрана рассматривать нет необходимости, а вот на содержимом Java контроллера стоит остановится по подробнее:

```java
public class JsScreen extends AbstractWindow {

    public static final String JS_CONTROLLER_PATH = "com/company/jscontroller/web/screens/js-screen.js";

    @Inject
    protected Resources resources;

    @Override
    public void init(Map<String, Object> params) {
        // reading js controller
        String jsController = resources.getResourceAsString(JS_CONTROLLER_PATH); (1)

        // getting Nashorn engine
        ScriptEngine engine = new ScriptEngineManager().getEngineByName("nashorn"); (2)

        // configuring the frame binding
        Bindings bindings = engine.getBindings(ScriptContext.ENGINE_SCOPE);
        bindings.put("frame", getFrame()); (3)

        // evaluating js
        try {
            engine.eval(jsController, bindings); (4)
        } catch (ScriptException e) {
            throw new RuntimeException(e);
        }
    }
}
```

Рассмотрим подробнее, что происходит на каждом из шагов:
(1) зачитываем исходный код JS контроллера. Так как JS контроллер расположен в `classpath` мы можем воспользоваться стандартным для CUBA.platform способом, зачитав файл с помощью бина `Resources`
(2) получаем Nashorn `ScriptEngine`
(3) настраиваем `Binding`
(4) исполняем код JS контроллера с преконфигурированным Binding

## Почему JS контроллер исполняется в init?
В связи с тем, что код JS контроллера не разделен на отдельные функции, то стоит рассматривать весь код как единую функцию. С точки же зрения поставленной задачи (PoC) `init` кажется хорошим кандидатом на эту роль.
В дальнейшем мы разобьем код JS контроллера на отдельные функции и будем определять их внутри JS файла только если это необходимо.

* [1] http://openjdk.java.net/projects/nashorn/
* [2] http://winterbe.com/posts/2014/04/05/java8-nashorn-tutorial/
* [3] https://www.n-k.de/riding-the-nashorn/
