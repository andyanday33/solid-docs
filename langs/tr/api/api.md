# Temel Reaktivite

Solid'in reaktiviteye genel yaklaşımı her bir reaktif hesaplamayı bir fonksiyonun içine sarıp o fonksiyonu bağımlılıkları güncellendiğinde tekrar yürütmek üzerine.
Solid JSX derleyicisi ayrıca çoğu JSX ibaresini (süslü olmayan parantezler içerisindeki kod) bir fonksiyon ile sarmakta ki bağımlılıkları güncellendiğinde otomatik olarak güncellenebilsinler (ve gerekli olan DOM güncellemelerini etkinleştirebilsinler).
Daha açık konuşmak gerekirse, fonksiyonların otomatik yeniden yürütülmesi fonksiyon JSX ibareleri ya da API çağrıları gibi "hesaplama" oluşturan (`createEffect`, `createMemo`, vb.) _izleme kapsamı_ içerisinde her çağrıldığında gerçekleşir.
Varsayılan olarak, bir fonksiyonun bağlılıkları izleme kapsamı içerisinde çağrıldıklarında fonksiyonun ne zaman reaktif durumu (örneğin Sinyal alıcı ya da depolama özelliği) okuyacağını tespit ederek otomatik olarak takibe alınırlar.
Sonuç olarak, sizin genel olarak bağımlılıklar konusunda kafa yormanıza pek gerek yok. (Ama eğer olur da otomatik bağımlılık izleme istediğiniz sonucu vermezse, [bağımlılık takibini geçersiz kılabilirsiniz](#reaktif-yardımcılar).)
Bu yaklaşım reaktiviteyi _birleştirilebilir_ kılar: bir fonksiyonu başka bir fonksiyon içerisinde çağırmak genellikle çağıran fonksiyonun çağrılan fonksiyonun bağımlılıklarını miras almasıyla sonuçlanır.

## `createSignal`

```ts
import { createSignal } from "solid-js";

function createSignal<T>(
  initialValue: T,
  options?: { equals?: false | ((prev: T, next: T) => boolean) }
): [get: () => T, set: (v: T) => T];

// createSignal'in döndürdüğü değer için uygun olan türler:
import type { Signal, Accessor, Setter } from "solid-js";
type Signal<T> = [get: Accessor<T>, set: Setter<T>];
type Accessor<T> = () => T;
type Setter<T> = (v: T | ((prev?: T) => T)) => T;
```

Sinyaller en temel reaktif primitiflerdir. Zamanla değişen tek bir değeri (herhangi bir Javascript objesi olabilir) izlerler. Sinyal'in değeri verilen ilk argümana `initialValue` (ya da eğer hiçbir argüman yoksa `undefined`) eşit olarak başlar. `createSignal` fonksiyonu iki elementlik bir dizi olacak şekilde bir çift fonksiyon döndürür. Bir _alıcı_ (ya da _erişici_) ve bir _ayarlayıcı_. Tipik kullanımda, bu diziyi isimlendirilmiş bir Sinyal'e aşağıdaki gibi ayrıştırırsınız:

```js
const [count, setCount] = createSignal(0);
const [ready, setReady] = createSignal(false);
```

Alıcıyı çağırmak (örneğin `count()` ya da `ready()`) Sinyal'in anlık değerini döndürür.
Otomatik bağımlılık izleyiciye hayati olarak, alıcıyı bir izleme kapsamı içerisinde çağırmak, çağıran fonksiyonun bu Sinyal'e bağımlı olmasına sebep olacaktır, bu da Sinyal güncellenirse fonksiyonun yeniden yürütüleceği anlamına geliyor.

Ayarlayıcıyı çağırmak (örneğin `setCount(nextCount)` ya da `setReady(nextReady)`) Sinyal'in değerini ayarlar ve Sinyal'i _günceller_ (bağımlıların yeniden yürütülmesini tetikler),
eğer ki değer gerçekten de değişmişse (detayları aşağıda görebilirsiniz).
Tek argümanı olarak, ayarlayıcı ya sinyal için yeni bir değer, ya da sinyalin son değerini yeni bir değere eşlemleyen bir fonksiyon alır.
Ayarlayıcı ayrıca güncellenmiş değeri döndürür. Örneğin:

```js
// sinyal'in anlık değerini oku, ve
// eğer izleme kapsamı içerisindeysen sinyal'e bağlı ol.
// (ama izleme kapsamı dışarısında reaktif olma):
const currentCount = count();

// ya da herhangi bir hesaplamayı bir fonksiyonla sarmala
// ve bu fonksiyon izleme kapsamı içerisinde kullanılabilsin:
const doubledCount = () => 2 * count();

// ya da bir izleme kapsamı oluştur ve sinyale bağımlı ol:
const countDisplay = <div>{count()}</div>;

// bir değer belirterek sinyal'e yaz:
setReady(true);

// bir fonksiyon ayarlayıcı sağlayarak sinyal'e yaz:
const newCount = setCount((prev) => prev + 1);
```

> Eğer sinyal içerisine bir fonksiyon depolamak istiyorsanız, fonksiyon formunu kullanmanız gerekir:
>
> ```js
> setValue(() => myFunction);
> ```
>
> Fakat, fonksiyonlar `initalValue` argümanının `createSignal` içerisinde olduğu gibi özel muamele
> görmezler, yani bir fonksiyon başlangıç değerini aşağıdaki gibi atamalısınız:
>
> ```js
> const [func, setFunc] = createSignal(myFunction);
> ``;
> ```

##### Ayarlar

Solid içerisindeki pek çok primitif bir "ayarlar" objesini isteğe bağlı son argüman olarak alırlar.
`createSignal`'in ayarlar objesi bir `equals` ayarı sağlamanıza olanak sağlar. Örneğin:

```js
const [getValue, setValue] = createSignal(initialValue, { equals: false });
```

Varsayılan olarak, bir sinyalin ayarlayıcısını çağırdığınızda, sinyal sadece Javascript'in `===` operatörüne göre yeni değer eski değerden gerçekten farklıysa güncellenir (ve bağımlılarının yeniden yürütülmesine sebep olur).

Alternatif olarak, `equals` değerini `false`'a eşitleyerek bağımlıların ayarlayıcı her çağrıldığında yeniden yürütülmesini sağlayabilir, ya da eşitliği test etmek için kendi fonksiyonlarınızı verebilirsiniz.
Bazı örnekler:

```js
// { equals : false }'ı kullanarak objenin durduğu yerde modifiye edilebilmesini sağlayın;
// normalde, obje değişimden öncesi ve değişimden sonra aynı kimliğe
// sahip olduğu için bu bir güncelleme olarak görünmeyecektir
const [object, setObject] = createSignal({ count: 0 }, { equals: false });
setObject((current) => {
  current.count += 1;
  current.updated = new Date();
  return current;
});

// { equals: false } sinyal'ini değer olmadan tetikleyici olarak kullanın:
const [depend, rerun] = createSignal(undefined, { equals: false });
// şimdi depend()'i izleme kapsamı içerisinde çağırmak
// o kapsamın her rerun() çağrıldığında yeniden yürütülmesine sebep olacak.

// eşitliği string uzunluğuna bağlı olarak tanımlayın:
const [myString, setMyString] = createSignal("string", {
  equals: (newVal, oldVal) => newVal.length === oldVal.length,
});

setMyString("strung"); // bir önceki değere eşit sayılacak ve güncelleme tetiklemeyecek.
setMyString("stranger"); // farklı sayılacak ve güncelleme tetikleyecek.
```

## `createEffect`

```ts
import { createEffect } from "solid-js";

function createEffect<T>(fn: (v: T) => T, value?: T): void;
```
