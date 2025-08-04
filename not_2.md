# Laravel Notları: View ve Blade Şablon Motoru

Bu notlar, Laravel'de `View` katmanının nasıl çalıştığını, `Blade` şablon motorunun temel özelliklerini ve Controller'dan View'a nasıl veri aktarılacağını anlatmaktadır.

---

## 1. View Dosyaları ve Klasör Yapısı

View'lar, uygulamanızın HTML'ini içeren dosyalardır ve `resources/views` dizininde bulunurlar. Laravel, bu dosyalarda PHP kodunu daha okunaklı ve pratik bir şekilde yazmamızı sağlayan **Blade** adında bir şablon motoru kullanır. Blade dosyaları `.blade.php` uzantısıyla biter.

- **Kök Dizin:** `resources/views/anasayfa.blade.php` dosyasına `view('anasayfa')` şeklinde erişilir.
- **Alt Klasörler:** `resources/views/sayfalar/hakkimizda.blade.php` gibi bir dosyaya erişmek için klasör ve dosya adını nokta (`.`) ile ayırırız: `view('sayfalar.hakkimizda')`.

---

## 2. Controller'dan View Döndürme

Bir route'a istek geldiğinde, genellikle ilgili Controller metodu bir View döndürür. Bu işlem için global `view()` helper fonksiyonu kullanılır.

#### Örnek: `PageController.php`

```php
// app/Http/Controllers/PageController.php

<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

class PageController extends Controller
{
    public function anasayfa()
    {
        // resources/views/anasayfa.blade.php dosyasını render eder ve kullanıcıya gönderir.
        return view('anasayfa');
    }

    public function iletisim()
    {
        // resources/views/pages/iletisim.blade.php dosyasını render eder.
        return view('pages.iletisim');
    }
}
```

#### Örnek: `web.php`

```php
// routes/web.php

use App\Http\Controllers\PageController;

Route::get('/', [PageController::class, 'anasayfa']);
Route::get('/iletisim', [PageController::class, 'iletisim']);
```

---

## 3. View'a Veri Gönderme

Controller'daki verileri (örneğin veritabanından gelen bilgileri) View'a göndermek için birkaç yöntem vardır.

### Yöntem 1: Dizi (Array) Kullanarak

En yaygın yöntem, `view()` fonksiyonuna ikinci parametre olarak bir anahtar-değer (key-value) dizisi göndermektir. Dizideki her anahtar, Blade dosyasında bir değişken haline gelir.

```php
// PageController.php
public function anasayfa()
{
    $data = [
        'ad' => 'Ethem',
        'kullanicilar' => ['Ali', 'Veli', 'Ayşe']
    ];

    return view('anasayfa', $data);

    // Veya direkt olarak:
    // return view('anasayfa', ['ad' => 'Ethem', 'kullanicilar' => ['Ali', 'Veli', 'Ayşe']]);
}
```

### Yöntem 2: `with()` Metodu ile

`with()` metodunu zincirleme şekilde kullanarak da veri gönderebilirsiniz. Bu, kodu daha okunabilir hale getirebilir.

```php
// PageController.php
public function anasayfa()
{
    return view('anasayfa')
        ->with('ad', 'Ethem')
        ->with('kullanicilar', ['Ali', 'Veli', 'Ayşe']);
}
```

---

## 4. Blade İçerisinde Veri Gösterme

### Güvenli Gösterim: `{{ ... }}`

Blade, `{{ $degisken }}` sözdizimini kullanarak değişkenleri güvenli bir şekilde ekrana basar. Bu yapı, olası XSS (Cross-Site Scripting) saldırılarını önlemek için değişkenin içindeki HTML etiketlerini otomatik olarak etkisiz hale getirir.

> **Önemli:** Eski PHP `<?= $degisken ?>` yapısı yerine daima `{{ $degisken }}` kullanın. Bu, Laravel'in güvenlik katmanından faydalanmanızı sağlar.

```blade
<!-- resources/views/anasayfa.blade.php -->

<h1>Merhaba, {{ $ad }}!</h1>
```

### Güvenli Olmayan (Raw) HTML Gösterimi: `{!! ... !!}`

Eğer bir değişkendeki HTML kodunu (örneğin bir zengin metin editöründen gelen veriyi) render etmeniz gerekiyorsa `{!! $degisken !!}` sözdizimini kullanabilirsiniz.

> **UYARI:** Bu yapıyı **sadece güvendiğiniz** kaynaklardan gelen verilerle kullanın. Kullanıcıdan gelen verileri bu şekilde basmak, ciddi güvenlik açıklarına yol açabilir.

```php
// Controller'da
$data["htmlVerisi"] = "<b>Bu yazı kalın olacak</b>";
return view('anasayfa', $data);
```

```blade
<!-- Blade dosyasında -->
<div>
    {!! $htmlVerisi !!}
</div>
```

---

## 5. Blade Kontrol Yapıları (Directives)

Blade, PHP'nin kontrol yapılarının yerine geçen `@` ile başlayan pratik direktifler sunar.

### Koşul İfadeleri (`@if`, `@elseif`, `@else`, `@endif`)

```blade
@if(count($kullanicilar) === 1)
    Sadece bir kullanıcı var.
@elseif(count($kullanicilar) > 1)
    Birden fazla kullanıcı var.
@else
    Hiç kullanıcı yok.
@endif

{{-- Değişkenin tanımlı olup olmadığını kontrol eder --}}
@isset($ad)
    'ad' değişkeni mevcut ve değeri: {{ $ad }}
@endisset
```

### Döngüler (`@foreach` ve `$loop` Değişkeni)

`@foreach` direktifi ile diziler veya koleksiyonlar üzerinde kolayca döngü kurabilirsiniz. Blade, her `@foreach` döngüsü içinde otomatik olarak bir `$loop` değişkeni oluşturur. Bu değişken, döngü hakkında yararlı bilgiler içerir.

```blade
<ul>
    @foreach($kullanicilar as $kullanici)
        <li style="@if($loop->first) color: red; @endif">
            {{ $loop->iteration }}. kullanıcı: {{ $kullanici }}

            @if($loop->last)
                (Bu son kullanıcıdır)
            @endif
        </li>
    @endforeach
</ul>
```

Bazı kullanışlı `$loop` özellikleri:

- `$loop->first`: Döngünün ilk elemanı ise `true` döner.
- `$loop->last`: Döngünün son elemanı ise `true` döner.
- `$loop->index`: Döngünün 0'dan başlayan index'i.
- `$loop->iteration`: Döngünün 1'den başlayan sayacı.
- `$loop->count`: Dizideki toplam eleman sayısı.

---

## 6. Alt View'ları Dahil Etme (`@include`)

Tekrar eden HTML bloklarını (header, footer, sidebar vb.) ayrı dosyalara bölüp `@include` ile istediğiniz yere dahil edebilirsiniz. Bu, kod tekrarını önler.

```blade
<!-- resources/views/layouts/header.blade.php dosyası olsun -->
<header>
    <h1>Sitemin Başlığı</h1>
</header>
```

```blade
<!-- resources/views/anasayfa.blade.php -->
<body>
    @include('layouts.header')

    <main>
        <p>Burası sayfa içeriği.</p>
    </main>

    @include('layouts.footer')
</body>
```

### `@include` ile Veri Gönderme

Dahil ettiğiniz view dosyasına özel veri gönderebilirsiniz.

```blade
@include('layouts.header', ['sayfaBasligi' => 'Anasayfa'])
```

---

## 7. Saf PHP Kodu Kullanımı (`@php`)

Nadiren de olsa, Blade içinde saf PHP kodu yazmanız gerekebilir. Bunu `@php` ve `@endphp` direktifleri arasında yapabilirsiniz.

> **En İyi Pratik:** Karmaşık mantığı Controller'da tutup, View'ı sadece veri göstermek için kullanmaya çalışın. `@php` direktifini zorunlu olmadıkça kullanmaktan kaçının.

```blade
@php
    $sayac = 1;
    $mesaj = "Bu PHP içinde tanımlandı.";
@endphp

<h3>SAYAÇ: {{ $sayac }}</h3>
<p>{{ $mesaj }}</p>
```
