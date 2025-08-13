# Laravel Request Kullanımı Notları

Laravel'de `Illuminate\Http\Request` sınıfı, uygulamamıza gelen HTTP isteklerini (form verileri, URL parametreleri, başlıklar, cookie'ler vb.) yönetmek ve bu verilerle etkileşim kurmak için kullanılır.

## 1. Form Oluşturma ve CSRF Koruması

Laravel, **CSRF (Cross-Site Request Forgery)** ataklarına karşı yerleşik bir koruma mekanizması sunar. Güvenlik nedeniyle, `POST`, `PUT`, `PATCH`, `DELETE` gibi state-changing (durum değiştiren) istekler gönderen tüm formlarınızda bir CSRF token'ı bulundurmak zorunludur.

Bu token, `@csrf` Blade direktifi ile forma kolayca eklenebilir. Bu direktif, formun içine token değerini içeren gizli (`hidden`) bir input alanı ekler.

```html
<form action="{{ route('iletisim.post') }}" method="post">
    {{-- Bu satır, CSRF korumasını aktif eder --}}
    @csrf

    <input type="text" name="ad_soyad">
    <button type="submit">Gönder</button>
</form>
```

## 2. Örnek Senaryo: İletişim Formu

Aşağıda, bir iletişim formunun oluşturulması, rotasının tanımlanması ve Controller'da verilerin nasıl işleneceği adım adım gösterilmiştir.

### Adım 1: View Dosyası (`iletisim.blade.php`)

```php
// resources/views/iletisim.blade.php

<!doctype html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>İletişim Formu</title>
</head>
<body>
<form action="{{ route('iletisim.post') }}" method="post">
    @csrf
    <label for="ad_soyad">Ad Soyad</label>
    <input type="text" name="ad_soyad" id="ad_soyad"/><br>

    <label for="mail">Mail Adresi</label>
    <input type="email" name="mail" id="mail"/><br>

    <label for="subject">İletişim Sebebi</label>
    <select name="subject" id="subject">
        <option value="is_teklifi">İş Teklifi</option>
        <option value="destek">Destek</option>
    </select><br>

    <label for="mesaj">Mesajınız</label>
    <textarea name="mesaj" id="mesaj" cols="30" rows="10"></textarea><br>

    <button type="submit">Gönder</button>
</form>
</body>
</html>
```

### Adım 2: Route Tanımlaması (`web.php`)

Formu göstermek için bir `GET` ve form verilerini işlemek için bir `POST` rotası tanımlarız.

```php
// routes/web.php

use App\Http\Controllers\IletisimController;

// Formu gösteren sayfa
Route::get('/iletisim', [IletisimController::class, 'show'])->name('iletisim.show');

// Formdan gelen verileri yakalayan rota
Route::post('/iletisim', [IletisimController::class, 'post'])->name('iletisim.post');
```

### Adım 3: Controller (`IletisimController.php`)

Controller metoduna `Request $request` parametresini ekleyerek Laravel'in gelen isteğe ait tüm bilgileri bu nesneye doldurmasını sağlarız (Dependency Injection).

```php
// app/Http/Controllers/IletisimController.php

<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

class IletisimController extends Controller
{
    // Formu gösteren metod
    public function show()
    {
        return view('iletisim');
    }

    // Form verilerini işleyen metod
    public function post(Request $request)
    {
        // Gelen verileri görmek için "dump and die" kullanabiliriz.
        dd($request->all());
    }
}
```

## 3. Gelen Verilere Erişmek

Controller içinde `$request` nesnesi üzerinden form verilerine birkaç farklı yolla erişebiliriz.

```php
public function post(Request $request)
{
    // Yöntem 1: Dinamik özellik olarak erişim (Kısa ve pratik)
    $adSoyad = $request->ad_soyad;

    // Yöntem 2: input() metodu ile erişim (Tavsiye edilen)
    $mail = $request->input('mail');

    // Yöntem 3: input() metodu ile varsayılan değer atama
    // Eğer 'subject' alanı formdan gelmezse veya boşsa, varsayılan olarak 'Genel' atanır.
    $konu = $request->input('subject', 'Genel');

    // Tüm form verilerini bir dizi (array) olarak almak için:
    $tumVeriler = $request->all();

    // Sadece belirli verileri almak için:
    $seciliVeriler = $request->only(['ad_soyad', 'mail']);

    // Belirli veriler hariç diğerlerini almak için:
    $haricVeriler = $request->except(['_token']);


    return "Ad Soyad: " . $adSoyad . " | Mail: " . $mail;
}
```

## 4. Request Bilgilerini ve Yöntemini Kontrol Etme

`$request` nesnesi, gelen verilerin yanı sıra isteğin kendisi hakkında da birçok bilgi içerir.

```php
public function post(Request $request)
{
    // 1. Request metodunu string olarak alma (GET, POST, PUT, DELETE vb.)
    echo 'Request Metodu: ' . $request->method();

    // 2. Request metodunun belirli bir tipte olup olmadığını kontrol etme (true/false döner)
    if ($request->isMethod('post')) {
        echo '<br>Bu bir POST isteğidir.';
    }

    // 3. Gelen isteğin URL'ini kontrol etme
    // Bu, tek bir metod içinde farklı URL'lerden gelen istekleri yönetmek için kullanışlıdır.
    // `*` wildcard (joker karakter) olarak kullanılabilir.
    if ($request->is('iletisim')) {
        echo '<br>İstek URL\'i tam olarak "iletisim".';
    }

    if ($request->is('admin/*')) {
        echo '<br>Bu bir admin panel isteğidir.';
    }

    // 4. Request'in tam URL'ini alma
    echo '<br>Tam URL: ' . $request->url();

    // 5. Request'in geldiği IP adresini alma
    echo '<br>IP Adresi: ' . $request->ip();
}
```
