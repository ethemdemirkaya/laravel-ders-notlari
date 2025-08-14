# Laravel Middleware (Ara Katman) Kullanımı

Middleware, uygulamanıza gelen HTTP isteklerini filtrelemek için bir mekanizma sağlar. Bir isteğin, hedeflenen Controller metoduna ulaşmasından **önce** veya Controller'dan dönen yanıtın kullanıcıya gönderilmesinden **hemen sonra** araya girerek çeşitli işlemler yapmanızı sağlar.

Bunu bir binanın güvenlik katmanlarına benzetebiliriz: Bir odaya (Controller'a) girmeden önce kimlik kontrolünden (Authentication Middleware), yetki kontrolünden (Authorization Middleware) ve belki de çanta kontrolünden (CSRF Middleware) geçmeniz gerekir.

### Neden Middleware Kullanılır?

- **Authentication:** Kullanıcının giriş yapıp yapmadığını kontrol etmek.
- **Authorization:** Kullanıcının belirli bir sayfayı görmeye yetkisi olup olmadığını (rol/izin) kontrol etmek.
- **Logging:** Uygulamaya gelen tüm istekleri kaydetmek.
- **CORS:** Farklı domain'lerden gelen isteklere izin vermek veya engellemek.
- **Maintenance Mode:** Siteyi bakım moduna almak.

---

### Dikkat: Laravel 11 ve Sonrası Değişiklikler

Senin de belirttiğin gibi, Laravel 11 ile birlikte Middleware yapısı önemli ölçüde değişti:

1. **`app/Http/Kernel.php` Dosyası Kaldırıldı:** Artık middleware'leri kaydettiğimiz merkezi `Kernel.php` dosyası bulunmuyor.
2. **`app/Http/Middleware` Klasörü Varsayılan Değil:** `make:middleware` komutunu çalıştırana kadar bu klasör projenizde yer almaz. Komut çalıştırıldığında klasör otomatik olarak oluşturulur.
3. **Yeni Merkez: `bootstrap/app.php`:** Tüm middleware yapılandırması (global, alias, group) artık `bootstrap/app.php` dosyası içindeki `->withMiddleware()` metodu ile zincirleme (fluent) bir şekilde yapılıyor.

---

## 1. Middleware Oluşturma

Bir middleware oluşturmak için Artisan komutu kullanılır. Bu komut, `app/Http/Middleware` klasörünü (eğer yoksa) oluşturur ve içine belirttiğiniz isimde bir sınıf dosyası ekler.

```shell
php artisan make:middleware CheckAge
```

Bu komut, `app/Http/Middleware/CheckAge.php` dosyasını oluşturacaktır.

## 2. Middleware Sınıfının Yapısı

Oluşturulan sınıfın içinde en önemli metod `handle` metodudur.

```php
// app/Http/Middleware/CheckAge.php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class CheckAge
{
    /**
     * Gelen isteği işle.
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  \Closure(\Illuminate\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
     * @return \Symfony\Component\HttpFoundation\Response
     */
    public function handle(Request $request, Closure $next): Response
    {
        // 1. "BEFORE" Mantığı: İstek Controller'a gitmeden önce çalışır.
        // Örnek: Kullanıcının yaşı 18'den küçükse, 403 "Yasak" hatası ver ve işlemi durdur.
        if ($request->age < 18) {
            abort(403, 'Bu sayfaya erişmek için yaşınız yeterli değil.');
        }

        // $next($request), isteğin bir sonraki katmana (başka bir middleware veya Controller)
        // geçmesini sağlar. Eğer bu satır olmazsa, istek hedefine asla ulaşamaz.
        $response = $next($request);

        // 2. "AFTER" Mantığı: Controller çalışıp yanıtı oluşturduktan sonra çalışır.
        // Örnek: Oluşturulan yanıta özel bir başlık (header) ekleyebiliriz.
        $response->headers->set('X-Checked-By', 'Age-Middleware');

        return $response;
    }
}
```

## 3. Middleware'i Kaydetme (Laravel 11+ Yöntemi)

Oluşturduğumuz middleware'i Laravel'e tanıtmak için `bootstrap/app.php` dosyasını kullanırız.

```php
// bootstrap/app.php

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )
    ->withMiddleware(function (Middleware $middleware) {

        // A) Global Middleware: Her HTTP isteğinde çalışır.
        // $middleware->append(ForceJsonContentType::class);

        // B) Route Middleware (Alias): Rotalara takma isimle (alias) atamak için kullanılır.
        // Bu en yaygın kullanım şeklidir.
        $middleware->alias([
            'auth'       => \App\Http\Middleware\Authenticate::class,
            'guest'      => \App\Http\Middleware\RedirectIfAuthenticated::class,
            'check.age'  => \App\Http\Middleware\CheckAge::class, // Kendi middleware'imiz
        ]);

        // C) Middleware Grupları: Genellikle 'web' ve 'api' grupları burada tanımlanır.
        // Varsayılan olarak Laravel bu grupları sizin için zaten ayarlar.
        $middleware->group('web', [
            \Illuminate\Cookie\Middleware\EncryptCookies::class,
            \Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse::class,
            // ... diğer web middleware'leri
        ]);

    })
    ->withExceptions(function (Exceptions $exceptions) {
        // ...
    })->create();
```

## 4. Middleware'i Rotalara Uygulama

Middleware'i kaydettikten sonra `routes/web.php` dosyasında rotalara uygulayabiliriz.

### Tek bir Rota'ya Uygulama

```php
use App\Http\Controllers\DashboardController;

// 'check.age' takma adıyla kaydettiğimiz middleware'i kullanıyoruz.
Route::get('/yetiskin-paneli', [DashboardController::class, 'index'])
     ->middleware('check.age');
```

### Rota Grubuna Uygulama (En İyi Pratik)

Genellikle birden fazla middleware bir rota grubuna uygulanır.

```php
Route::middleware(['auth', 'check.age'])->group(function () {

    Route::get('/dashboard', [DashboardController::class, 'index']);
    Route::get('/profile', [ProfileController::class, 'show']);

    // Bu grup içindeki tüm rotalar, önce giriş kontrolünden ('auth'),
    // sonra da yaş kontrolünden ('check.age') geçecektir.
});
```

## 5. Middleware'e Parametre Gönderme

Bazen middleware'in dinamik çalışması için ona dışarıdan bir parametre göndermek isteyebiliriz. Örneğin, kullanıcının rolünü kontrol eden bir middleware.

**Örnek: `RoleMiddleware`**

```bash
php artisan make:middleware RoleMiddleware
```

```php
// app/Http/Middleware/RoleMiddleware.php
public function handle(Request $request, Closure $next, string $role): Response
{
    // Eğer kullanıcının rolü, rotadan gönderilen role ile eşleşmiyorsa...
    if (! $request->user() || ! $request->user()->hasRole($role)) {
        abort(403, 'BU İŞLEM İÇİN YETKİNİZ YOK.');
    }

    return $next($request);
}
```

**Kaydetme:**

```php
// bootstrap/app.php -> withMiddleware()
$middleware->alias([
    'role' => \App\Http\Middleware\RoleMiddleware::class,
]);
```

**Kullanım:**
Parametre, middleware adından sonra `:` ile belirtilir.

```php
// Sadece 'editor' rolüne sahip olanlar erişebilir.
Route::get('/makale/duzenle/{id}', ...)->middleware('role:editor');

// Sadece 'admin' rolüne sahip olanlar erişebilir.
Route::get('/kullanici/sil/{id}', ...)->middleware('role:admin');
```
