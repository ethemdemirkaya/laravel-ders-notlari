# Laravel'de Validation (Doğrulama) Kullanımı

Kullanıcıdan gelen hiçbir veriye güvenilmez. Validation, Controller'a gelen verilerin (form girdileri, URL parametreleri vb.) belirlediğimiz kurallara uygun olup olmadığını kontrol etme işlemidir. Bu, uygulamanın veri bütünlüğünü ve güvenliğini sağlamak için kritik bir adımdır.

## 1. Temel Yöntem: `request->validate()`

En hızlı ve yaygın kullanılan yöntemdir. Controller metodunun içinde doğrudan `$request` nesnesi üzerinden `validate()` metodu çağrılır.

- **Çalışma Prensibi:** Belirtilen kurallar geçerli değilse, Laravel kullanıcıyı **otomatik olarak geldiği sayfaya geri yönlendirir**. Tüm hataları (`$errors`) ve form girdilerini (`old()`) de session'a flash'lar (geçici olarak kaydeder).
- Kurallar `|` karakteri ile veya bir dizi (array) içinde ayrılarak yazılabilir.

```php
// app/Http/Controllers/KayitController.php

public function post(Request $request) {
    $request->validate([
        'username' => 'required|min:3|unique:users,username', // users tablosunda bu username'den başka olmamalı
        'email'    => ['required', 'email'], // Dizi olarak kullanım
        'password' => 'required|min:6|confirmed', // password_confirmation ile eşleşmeli
    ]);

    // Eğer kod bu satıra ulaştıysa, tüm doğrulamalar başarılıdır.
    // Veritabanına kayıt vb. işlemler burada yapılır.
    return "Kayıt Başarılı!";
}
```

> **`password_confirmation` kuralı:** Bu kuralın çalışması için formda `password` alanı ile aynı isme sahip ve `_confirmation` eki almış bir input olmalıdır. (`name="password_confirmation"`)

## 2. Hata Mesajlarını Blade'de Gösterme

Doğrulama başarısız olduğunda, Laravel hataları otomatik olarak tüm view dosyalarında erişilebilen `$errors` adında bir değişkene atar.

### Tüm Hataları Tek Seferde Listeleme

```php
// resources/views/kayit.blade.php

@if ($errors->any())
    <div style="background-color: #f8d7da; color: #721c24; padding: 1rem; border-radius: 5px; margin-bottom: 1rem;">
        <strong>Lütfen aşağıdaki hataları düzeltin:</strong>
        <ul>
            @foreach($errors->all() as $error)
                <li>{{ $error }}</li>
            @endforeach
        </ul>
    </div>
@endif
```

### Belirli Bir Alanın Hatasını Gösterme (`@error`)

Her input alanının altında kendi hatasını göstermek daha kullanıcı dostu bir yaklaşımdır. `@error` direktifi bu işi çok kolaylaştırır.

```html
<label for="username">Kullanıcı Adı</label>
<input type="text" name="username" id="username">
@error('username')
    <div style="color: red; font-size: 0.8rem;">{{ $message }}</div>
@enderror

<label for="password">Şifre</label>
<input type="password" name="password" id="password">
@error('password')
    <div style="color: red; font-size: 0.8rem;">{{ $message }}</div>
@enderror
```

## 3. Form Verilerini Koruma (`old()` Fonksiyonu)

Bir doğrulama hatası olduğunda kullanıcının tüm formu baştan doldurmasını engellemek için `old()` helper fonksiyonu kullanılır. Bu fonksiyon, bir önceki istekte gönderilen değeri input'un `value` kısmına yazar.

```html
<form action="{{ route('kayit.post') }}" method="post">
    @csrf

    <label for="username">Kullanıcı Adı</label>
    <input type="text" name="username" value="{{ old('username') }}" id="username"><br>

    <label for="email">Mail</label>
    <input type="email" name="email" value="{{ old('email') }}" id="email"><br>

    <label for="password">Şifre</label>
    <input type="password" name="password" id="password"><br>

    <label for="password_confirmation">Şifre Tekrar</label>
    <input type="password" name="password_confirmation" id="password_confirmation"><br>

    <button type="submit">Kayıt Ol</button>
</form>
```

---

## 4. Gelişmiş Validation Yöntemleri

### 4.1. Özel Hata Mesajları Tanımlama

Varsayılan İngilizce hata mesajlarını Türkçeleştirmek veya daha anlamlı mesajlar göstermek için `validate` metoduna ikinci bir dizi olarak kendi mesajlarınızı ekleyebilirsiniz.

- **Format:** `'alan_adi.kural' => 'Mesajınız'`
- `:attribute` anahtar kelimesi, otomatik olarak alan adını (username, email vb.) mesajın içine yerleştirir.

```php
public function post(Request $request) {
    $request->validate([
        'username' => 'required|min:3',
        'email'    => 'required|email',
        'password' => 'required|min:6',
    ], [
        'username.required' => 'Kullanıcı adı alanı boş bırakılamaz.',
        'username.min'      => 'Kullanıcı adı en az 3 karakter olmalıdır.',
        'email.required'    => 'E-posta alanı zorunludur.',
        'email.email'       => 'Lütfen geçerli bir e-posta adresi girin.',
        'password.min'      => ':attribute alanı en az :min karakter olmalıdır.', // "Şifre alanı en az 6 karakter olmalıdır."
    ]);

    // ...
}
```

### 4.2. Manuel Validator Oluşturma: `Validator::make()`

`request->validate()` otomatik yönlendirme yapar. Bazen bu yönlendirme üzerinde daha fazla kontrol sahibi olmak (örneğin bir AJAX isteğine JSON yanıtı dönmek) isteyebiliriz. Bu durumda `Validator` facade'ını kullanırız.

```php
use Illuminate\Support\Facades\Validator;

public function post(Request $request)
{
    $validator = Validator::make($request->all(), [
        'username' => 'required|min:3',
        'email'    => 'required|email',
    ]);

    // Doğrulama başarısız olursa...
    if ($validator->fails()) {
        return redirect()
            ->back() // Geldiği sayfaya geri dön
            ->withErrors($validator) // Hataları session'a flash'la
            ->withInput(); // Form girdilerini (old data) session'a flash'la
    }

    // Doğrulama başarılıysa kod buradan devam eder.
    // ...
}
```

### 4.3. En İyi Yöntem: Form Request Sınıfları

Karmaşık formlar için doğrulama kurallarını ve mesajlarını Controller içinde tutmak, Controller'ı şişirir ve kod tekrarına yol açar. Laravel'in en temiz çözümü, her form için ayrı bir **Request** sınıfı oluşturmaktır.

**Adım 1: Form Request Sınıfını Oluşturma**

Aşağıdaki Artisan komutu ile `app/Http/Requests` dizini altında `KayitRequest.php` adında bir sınıf oluşturulur.

```bash
php artisan make:request KayitRequest
```

**Adım 2: Sınıfı Düzenleme**

Oluşturulan `KayitRequest.php` dosyasını açıp düzenleyelim.

```php
// app/Http/Requests/KayitRequest.php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class KayitRequest extends FormRequest
{
    /**
     * Kullanıcının bu isteği yapmaya yetkisi olup olmadığını belirler.
     * Genellikle yetkilendirme (authorization) için kullanılır. Şimdilik true yapalım.
     * @return bool
     */
    public function authorize()
    {
        return true;
    }

    /**
     * İstek için geçerli olan doğrulama kurallarını döndürür.
     *
     * @return array
     */
    public function rules()
    {
        return [
            'username' => 'required|min:3|unique:users,username',
            'email'    => 'required|email|unique:users,email',
            'password' => 'required|min:6|confirmed',
        ];
    }

    /**
     * Kurallar için özel hata mesajları tanımlar.
     *
     * @return array
     */
    public function messages()
    {
        return [
            'username.required' => 'Kullanıcı adı alanı zorunludur.',
            'email.required'    => 'E-posta alanı zorunludur.',
            'password.min'      => 'Şifreniz en az 6 karakter olmalıdır.',
        ];
    }
}
```

**Adım 3: Controller'da Kullanma**

Artık Controller metodumuz inanılmaz derecede temiz ve sade! Parametre olarak `Request` yerine oluşturduğumuz `KayitRequest` sınıfını "type-hint" yaparız.

```php
// app/Http/Controllers/KayitController.php

// use Illuminate\Http\Request; // Buna artık gerek yok
use App\Http\Requests\KayitRequest; // Yeni request sınıfımızı dahil ediyoruz

class KayitController extends Controller
{
    // Laravel, bu metoda gelmeden önce KayitRequest içindeki kuralları otomatik çalıştırır.
    // Eğer doğrulama başarısız olursa, Controller'daki koda hiç ulaşmadan otomatik geri yönlendirir.
    public function post(KayitRequest $request)
    {
        // Kod bu satıra ulaştıysa, tüm doğrulamalar %100 başarılıdır.
        // Doğrulanmış verilere $request->validated() ile de erişebilirsiniz.
        $validatedData = $request->validated();

        // Veritabanı işlemleri...

        return "Kayıt Başarıyla Tamamlandı!";
    }
}
```

Bu yöntem, büyük projelerde kodun okunabilirliğini, bakımını ve yeniden kullanılabilirliğini ciddi ölçüde artırır.
