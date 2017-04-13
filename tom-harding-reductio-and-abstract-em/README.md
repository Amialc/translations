# Доказательство и выводы

*Перевод статьи [Tom Harding](http://www.tomharding.me): [Reductio and Abstract 'em](http://www.tomharding.me/2017/02/24/reductio-and-abstract-em/). Опубликовано с разрешения автора.*

О, привет, незнакомец! Давно не общались. Если тебе интересно, я сменил дом, работу и компанию с момента написания моей последней статьи, отсюда и перерыв. **Прости!** В любом случае, говоря о страшных переходах, вы когда-нибудь замечали, что *вы можете написать каждую функцию списка с [reduceRight](https://developer.mozilla.org/ru/docs/Web/JavaScript/Reference/Global_Objects/Array/reduceRight)?*

## *… Э, нет, вы не можете?*

Хорошо, будьте ко мне снисходительны: есть **два предостережения.** Сейчас, давайте предположим, у нас просто есть две функции:

```js
const head = ([x, ... xs]) =>  x
const cons = ( x,     xs ) => [x, ... xs]
```

Тут же мы можем увидеть, что `cons` странное имя для `prepend`. Я объясню *почему* это так чуть позже, пока что опустим это и пойдем дальше. До тех пор я обещаю, что это не отмазка!

## *… Так, хорошо. Что ты там говорил?*

Давайте начнём со всеми любимой функции списка: `map`. В этой функции хорошо, что её накопитель **другой список** - мы преобразуем один список в другой!

```js
const map = (f, xs) => xs.reduceRight(
  (acc, x) => cons(f(x), acc], [])
)

// [2, 3, 4]
map(x => x + 1)([1, 2, 3])
```

Здорово, да? С этой реализацией довольно просто получить вторую всеми любимую функцию списка - `filter`:

```js
const filter = (p, xs) => xs.reduceRight(
  (acc, x) => p(x) ? cons(x, acc)
                   :         acc,
  []
)

// [1, 2]
filter(x => x < 3)([1, 2, 3, 4])
```

**Бам!** Если условие будет выполнено, мы **сохраним** элемент. В противном случае, мы пробросим накопитель нетронутым. Что насчёт *третьей* всеми любимой функции списка - `reduce`? …Ну, это немного сложно, поэтому давайте разбираться с ней.

## *Хорошо… а что насчёт ____?*

Назовите её и мы напишем её! Может начнём с `append`?

```js
const append = (x, xs) => xs.reduceRight(
  (acc, h) => cons(h, acc), [x]
)

// [1, 2, 3, 4]
append(4)([1, 2, 3])
```

Эта операция `reduceRight` на самом деле *ничего* не делает, но начинается с непустым накопителем, в который уже добавлено значение! Таким же способом, мы можем написать `concat`:

```js
const concat = (xs, ys) =>
  xs.reduceRight(
    (acc, h) => cons(h, acc), ys
  )

// [1, 2, 3, 4]
concat([1, 2])([3, 4])
```

В любом случае, теперь у нас есть `append`, мы можем написать `reverse`:

```js
const reverse = xs => xs.reduceRight(
  (acc, x) => append(x, acc), []
)

// [3, 2, 1]
reverse([1, 2, 3])
```

Она просто берет каждый элемент с конца списка и переносит его в конец накопителя. Легко! Продолжаем, `length` еще проще:

```js
const length = xs => xs.reduceRight(
  (n, _) => n + 1, 0
)

// 4
length([1, 2, 3, 4])
```

Это всё весело, но это не *взрывает мозг*; есть вероятность, что вы уже видели получение длинны через свёртку в какой-то момент. Почему бы нам не попробовать что-то посложнее? Давайте напишем `elemAt` - функцию, которая возвращает элемент по заданному индексу. Например, `elemAt(2, xs)` тоже самое, что `xs[2]`. Ах, да, верно: **доступ к массиву — это свёртка.**

```js
const elemAt = (n, xs) => head(xs.reduce(
  ([e, n], x) => [n == 0 ? x : e, n - 1],
  [undefined, n]
))

// 3
elemAt(2, [1, 2, 3])
```

Так, вот ещё хитрость: мы считаем индекс до того пока не получим `0`, потом “сохраняем” значение из этой позиции. Но **подождите!** Мы использовали `reduce`, не `reduceRight`!

Хорошо, да, вы можете написать эту функцию с помощью `reduceRight`, и я оставлю это как ([довольно сложное](http://stackoverflow.com/questions/14526254/find-the-kth-element-of-a-list-using-foldr-and-function-application-explana)) упражнение для читателя. Однако *гораздо* проще разобраться с `reduce`. Кроме того, если мы сможем доказать, что `reduce` может быть записан с помощью `reduceRight`, это не уловка, не так ли?

```js
const reduce = (f, acc, xs) =>
  xs.reduceRight(
    (accF, x) => z => accF(f(z, x)),
    x => x
  )(acc)
```

Поделом тебе за вопрос! Идея заключается в том, что **мы сворачиваем список в функцию** для вычисления `reduce`. Мы начали с `x => x`, которая ничего не делает, и потом применили новую функцию для каждого элемента в списке. Давайте пройдемся по простейшему примеру:

```js
reduce((x, y) => x - y, 10, [1, 2])

  // Разворачиваем `reduce` в `reduceRight`
  == [1, 2].reduceRight(
       (g, x) => z => g(
         ((x, y) => x - y)(z, x)
       ),

       x => x
     )(10)

  // Упрощаем редьюсер
  == [1, 2].reduceRight(
       (g, x) => z => g(z - x),
       x => x
     )(10)

  // Поглащаем первый элемент
  == [1].reduceRight(
       (g, x) => z => g(z - x),
       z => (x => x)((x => x - 2)(z))
     )(10)

  // Упрощаем некрасивый накопитель
  == [1].reduceRight(
       (g, x) => z => g(z - x),
       x => x - 2
     )(10)

  // Поглащаем следующий элемент
  == [].reduceRight(
       (g, x) => z => g(z - x),
       z => (x => x - 2)((x => x - 1)(z))
     )(10)

  // Упрощаем некрасивый накопитель
  == [].reduceRight(
       (g, x) => z => g(z - x),
       z => z - 3
     )(10)

  // `reduceRight` на [] == acc
  == (z => z - 3)(10)

  // Вычисляем
  == 7
```
Мы выжили! Может потребоваться пара прочтений, но основная мысль — это то, что наш накопитель создал функцию, выполняющую каждое действие задом наперёд. Конечно же, `reduce` и `reduceRight` вычислят одинаковые значения для `(x, y) => x - y`, поэтому попробуйте что-нибудь такое `(x, y) => [x, y]`, чтобы почувствовать разницу.

Ты убедился? Мы можем рассмотреть еще примеры, если ты... нет? Ну что ж, хорошо. Давай вернёмся к тому, *почему* каждая функция списка — это вид `reduceRight`.

## ([Поразительно знакомый](http://www.tomharding.me/2016/10/29/peano-forte/)) список

Список может быть выражен как `[]` (**пустой**) или `[x, ... xs]`, **непустой** список - элемент, *за которым еще один список\*.* Это точно [связанный список](https://ru.wikipedia.org/wiki/Связный_список)!

На данный момент мы можем объяснить почему просто получили `cons` и `head` раньше: всё, что они делают, это **создают** и **разрушают** списки. Они - просто способ описания *структуры* нашего списка.

## Представляем нашего героя

Запишем два выражения, которые определяют как `reduceRight` работает:

```js
[].reduceRight(f, acc) = acc

[x, ... xs].reduceRight(f, acc) =
  f(xs.reduceRight(f, acc), x)
```

Вот и весь `reduceRight`. Пустой список сворачивается до накопителя, непустой список сворачивается до `f` хвостовой свёртки и головы… Код яснее, чем предложение.

Теперь, когда **reduceRight позволяет нам устанавливать пустое и непустое представление,** и **имеет накопитель,** мы можем свободно изменять структуру листа *полностью*. Помните, что мы не можем написать `length` с помощью `map`, потому что `map` не позволяет нам изменять структуру (длину!) листа. Также, мы не можем написать `length` с помощью `filter`, потому что `filter` не имеет накопителя!

Чем `reduceRight` является на самом деле, формально, это **катаморфизм**: способ сворачивания типов (в нашем случае, **списка**) в значение. Теория тут простая: если у тебя есть доступ ко всем возможным конфигурациям твоей структуры, ты можешь делать, что захочешь. Если — нет, то не можешь!

## `reduce` против `reduceRight`?

Учитывая то, что вы можете получить `reduceRight` с помощью `reduce`, это может показаться странным брать за основу менее популярный. Ответ кроется в [ленивых языках и бесконечности](https://wiki.haskell.org/Foldl_as_foldr), и есть уже много [объяснений ленивого `reduceRight`](https://www.quora.com/Haskell-programming-language-Isnt-foldr-just-foldl-applied-on-a-reversed-list) онлайн - вам не нужна *моя* плохая попытка!

## *И так… `reduceRight` может делать, что угодно со списками?*

Да! Для дальнейшего чтения, катаморфизм ещё называется **свёрткой**, не предполагающую *разворачивания* (**анаморфизм** - больше чудесных названий!), и [функция Ramda unfold](http://ramdajs.com/docs/#unfold) может объяснить, что это значит. Подумайте о функции, создающую *диапазон*, который разворачивает изначальное число в список от 0 до числа! Всё же, мы можем думать о ней ни как о *функции списка*, потому что она не функция списка, а просто функция, возвращающая список\*\*.

**tl;dr?** Когда The Beatles сказали, что всё, что нам нужно, — *любовь*, они, наверное, хотели сказать `reduceRight`.

- - - -

На этом всё! Я надеюсь писать на регулярной основе, теперь я заселился. Увидимся в следующий раз!

Берегите себя ♥

*\* Просто как [числа Пеано](http://www.tomharding.me/2016/10/29/peano-forte/), где либо ноль (`Z`), либо на один больше, чем другие числа Пеано (`S Peano`).*

*\*\* Если ты морщинистый математик, прости, это пособие для новичков!*

- - - -

*Читайте нас на [Medium](https://medium.com/devschacht), контрибьютьте на [Github](https://github.com/devSchacht), общайтесь в [группе Telegram](https://t.me/devSchacht), следите в [Twitter](https://twitter.com/DevSchacht) и [канале Telegram](https://t.me/devSchachtChannel), рекомендуйте в [VK](https://vk.com/devschacht) и [Facebook](https://www.facebook.com/devSchacht). Скоро подъедет подкаст, не теряйтесь.*

[Статья на Medium](https://medium.com/devschacht/tom-harding-reductio-and-abstract-em-b42b956efe56)