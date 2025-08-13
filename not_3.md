# Laravel Route Kullanımı Notları

Bu döküman, Laravel'de sık kullanılan route (rota) tanımlama yöntemlerini özetlemektedir.

## 1. Temel Route Tanımlaması (Controller ile)

En yaygın kullanım, bir URL'i bir Controller içerisindeki metoda yönlendirmektir.

- **URL:** `/kitap/{slug}`
- **Controller:** `Uygulama`
- **Method:** `kitap`

```php
// routes/web.php

use App\Http\Controllers\Uygulama;

Route::get('/kitap/{slug}', [Uygulama::class, 'kitap']);
```

## 2. Opsiyonel (İsteğe Bağlı) Parametreler

URL'deki bir parametrenin zorunlu olmamasını istiyorsak, parametre adının sonuna `?` işareti ekleriz.

**Önemli:** Route tanımında opsiyonel yapılan parametrelerin, ilgili Controller metodunda varsayılan bir değere (`null` veya başka bir değer) sahip olması gerekir.

- **Route Tanımı:**
  
  ```php
  // routes/web.php
  Route::get('/kitap/{slug}/{yil?}/{yazar?}', [Uygulama::class, 'kitap']);
  ```

- **Controller Metodu:**

```php
// app/Http/Controllers/Uygulama.php

public function kitap($slug, $yil = null, $yazar = null) {
    // Eğer $yil veya $yazar URL'de belirtilmediyse, değerleri null olacaktır.
    // ...
}
```

## 3. Route Kısıtlamaları (Constraints) - `where()`

Route parametrelerinin belirli bir formata uymasını zorunlu kılmak için `where` metodu ile regular expression (regex) kuralları tanımlayabiliriz. Bu, geçersiz formatları Controller'a ulaşmadan engeller.

Aşağıdaki örnekte `dogum_tarihi` parametresinin sadece 4 haneli bir sayı olmasını sağlıyoruz. Farklı bir formatta istek yapılırsa, Laravel 404 "Not Found" hatası döndürür.

```php
// routes/web.php

Route::get('/yas/{dogum_tarihi}', [Uygulama::class, 'yasHesapla'])
    ->where('dogum_tarihi', '[0-9]{4}');
```

> Diğer regex örnekleri:
> 
> - `->where('isim', '[A-Za-z]+');` // Sadece harfler
> - `->where('id', '[0-9]+');` // Sadece sayılar
> - `->whereAlpha('isim');` // Sadece harfler (kısayol)
> - `->whereNumber('id');` // Sadece sayılar (kısayol)
> - `->whereAlphaNumeric('kullanici_adi');` // Harf ve sayılar (kısayol)

## 4. İsimli Rotalar (Named Routes)

URL'leri kod içerisinde manuel yazmak yerine (`/hakkimizda` gibi), onlara bir isim atayarak daha esnek ve yönetilebilir bir yapı kurabiliriz. URL yapısı değişse bile, ismi çağırdığımız yerleri değiştirmemize gerek kalmaz.

`->name('rota-ismi')` şeklinde isimlendirme yapılır. View dosyalarında `route('rota-ismi')` helper fonksiyonu ile çağrılır.

### Örnek Dosyalar

- **`routes/web.php` (Rotaların Tanımlanması)**
  
  ```php
  <?php
  use App\Http\Controllers\Uygulama;
  use Illuminate\Support\Facades\Route;
  
  Route::get('/anasayfa', [Uygulama::class, 'anasayfa'])->name('anasayfa');
  Route::get('/hakkimizda', [Uygulama::class, 'hakkimizda'])->name('hakkimizda');
  ```

- **`app/Http/Controllers/Uygulama.php` (Controller)**

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

class Uygulama extends Controller
{
    public function anasayfa() {
        return view('anasayfa');
    }
    public function hakkimizda() {
        return view('hakkimizda');
    }
}
```

- **`resources/views/anasayfa.blade.php` (View'da Çağırma)**
  
  ```html
  <!DOCTYPE html>
  <html lang="en">
  <head>
    <meta charset="UTF-8">
    <title>Anasayfa</title>
  </head>
  <body>
    {{-- URL'i elle yazmak yerine ismini kullanıyoruz --}}
    <a href="{{ route('hakkimizda') }}">Hakkımızda Sayfasına Git</a>
  </body>
  </html>
  ```

### Parametreli İsimli Rotalar

Eğer isimlendirdiğimiz rota bir parametre alıyorsa, `route()` fonksiyonuna ikinci argüman olarak bu parametreyi gönderebiliriz.

- **Route Tanımı:**
  
  ```php
  // routes/web.php
  Route::get('/kategori/{slug}', [Uygulama::class, 'kategoriGoster'])->name('kategori.goster');
  ```

- **View'da Kullanımı:**

```html
<a href="{{ route('kategori.goster', 'teknoloji') }}">Teknoloji Kategorisi</a>
<a href="{{ route('kategori.goster', 'spor') }}">Spor Kategorisi</a>
```

> Oluşturulacak URL'ler sırasıyla `/kategori/teknoloji` ve `/kategori/spor` olacaktır.

## 5. Route Gruplama (Grouping)

Benzer özelliklere sahip rotaları (örneğin hepsi `/admin` ile başlayan veya aynı middleware'i kullanan) gruplayarak kod tekrarını azaltabiliriz.

### Prefix (URL Ön Eki) ile Gruplama

Aşağıdaki örnekte tüm rotaların başına otomatik olarak `/panel` ön eki eklenecektir.

```php
// routes/web.php

Route::prefix('panel')->name('panel.')->group(function () {

    Route::get('/anasayfa', [PanelController::class, 'anasayfa'])->name('anasayfa');
    // URL: /panel/anasayfa
    // Rota İsmi: panel.anasayfa

    Route::get('/kullanicilar', [PanelController::class, 'kullanicilar'])->name('kullanicilar');
    // URL: /panel/kullanicilar
    // Rota İsmi: panel.kullanicilar

    Route::get('/urunler/{id}', [PanelController::class, 'urunDetay'])->name('urun.detay');
    // URL: /panel/urunler/{id}
    // Rota İsmi: panel.urun.detay
});
```

> `name('panel.')` kullanımı, grup içindeki tüm rota isimlerinin başına `panel.` ekleyerek isimlendirmeyi de organize eder.

## 6. Sadece View Döndüren Rotalar (Route::view)

Eğer bir rota sadece statik bir view dosyasını göstermekle görevliyse, bunun için Controller'da bir metod oluşturmaya gerek yoktur. `Route::view()` metodu bu işi basitleştirir.

- **1. Parametre:** Erişilecek URL
- **2. Parametre:** Gösterilecek view dosyasının adı

```php
// routes/web.php

// /iletisim adresine gidildiğinde resources/views/sayfalar/iletisim.blade.php dosyasını gösterir.
Route::view('/iletisim', 'sayfalar.iletisim')->name('iletisim');
```
