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

Efektler bağımlılıklar her değiştiğinde isteğe bağlı kodu ("yan efektler") yürüterek DOM'u manuel olarak manipüle etmek için kullanılan yaygın bir yöntemdir. `createEffect` verilen fonksiyonu izleme kapsamı içerisinde çağıran yeni bir hesaplama oluşturur, ve buna bağlı olarak bağımlılıkların otomatik takibe alınmasını ve bağımlılıklar her güncellendiğinde fonksiyonların otomatik olarak yeniden yürütülmesini sağlar.
Örneğin:

```js
const [a, setA] = createSignal(initialValue);

// `a` sinyaline bağımlı olan efekt
createEffect(() => doSideEffect(a()));
```

Efekt fonksiyonu, efekt fonksiyonunun son yürütülmesinden döndürülen değere eşit bir değer argümanıyla, ya da ilk çağrımda, `createEffect`'in isteğe bağlı ikinci argümanına eşit bir değer argümanıyla çağrılır.
Bu, farklılıkları son hesaplanan değeri hatırlamak için ekstra kapamalar yaratmadan hesaplamanızı sağlar.
Örneğin:

```js
createEffect((prev) => {
  const sum = a() + b();
  if (sum !== prev) console.log("sum changed to", sum);
  return sum;
}, 0);
```

Efektler öncelikli olarak reaktif sistemden okuyan ancak reaktif sisteme yazmayan yan efektler içindir:
efektlerin içerisinde sinyal oluşturmaktan kaçınılmalıdır, çünkü bu durum ekstra işlemeler ya da hatta sonsuz efekt döngüleri tetikleyebilir.
Onun yerine, reaktif değerlere bağımlı yeni değerler hesaplamak için [`createMemo`](#createMemo) kullanmayı tercih edin ki reaktif sistem neyin neye bağımlı olduğunu bilebilsin ve buna bağlı olarak optimizasyon yapabilsin.

Efekt fonksiyonunun _ilk_ yürütümü derhal değildir;
şimdiki işleme aşamasından sonrasında yürütülmek için sıraya alınır (örneğin [`render`](#render), [`createRoot`](#createroot), ya da [`runWithOwner`](#runwithowner) içerisine iletilmiş fonksiyonu çağırdıktan sonra.)
Eğer ilk yürütülmenin oluşmasını beklemek istiyorsanız, [`queueMicrotask`](https://developer.mozilla.org/en-US/docs/Web/API/queueMicrotask)'ı (`queueMicrotask` tarayıcı DOM'u işlemeden önce yürütülür) ya da `await Promise.resolve()` ya da `setTimeout(..., 0)` (bu ikili tarayıcı işlemesi gerçekleştikten sonra yürütülür.) kullanın.

```js
// bu kodun bir bileşen fonksiyonu içerisinde bulunduğunu farz edin, yani işleme aşamasının bir parçası olarak.
const [count, setCount] = createSignal(0);

// bu efekt count değerini başlangıçta ve değer her değiştiğinde yazdırır.
createEffect(() => console.log("count =", count()));
// efekt henüz yürütülmedi.
console.log("hello");
setCount(1); // efekt hala yürütülmedi.
setCount(2); // efekt hala yürütülmedi.

queueMicrotask(() => {
  // şimdi `count = 2` yazdırılacak.
  console.log("microtask");
  setCount(3); // anında `count = 3` yazdırıyor.
  console.log("goodbye");
});

// --- genel çıktı: ---
// hello
// count = 2
// microtask
// count = 3
// goodbye
```
