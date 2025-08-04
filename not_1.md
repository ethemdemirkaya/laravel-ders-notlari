# Laravel Notları: Controller ve Route Temelleri

Bu notlar, Laravel'de Controller oluşturma, Route tanımlama ve ikisi arasındaki ilişkiyi anlamak için temel bilgileri içerir.

---

## 1. Controller Oluşturma

Yeni bir Controller dosyası oluşturmak için Artisan konsol komutunu kullanırız. Controller'lar, uygulamanızın mantığını gruplamak için `app/Http/Controllers` dizininde bulunur.

**Komut:**

```bash
# 'PostController' yerine kendi istediğiniz ismi yazabilirsiniz.
# İsimlendirmede 'Controller' sonekini kullanmak yaygın bir standarttır.
php artisan make:controller PostController
```

> **Not:** Orijinal nottaki `make:contoller` komutunda yazım hatası vardı, doğrusu `make:controller` şeklindedir.

---

## 2. Controller'ı Route'a Bağlama

Oluşturduğunuz bir Controller metodunu web üzerinden erişilebilir kılmak için `routes/web.php` dosyasında bir rota (route) tanımlamanız gerekir.

#### Örnek: `HomeController.php`

```php
// app/Http/Controllers/HomeController.php

<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

// DİKKAT: Controller ismini 'App' gibi sistem isimleriyle çakışacak şekilde kullanmayın.
// 'HomeController', 'PageController' gibi açıklayıcı isimler daha uygundur.
class HomeController extends Controller
{
    // Genellikle sitenin anasayfası için kullanılır.
    public function index()
    {
        return "Burası anasayfa";
    }

    // Hakkımızda sayfası için bir metod.
    public function hakkimizda()
    {
        return "Burası hakkımızda sayfası";
    }
}
```

#### Örnek: `web.php`

Bu Controller'daki metodlara rota tanımlayalım.

```php
// routes/web.php

<?php

use Illuminate\Support\Facades\Route;
// Oluşturduğumuz Controller'ı burada import etmemiz gerekir.
use App\Http\Controllers\HomeController;

// Tarayıcıda 'siteadresiniz.com/' adresine gidildiğinde HomeController'daki index metodu çalışır.
Route::get('/', [HomeController::class, 'index']);

// Tarayıcıda 'siteadresiniz.com/hakkimizda' adresine gidildiğinde hakkimizda metodu çalışır.
Route::get('/hakkimizda', [HomeController::class, 'hakkimizda']);
```

---

## 3. Route Üzerinden Veri Gönderme (Route Parametreleri)

Route tanımında süslü parantez `{}` kullanarak dinamik segmentler oluşturabilirsiniz. Bu segmentlerdeki değerler, Controller metodunuza parametre olarak gönderilir.

#### Örnek: `UserController.php`

```php
// app/Http/Controllers/UserController.php

<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

class UserController extends Controller
{
    // Route'daki {kullanici_adi} parametresi, $kullaniciAdi değişkenine atanır.
    public function profil($kullaniciAdi)
    {
        return "Kullanıcı Profili: " . $kullaniciAdi;
    }

    // Birden fazla parametre de alabiliriz.
    public function paylasim($kullaniciAdi, $paylasimId)
    {
        return $kullaniciAdi . " kullanıcısının " . $paylasimId . " ID'li paylaşımı.";
    }
}
```

#### Örnek: `web.php`

```php
// routes/web.php

use App\Http\Controllers\UserController;

// Örn: /uye/ahmet -> "Kullanıcı Profili: ahmet" çıktısı verir.
Route::get('/uye/{kullaniciAdi}', [UserController::class, 'profil']);

// Örn: /uye/ahmet/paylasim/123 -> "ahmet kullanıcısının 123 ID'li paylaşımı." çıktısı verir.
Route::get('/uye/{kullaniciAdi}/paylasim/{paylasimId}', [UserController::class, 'paylasim']);
```

---

## 4. Resource Controller Kullanımı

Sık kullanılan CRUD (Create, Read, Update, Delete) işlemleri için tek tek `get`, `post`, `put`, `delete` rotaları yazmak yerine Laravel'in `Route::resource()` metodunu kullanabiliriz. Bu, standart 7 rotayı tek bir satırda tanımlamamızı sağlar.

#### Resource Controller Oluşturma

`--resource` bayrağı (flag) ile bir Controller oluşturduğunuzda, Laravel sizin için tüm standart metodları (index, create, store, show, edit, update, destroy) hazır olarak oluşturur.

```bash
php artisan make:controller UrunController --resource
```

#### Örnek: `UrunController.php` (kısaltılmış hali)

```php
// app/Http/Controllers/UrunController.php

<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

class UrunController extends Controller
{
    /**
     * Tüm kaynakları listeler.
     * URI: GET /urunler
     */
    public function index()
    {
        return "Tüm ürünlerin listesi (index sayfası)";
    }

    /**
     * Belirtilen kaynağı gösterir.
     * URI: GET /urunler/{urun}
     */
    public function show($urun)
    {
        return "Tek bir ürün detayı gösteriliyor: " . $urun;
    }

    // Diğer metodlar (create, store, edit, update, destroy)
    // --resource flag'ı ile otomatik olarak buraya eklenir.
}
```

#### Örnek: `web.php`

```php
// routes/web.php

use App\Http\Controllers\UrunController;

// Bu tek satır, UrunController için tüm CRUD rotalarını otomatik olarak tanımlar.
Route::resource('urunler', UrunController::class);
```

Bu tek satırlık kodun sizin için tanımladığı rotalar şunlardır:

| Verb        | URI                    | Action  | Route Name      |
| ----------- | ---------------------- | ------- | --------------- |
| `GET`       | `/urunler`             | index   | urunler.index   |
| `GET`       | `/urunler/create`      | create  | urunler.create  |
| `POST`      | `/urunler`             | store   | urunler.store   |
| `GET`       | `/urunler/{urun}`      | show    | urunler.show    |
| `GET`       | `/urunler/{urun}/edit` | edit    | urunler.edit    |
| `PUT/PATCH` | `/urunler/{urun}`      | update  | urunler.update  |
| `DELETE`    | `/urunler/{urun}`      | destroy | urunler.destroy |
