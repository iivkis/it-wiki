# Go map: ключи-структуры, указатели, коллизии и NaN

> Небольшие коллизии случаются даже в самых приличных семьях  
> — Go runtime

## 1. Ключи-структуры и указатели  
В Go ключом карты может быть только **сравнимый** (`comparable`) тип — т.е. такой, к которому применимы `==` / `!=`. 
Структура считается сравнимой, если *каждое* поле сравнимо. Достаточно одного `[]byte` или `map`, чтобы структура «вылетела».

```go
type Point struct{ X, Y int } // оба поля сравнимы
var m map[Point]string        // ключ-значение
```

### 1.1. Значение vs указатель

```go
p := Point{1, 2}
m := make(map[any]int)
m[p]  = 1   // динамический тип ключа Point
m[&p] = 2   // динамический тип ключа *Point
fmt.Println(len(m)) // → 2
```

* Внутри пустого интерфейса `any` рантайм хранит **динамический тип + данные**. Раз типы разные — ключи разные.
* В типизированной карте (`map[Point]…` vs `map[*Point]…`) компилятор не даст «перепутать» два подхода.

**Изменяем поля?**

* `map[Point]…` безопасна: в карту уже скопирована *копия* ключа.
* `map[*Point]…` остаётся работоспособной (хэш строится по адресу, а он неизменен), но логическая идентичность может «поплыть» — поэтому лучше не менять данные под ключом. ([Medium][3])

**Лайфхак:** нужна Java-style логическая идентичность? Сгенерируйте компактный ключ — строку, `uint64`, крипто-хэш.

## 2. Одно правило хэш-таблицы

`a == b` ⇒ `hash(a) == hash(b)`. Обратное необязательно: коллизии — штатная ситуация.

### 2.1. Два `NaN` — одинаковый хэш, разные слоты

```go
a := math.NaN(); b := math.NaN()
fmt.Println(a == b)       // false
m := map[float64]int{a: 1}
m[b] = 2
fmt.Println(len(m))       // 2
```

IEEE-754 предписывает `NaN != NaN`; рантайм видит равный хэш, но второе сравнение `==` возвращает *false* и кладёт запись рядом. ([go.dev][6])

### 2.2. Демонстрация коллизии FNV-1a

```go
s1, s2 := "oCrpnX", "DKABkA"
fmt.Println(fnva32(s1), fnva32(s2)) // 1027557229, 1027557229
```

FNV-1a быстра, но 32-битное пространство тесновато — коллизии неизбежны. В рантайме Go сейчас используется seeded `memhash`, но пример удобен для иллюстрации. ([GitHub][4], [GitHub][9])

**Важно помнить**

* Коллизия ≠ ошибка; после совпадения хэша рантайм ищет ключ линейно в бакете. ([dave.cheney.net][10])
* С Go 1.3 добавлен случайный seed, чтобы усложнить DOS-атаки на коллизиях. ([GitHub][5])

## 3. Что впереди: Swiss Tables

Go 1.24 приносит полностью новый движок карт на Swiss Tables:

* +20-30 % к скорости выборки и вставки;
* −15 % к памяти под большие карты;
* обратная совместимость 100 %. ([go.dev][7], [go.dev][8])

## Итоги практики

| Требование                                      | Рекомендуйте                                          |
| ----------------------------------------------- | ----------------------------------------------------- |
| **Логическая** идентичность                     | Ключ-значение (struct)                                |
| **Идентичность по адресу** или несравнимые поля | Ключ-указатель                                        |
| Производительность при больших структурах       | Указатель + осторожный lifecycle                      |
| Безопасность, DoS-устойчивость                  | Пользуйтесь встроенной картой — seeded-хэш уже внутри |

Коллизии? Не паникуйте. Просто доверьтесь runtime — он давно умеет с ними жить.

---

### Использованные источники  
1. Go Specification — Map types :contentReference[oaicite:14]{index=14}  
2. «All your comparable types» (go.dev blog) :contentReference[oaicite:15]{index=15}  
3. Dave Cheney — «How the Go runtime implements maps efficiently» :contentReference[oaicite:16]{index=16}  
4. GitHub issue #2630 «hash function should be randomized» :contentReference[oaicite:17]{index=17}  
5. Medium — «Golang’s Map Key Restrictions…» :contentReference[oaicite:18]{index=18}  
6. runtime/map_noswiss.go (GitHub) :contentReference[oaicite:19]{index=19}  
7. sindresorhus/fnv1a (GitHub) :contentReference[oaicite:20]{index=20}  
8. Go issue #59531 — NaN comparisons :contentReference[oaicite:21]{index=21}  
9. Swiss Tables blog post (go.dev) :contentReference[oaicite:22]{index=22}  
10. Go 1.24 Release Notes – Runtime section :contentReference[oaicite:23]{index=23}
::contentReference[oaicite:24]{index=24}


[1]: https://go.dev/ref/spec "The Go Programming Language Specification"
[2]: https://go.dev/blog/comparable "All your comparable types - The Go Programming Language"
[3]: https://medium.com/%40rahul.b.shinde1/golangs-map-key-restrictions-and-understanding-channels-interfaces-and-pointers-as-key-types-498dc6e8a006 "Golang's Map Key Restrictions; Understanding Channels, Interfaces ..."
[4]: https://github.com/golang/go/blob/master/src/runtime/map_noswiss.go "go/src/runtime/map_noswiss.go at master · golang/go - GitHub"
[5]: https://github.com/golang/go/issues/2630 "runtime: hash function should be randomized to prevent DOS ..."
[6]: https://go.dev/issue/59531 "all: ordering numeric values, including NaNs, and the consequences ..."
[7]: https://go.dev/blog/swisstable "Faster Go maps with Swiss Tables - The Go Programming Language"
[8]: https://go.dev/doc/go1.24? "Go 1.24 Release Notes - The Go Programming Language"
[9]: https://github.com/sindresorhus/fnv1a "sindresorhus/fnv1a: FNV-1a non-cryptographic hash function - GitHub"
[10]: https://dave.cheney.net/2018/05/29/how-the-go-runtime-implements-maps-efficiently-without-generics "How the Go runtime implements maps efficiently (without generics)"
