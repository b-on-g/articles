# Всё еще в догоняющих: обзор Solid2

Солид2 порадовал нас новой системой реактивности. В [cтатье](https://www.solidjs.com/blog/solid-2-0-the-long-hold/) Ryan Carniato пишет:

> Today signals are everywhere.

Штож, давайте проверим что получилось

## Тесты

Тестировать будем оптимизацию вычислений в графе по (этому бенчмарку)[https://habr.com/ru/articles/707600/]

Наш граф выглядит так:

```
C ← A, B
D ← A, B
E ← C, A, D
F ← D, B
G ← C, E (только если C ложно), D, F
H ← G
I ← G
J ← F
```

Где A,B это вход. Формула для выявления лишних вычислений:
`G = C() + ( C() || E()%2 ) + D()[4].x + F()`. Когда `C()` истинно, `G` не читает `E`, значит `E` не нужно.

Код теста Solid 2:

```js
const [A, setA] = createSignal(0)
const [B, setB] = createSignal(0)

createRoot(() => {
	const C = createMemo(() => (A() % 2) + (B() % 2))
	const D = createMemo(
		() => numbers.map(i => ({ x: i + (A() % 2) - (B() % 2) })),
		{ equals }, // $mol_compare_deep
	)
	const E = createMemo(() => hard(C() + A() + D()[0].x, 'E'))
	const F = createMemo(() => hard(D()[2].x || B(), 'F'))
	const G = createMemo(() => C() + (C() || E() % 2) + D()[4].x + F())
	createEffect(G, g => {
		res.push(hard(g, 'H'))
	})
	createEffect(G, g => {
		res.push(g)
	})
	createEffect(F, f => {
		res.push(hard(f, 'J'))
	})
})

setB(1)
setA(1)
flush() // считается E и H
setA(2)
setB(2)
flush() // считается E и H
```

`equals` здесь это `$mol_compare_deep`. По умолчанию её в Solid нет, я добавил её сам. Без неё `D` каждый раз отдаёт новый массив, и лишний раз считается еще и `F`.

Легенда таблицы:
🥶 Cold Run - итерации в секунду
🥵 Hot Run - итерации в секунду

Результаты:

| Вариант          | Размер кода | 🥶 холодный | 🥵 горячий |
| ---------------- | ----------- | ----------- | ---------- |
| SolidJS 2        | 116         | 10 074      | 10 830     |
| SolidJS 1        | 114         | 11 121      | 11 454     |
| `$mol_wire_solo` | 130         | 13 402      | 13 373     |
| `$mol_wire_let`  | 96          | 13 955      | 13 897     |

Что мы видим? Обе версии Solid лишний раз пересчитывают `E` в первом шаге, хотя `G` его там не читает. `H` считается у всех

Отдельный прогон в Node 24, с `equals` и без неё. Буквы это вызовы `hard()` в шаге:

| Вариант                     | шаг 1   | шаг 2   | оп/с   |
| --------------------------- | ------- | ------- | ------ |
| `$mol_wire_let`             | H       | E, H    | 12 300 |
| SolidJS 1 с `equals`        | E, H    | E, H    | 10 600 |
| SolidJS 2 с `equals`        | E, H    | E, H    | 9 600  |
| SolidJS 1 по умолчанию      | E, F, H | E, F, H | 7 400  |
| SolidJS 2 по умолчанию      | E, F, H | E, F, H | 7 200  |

Сравним так же размеры бандлов gzip:

- `solid-js@2` 49,7 КБ (из них `@solidjs/signals` 43,2 КБ) \\ Помните что сигналы сейчас везде, да ?)
- `solid-js@1` 8,6 КБ,
- `mol_wire_lib` 8,9 КБ,

В общем впечатления не впечатляющие. К сожалению прежде чем делать свою систему реактивности не был проведен рисерч что бы перенять, или хотя бы попробовать перенять, практику лучших. Прежде чем хейтить в коментариях дочитайте до конца. Они там придумали сделать dsl. Да, прямо как view.tree в $mol, иронично. Только пока у них 2 разных прототипа

Далее сравним другие улучшения из второй версии

## Асинхронность

Теперь промис ждет всё что от него зависит сам!

```jsx
function Profile(props) {
	const user = createMemo(() => fetchUser(props.id))
	return (
		<Loading fallback={<Skeleton />}>
			<div class={{ stale: isPending(() => user()) }}>{user().name}</div>
		</Loading>
	)
}
```

В $mol так был устроено изначально, еще 10 лет назад. При этом свойство пишется синхронно, как будто данные уже пришли:

```ts
export class $my_profile extends $.$my_profile {
	@$mol_mem
	user() {
		return this.$.$mol_fetch.json(`/api/user/${this.id()}`) as { name: string }
	}

	name() {
		return this.user().name
	}
}
```

```tree
$my_profile $mol_view
	id \
	sub /
		<= name \
```

И никакого Loading писать не нужно. Блок сам покажет что запрашиваются какие то данные. + при смене экрана запрос абортиться.

## Server Functions

Глубоко не понимаю и недоумеваю от этой "фичи".
Поясню, в моей картине мира у нас есть бэк со свагером OpenAPI и на клиенте у нас генериться код для бэка как то.
В общем я порисерчил - идея в том чтобы сделать фулстак фреймворк. Для этого нужно разделение кода как то организовать. Нужна изоморфность.

В Солиде так:

```js
'use server'
import { live } from '@solidjs/web/server-functions'

export const stockPrice = live(async function* (symbol) {
	for await (const tick of subscribe(symbol)) yield tick.price
})
```

```jsx
const price = createMemo(() => stockPrice(props.symbol))
```

В $mol у нас есть тэги для файлов. т.е один и тот же модуль может быть как фронтовым. так и бэковым так и общим, что бы шарить типы к примеру.

Выглядит это так: в том же самом репо у нас есть

file: api.node.ts

```ts
export class $my_api extends $mol_rest_resource {
	GET(msg: $mol_rest_message) {
		msg.reply({ path: msg.uri().pathname })
	}

	@$mol_mem
	price() {
		return $my_api_price.make({})
	}
}
$my_api.serve()
```

И рядом  
file: api.web.ts или просто api.ts
С тем же самым кодом как выше

```ts ------ подумать всё ли ок с примером
export class $my_profile extends $.$my_profile {
	@$mol_mem
	user() {
		return this.$.$mol_fetch.json(`/api/user/${this.id()}`) as { name: string }
	}
}
```

Или еще круче. Описать таблицу на клиенте, работать с данными локально, а синхронизироваться они будут сами.

```ts
export class $my_stock extends $giper_baza_dict.with({
	Price: $giper_baza_atom_real,
}) {}
```

```ts
stock() {
	return this.$.$giper_baza_glob.Pawn( this.link(), $my_stock )
}

price() {
	return this.stock().Price()?.val() ?? 0
}
```

## Диагностика

В солиде добавили нативную поддержку для агентов, и коды ошибок для быстрого поиска а-ля:

```
[STRICT_READ_UNTRACKED] Reactive value read directly in an effect callback will not update.
  at src/Item.tsx:12:19
```

В моле чуть проще:

```
$bog_collider.Root(0).Case(3).sample()
```

Человекочитаемые уникальные fqn имена.

## DSL

ReactiveScript и TSRX по идее для агентов, но как бы то что проще агентам писать то и людям проще
Примеры:

В $mol `view.tree` компилируется в TS-класс с типами:

```tree
$my_counter $mol_view
	sub /
		<= Value $mol_view
			sub / <= value? 0
		<= Inc $mol_button_minor
			title \+
			click? <=> inc? null
```

Получается класс со свойствами `Value()`, `value()`, `Inc()`, `inc()`. Логика дописывается в `.view.ts` переопределением:

```ts
export class $my_counter extends $.$my_counter {
	inc() {
		this.value(this.value() + 1)
	}
}
```

_Любое вложенное свойство_ **любого компонента** переопределяется у наследника одной строкой, без пропсов и без форка. То есть в солиде пытаються сделать точно такую же втстроенную композицию по умолчанию как и в моле.

Представленный [коллайдер](https://b-on-g.github.io/collider/) так и сделан, `Source` из глубины `$hyoo_js_perf` заменён тремя строками:

```tree
$bog_collider $hyoo_js_perf
	Source $mol_link_source
		uri \https://github.com/b-on-g/collider
```

## Ох уж эти конфиги

Вот что я точно знаю дак это то что никто не любит с ними возиться. Вот поэтому в $mol их и нет вообще) Есть соглашения, как ранее я описывал изоморфный код.

Солид залетел в тренд метафреймворков

```sh
npm create solid@latest
npm i @tanstack/solid-router @tanstack/solid-query
```

Не вижу смысла разделять на мета и не мета.

## Api

Ранее я писал как раз об этом статью https://habr.com/ru/articles/1056766/ она о том что в реакте нужно сделать выбор из 12 вариантов. В солиде сократили кол-во выборов до 2, что просто отлично

```js
createEffect(
	() => count(),
	value => {
		console.log(value)
	},
)
```

```jsx
const states = createMemo( ()=> getStates( 'US' ) )
const [ state, setState ] = createSignal( '' )
const cities = createMemo( ()=> state() ? getCities( state() ) : [] )

<Loading fallback="Загрузка…">
	<select onChange={ e => setState( e.target.value ) }>
		<For each={ states() }>{ s => <option>{ s }</option> }</For>
	</select>
	<select disabled={ isPending( ()=> cities() ) }>
		<For each={ cities() }>{ c => <option>{ c }</option> }</For>
	</select>
</Loading>
```

Как и в мол, там тоже всего два варианта

```tree
$my_geo $mol_list
	rows /
		<= State $mol_select
			value? <=> state? \
			options <= states /string
		<= City $mol_select
			value? <=> city? \
			options <= cities /string
```

```ts
export class $my_geo extends $.$my_geo {
	states() {
		return this.$.$mol_fetch.json('/api/states?country=US') as string[]
	}

	cities() {
		if (!this.state()) return []
		return this.$.$mol_fetch.json(`/api/cities?state=${this.state()}`) as string[]
	}

	@$mol_mem
	city(next?: string) {
		this.state()
		return next ?? ''
	}
}
```

Либо у вас метод с декоратором `@ mol_mem ` - для мемоизации. Либо `@ mol_action ` - для действия, например клика.

---

Есть много всего о чем я еще могу рассказать, о моле, конечно. Но пока на этом всё, подписывайтесь что бы следить. t.me/Dev_cmuser
