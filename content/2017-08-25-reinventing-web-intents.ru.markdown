---
slug: reinventing-web-intents
date: 2017-08-25T13:20:31+01:00
title: "Reinventing Web Intents"
description: ""
tags: ["intents"]
image_header: /images/bridges.png
---
Я никогда не перестал [смерти веб-намерений](/what-happened-to-web-intents/). Я всегда чувствовал, что в Интернете все еще есть серьезная проблема, мы строим [силосы](/unintended-silos/), которые блокируют пользователя на одном веб-сайте, и мы не соединяем наши приложения вместе, чтобы строить более богатые впечатления. У нас есть ссылки, которые позволяют нам перейти на другой сайт, но мы не подключаем наши приложения к функциональным возможностям, которые мы можем использовать на наших сайтах. Собирайте изображение из облачной службы для использования в своем приложении или редактируйте изображение в предпочтительном редакторе пользователей; мы просто не связываем наши сервисы так, как мы связываем наши страницы.

[Web Intents](https://en.wikipedia.org/wiki/Web_Intents) была неудачной попыткой исправить это. API [Share API](/navigator.share/) разрешает один вариант использования для соединения сайтов и приложений, но, как правило, IPC и обнаружение служб никогда не были решены, и я думаю, что у меня есть решение ... Хорошо, у меня нет решения, у меня есть эксперимент, который я невероятно волнуюсь.

За последние пару месяцев Сурма в моей команде и Ян Килпатрик работали над прокладкой для [Tasklets API](https://github.com/GoogleChromeLabs/tasklets). API Tasklets был разработан таким образом, чтобы в Интернете существовал легкий многопоточный API. Класс ES6 может быть представлен как «tasklet», и вы можете назвать его без блокировки основного потока - отлично подходит для пользовательских интерфейсов. API-интерфейс Tasklet очень интересен, но наиболее интересной для меня было то, что они создали Polyfill с помощью веб-рабочего и разработали способ раскрытия функциональности класса ES6, который был определен в Worker. Они отбросили все сложности API PostMessage до аккуратного пакета и разумной модели для разработчиков JS.

Одна из причин, по которой мы создали API Web Intents, заключалась в том, что опыт разработчиков в создании API и службы, работающей с API PostMessage, был невероятно сложным, вам нужно иметь дело с API PostMessage, а затем вам нужно управлять сложным сообщением системы обработки и связанного автомата.

<figure><img src="/images/worker-dx.png"><figcaption> Традиционный рабочий </figcaption></figure>

Это просто сложно. Это становится еще хуже, если вы хотите, чтобы два окна взаимодействовали друг с другом. Открываемое окно должно подать сигнал «открывать», что он готов, прежде чем вы сможете начать отправлять сообщения. TL; DR - `window.open` открывает` about: blank`, прежде чем он перейдет к указанному вами URL-адресу.

<figure><img src="/images/window-dx.png"><figcaption> Опыт работы в PostMessage </figcaption></figure>

Это становится еще более сложным, когда вы хотите передавать сообщения между несколькими окнами или рабочими в других окнах.

<figure><img src="/images/complex-workers.png"><figcaption> Еще сложнее ... </figcaption></figure>

Я думаю, что это одна из основных причин, по которым люди выставляют API-интерфейс на стороне клиента. Это слишком сложно.

У талисманов polyfill был встроенный в него раствор, и я настойчиво попросил Surma, чтобы он мог реорганизовать API-интерфейсы задач в простой прокси-API, через пару часов выскочил [Comlink](https://github.com/GoogleChromeLabs/comlink/). Comlink - это небольшой API, который абстрагирует API MessageChannel и postMessage API для API, который выглядит так, будто вы создаете удаленные классы и функции в локальном контексте. Например:


**Веб-сайт**


```javascript
const worker = new Worker('worker.js');
const api = Comlink.proxy(worker);
const work = await new api.HardWork();
const results = await work.expensive();
```



** Веб-работник **


```javascript
class HardWork {
  expensive() {
    for(let i = 0; i < 1e12; i++)
      sum += /* …omg so much maths… */
    return sum;
  }
}

Comlink.expose({HardWork}, self);
```


Мы предоставляем API для службы, мы используем API в клиенте через прокси.

Я думаю, что это невероятно убедительно, и Comlink сам по себе способен революционизировать использование веб-работника, резко улучшая опыт разработчиков, предоставляя простой API для своей команды, чтобы иметь возможность использовать.

Сделать то же самое между окнами так же просто.

<figure><img src="/images/comlink.png"><figcaption> Comlink </figcaption></figure>

Но у меня была другая мысль ... Я могу изобрести небольшую часть Web Intents, которая должна была: улучшить обнаружение службы и упростить взаимодействие разработчиков с сервисами.

### Веб-намерения?

Одна из опрятных особенностей API Comlink заключается в том, что он автоматически попытается использовать объекты «Transferable» для передачи данных между клиентом и службой, и оказывается, что «MessagePorts» могут быть переданы. Идея, которая у меня возникла, состоит в том, что если бы я мог создать простой API, который предназначен для того, чтобы просто вернуть MessagePort на основе некоторых критериев (например, глагола), то в качестве клиента мне было бы безразлично, откуда пришел этот MessagePort.

Вот мое мышление: у меня будет сайт, который будет действовать как средний человек и будет поддерживать список услуг и где они будут жить, и сможет подключить клиентов, которые просят типы услуг, вроде как так.


* Сайт службы сможет сказать среднему человеку «Я предлагаю сервис X, который работает с данными Y и живет на странице Z»
* Клиентский сайт сможет сказать среднему человеку: «Мне нужна служба, которая делает X по этим данным Y. Что у вас есть?»

Сопоставляя это с грубым дизайном, мне нужна Служба, которая предоставляет два метода: `register` и` pick`.

`register`, будет, хорошо зарегистрировать услугу со средним человеком. «pick», с другой стороны, немного интереснее, и я сломал его на пару шагов.

<figure><img src="/images/webintents-step-1.png"><figcaption> Подключение сайтов </figcaption></figure>

Поток не слишком сложный, когда вы погружаетесь в него. Я упаковал [базовую оболочку, которую вы включаете в каждую службу и клиентское приложение](https://web-intents.glitch.me/scripts/service.js). Обертка обрабатывает первое взаимодействие с посредником и выполняет некоторую базовую ведение домашнего хозяйства путем упаковки сложностей открытия окна в сборщик услуг на странице «https://web-intents.glitch.me/pick».

Как только сборщик открыт, он найдет все службы, соответствующие критериям, которые нужны пользователю, и затем представит их пользователю в виде простого списка. Пользователь открывает свой предпочтительный сайт и за кулисами, который сайт предоставляет API для исходного клиента через посредника. Наконец, когда соединение завершено, и мы говорим с выбранным сервисом, мы можем удалить среднего человека.

<figure><img src="/images/webintents-step-2.png"><figcaption> Удаление посредника </figcaption></figure>

Процесс на самом деле немного сложнее, чем я допускаю. Под капотом мы пропускаем много MessagePorts между окнами, но потребители API никогда не видят такой сложности. Хорошо, что когда клиент и служба подключены, и они говорят напрямую через хороший API, определенный сервисом, и они фактически не знают, кто с обеих сторон. Ухоженная.

Ниже приведено быстрое погружение в код, чтобы показать, насколько это просто.


** Сервис ** ([demo](https://web-intents-service-1.glitch.me/))

Сервис относительно прост, он имеет класс, который взаимодействует с DOM и регистрирует некоторый вывод.

Мы выставляем класс `Test` в` ServiceRegistry`, и мы предлагаем способ зарегистрировать возможности этой службы.


```javascript
class Test {
  constructor() {}

  outputToPre(msg) {
    let output = document.getElementById('output');
    output.innerText += msg + '\n';
  }
}

let registry = new ServiceRegistry({ Test })
register.onclick = async () => {    
  let resolvedService = await registry.register('test-action','*', location.href);  
};
```



** Клиент ** ([demo](https://web-intents-client.glitch.me/))

Клиент прост, мы создаем экземпляр реестра и вызываем `pick`.

`pick` подключается к посреднику и ждет, когда пользователь выберет услугу. Когда пользователь выбирает услугу, посредник (`ServiceRegistry`) передает API, который удаленная служба открыла для клиента. Затем мы можем создать экземпляр удаленного API и вызвать на нем методы.


```javascript
let registry = new ServiceRegistry();
let resolvedService = await registry.pick('test-action','image/*');
remote = await new resolvedService.Test();
remote.outputToPre('calling from window.');
```


Я очень доволен этим как эксперимент. Вот видео из Обнаружения Сервиса и вызова вышеуказанного кода.

<figure> {{&lt;youtube 1igal-ehMB4&gt;}} <figcaption> сквозная демонстрация </figcaption></figure>

Дайте мне знать, что вы думаете. Слишком много?