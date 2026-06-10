# Analiza projektu RentOpinion — odpowiedzi na pytania

Projekt **RentOpinion** to portal z recenzjami wynajmowanych mieszkań, zbudowany w Laravel (PHP 8.5 / Laravel 13) z widokami Blade i bazą SQLite. Uwierzytelnianie oparte jest na Laravel Breeze.

---

## 1. Struktura bazy danych + przykładowa tabela i dane początkowe

Strukturę przygotowano w mechanizmie **migracji** Laravela (`database/migrations/`). Każda tabela ma osobny plik z metodą `up()` (tworzenie) i `down()` (wycofanie). Są cztery główne tabele domenowe: `users`, `properties`, `property_images`, `reviews` — plus tabele systemowe (`sessions`, `cache`, `jobs`, `password_reset_tokens`).

Przykład — tabela **`reviews`** (`2024_01_01_000004_create_reviews_table.php`). Najważniejsze pola:

```php
$table->foreignId('property_id')->constrained()->cascadeOnDelete();
$table->foreignId('user_id')->constrained()->cascadeOnDelete();
$table->unsignedTinyInteger('rating_overall');   // ocena 1–5
$table->unsignedTinyInteger('rating_landlord');
// ... kolejne oceny cząstkowe
$table->string('title', 200);
$table->text('content');
$table->boolean('is_verified')->default(false);   // moderacja
$table->boolean('is_hidden')->default(false);
$table->foreignId('verified_by')->nullable()->constrained('users')->nullOnDelete();
$table->softDeletes();
$table->unique(['property_id', 'user_id']);        // jedna recenzja na ogłoszenie
$table->index(['property_id', 'is_hidden']);
```

Warto zwrócić uwagę na trzy decyzje: `unique(['property_id','user_id'])` gwarantuje na poziomie bazy, że jeden użytkownik nie wystawi dwóch recenzji tego samego ogłoszenia; `cascadeOnDelete` usuwa recenzje wraz z ogłoszeniem/użytkownikiem; `softDeletes()` powoduje, że rekord nie znika fizycznie, tylko dostaje datę w `deleted_at`.

**Dane początkowe** pochodzą z dwóch źródeł:

- **Seedery** (`database/seeders/`) — uruchamiane przez `php artisan db:seed`. `DatabaseSeeder` wywołuje kolejno `AdminSeeder`, `UserSeeder`, `PropertySeeder`, `ReviewSeeder`. `AdminSeeder` tworzy stałe konto administratora metodą `updateOrCreate` (idempotentnie, więc da się go uruchomić wielokrotnie):

```php
User::updateOrCreate(
    ['email' => 'admin@rentopinion.pl'],
    ['name' => 'Administrator', 'role' => UserRole::Admin,
     'password' => Hash::make(env('ADMIN_PASSWORD', 'Admin1234!')), ...]
);
```

`PropertySeeder` ma zaszytą tablicę realistycznych ogłoszeń z polskich miast (Kraków, Warszawa…).

- **Fabryki** (`database/factories/`) — generują losowe, ale sensowne dane (np. `ReviewFactory` losuje oceny `numberBetween(1,5)` i polskie zdania przez `fake('pl_PL')`). Używane głównie w testach i do masowego zasilania.

---

## 2. Zabezpieczenie formularzy przed wysłaniem „z zewnątrz" (CSRF)

Mechanizmem jest **token CSRF**. Każdy formularz modyfikujący dane zawiera dyrektywę `@csrf`, która renderuje ukryte pole z tokenem sesji. Przykład — formularz dodawania recenzji (`resources/views/properties/_review-form-multi.blade.php`):

```blade
<form method="POST" action="{{ route('reviews.store', $property) }}" id="review-form">
    @csrf
    ...
</form>
```

Globalny middleware web (`VerifyCsrfToken`, domyślnie włączony w stosie `web`) przy każdym żądaniu `POST/PATCH/DELETE` porównuje token z formularza z tokenem zapisanym w sesji użytkownika. Jeśli się nie zgadzają (np. żądanie pochodzi z obcej strony, która nie zna tokenu), Laravel zwraca błąd **419 Page Expired** i żądanie nie dociera do kontrolera. Dzięki temu atakujący nie może spreparować strony, która w tle wyśle formularz w imieniu zalogowanego użytkownika. Token jest powiązany z sesją, więc skuteczne podrobienie wymagałoby jego znajomości — co jest możliwe tylko ze strony serwowanej przez samą aplikację.

---

## 3. Reakcja na sytuacje nietypowe

Aplikacja obsługuje kilka takich przypadków:

**Nieistniejący lub zablokowany zasób.** W `PropertyController::show` jest jawna instrukcja:

```php
abort_if($property->isBlocked(), 404);
```

Jeśli ogłoszenie ma status `blocked`, zwykły użytkownik dostaje stronę **404** zamiast podglądu — celowo, aby nie zdradzać, że taki zasób istnieje. Gdy ktoś wejdzie na ID, którego nie ma w bazie, mechanizm **route model binding** sam zgłasza 404 (Laravel nie znajduje rekordu i przerywa).

**Brak uprawnień.** Przy próbie usunięcia cudzej recenzji `$this->authorize('delete', $review)` rzuca wyjątek **403 Forbidden**. Wejście na panel admina bez roli admina — `AdminMiddleware` wywołuje `abort(403, 'Brak dostępu do panelu administracyjnego.')`.

**Konto zablokowane.** `EnsureUserNotBlocked` (middleware globalny) przy każdym żądaniu sprawdza `Auth::user()->isBlocked()`; jeśli tak — wylogowuje, unieważnia sesję i przekierowuje na logowanie z komunikatem o blokadzie.

**Pusta lista wyników.** Widoki list używają warunku `@forelse … @empty`, więc zamiast pustej strony użytkownik widzi czytelny komunikat „brak wyników".

Co widzi użytkownik: w zależności od przypadku to strona błędu 403/404/419 albo komunikat flash (`with('success'…)`, `withErrors(...)`) wyświetlany w layoutie.

---

## 4. Jak aplikacja rozpoznaje intencję użytkownika

Rozpoznanie opiera się na **routingu RESTowym** — kombinacji metody HTTP + ścieżki URL, mapowanej na konkretną metodę kontrolera (`routes/web.php`):

| Intencja | Metoda HTTP | URL | Akcja |
|---|---|---|---|
| Wyświetlić listę | GET | `/ogloszenia` | `PropertyController@index` |
| Wyświetlić szczegóły | GET | `/ogloszenia/{property}` | `PropertyController@show` |
| Formularz dodania | GET | `/ogloszenia/dodaj` | `create` |
| Utworzyć | POST | `/ogloszenia` | `store` |
| Edytować | PATCH | `/ogloszenia/{property}` | `update` |
| Usunąć | DELETE | `/ogloszenia/{property}` | `destroy` |

Ta sama ścieżka `/ogloszenia/{property}` znaczy co innego w zależności od metody (GET = pokaż, PATCH = aktualizuj, DELETE = usuń). Ponieważ przeglądarka z formularza HTML potrafi wysłać tylko GET/POST, do PATCH/DELETE używana jest dyrektywa `@method('PATCH')`/`@method('DELETE')` (ukryte pole `_method`), które Laravel tłumaczy na właściwą metodę.

Ważny szczegół projektu — kolejność tras. Konkretne ścieżki (`/dodaj`, `/dodaj-pokoje`) są zdefiniowane **przed** wildcardem `/{property}`, inaczej Laravel potraktowałby słowo „dodaj" jako wartość `{property}`. Fragment kodu obsługujący np. tworzenie:

```php
Route::post('/', [PropertyController::class, 'store'])->name('store');
```

---

## 5. Miejsce do poprawy technicznej

**Problem: niespójna obsługa akcji `verify` na recenzji.** Istnieją dwie niemal identyczne metody weryfikacji — `ReviewController::verify` i `Admin\ReviewModerationController::verify` — pod dwiema różnymi trasami (`reviews.verify` i `admin.reviews.verify`). Dodatkowo trasa zwykłego użytkownika `recenzje/{review}/weryfikuj` jest dostępna dla każdego zalogowanego (chroni ją dopiero `authorize` wewnątrz metody), choć to operacja czysto administracyjna.

**Wpływ:** zdublowana logika łatwo się rozjeżdża przy zmianach (poprawisz jedną, zapomnisz drugą), a trasa „użytkownika" wykonująca akcję admina to mylące rozmycie odpowiedzialności i potencjalne ryzyko, jeśli ktoś osłabi Policy.

**Propozycja:** usunąć `verify` z `ReviewController` i z bloku tras użytkownika, a w widoku ogłoszenia kierować przycisk weryfikacji na jedną, administracyjną trasę `admin.reviews.verify`. Wtedy jedna metoda, jeden punkt wejścia, spójna ochrona przez `auth + admin`.

Drugi kandydat (mniejszy): `markHelpful` blokuje wielokrotne kliknięcia tylko przez sesję (`session()->has(...)`) — wyczyszczenie ciasteczek resetuje licznik. Docelowo lepsza byłaby osobna tabela `review_helpful` z parą (review_id, user_id/IP).

---

## 6. Walidacja danych wpisanych przez użytkownika

Walidacja jest wydzielona do klas **Form Request** (`app/Http/Requests/`), a nie wpisana w kontroler. Laravel uruchamia ją automatycznie, zanim kod kontrolera w ogóle się wykona. Przykład — `StoreMultiCriteriaReviewRequest`:

```php
public function rules(): array {
    return [
        'title'                  => ['required', 'string', 'min:5', 'max:200'],
        'content'                => ['required', 'string', 'min:150', 'max:5000'],
        'rating_overall'         => ['required', 'integer', 'between:1,5'],
        'rating_landlord'        => ['required', 'integer', 'between:1,5'],
        'tenancy_end'            => ['nullable','date','before_or_equal:today','after_or_equal:tenancy_start'],
        // ...
    ];
}
```

Sprawdzane są m.in.: obecność i długość tytułu oraz treści (treść min. 150 znaków), zakres ocen 1–5, poprawność i logika dat najmu (koniec nie wcześniej niż początek, nie w przyszłości). W formularzu dodawania ogłoszenia walidowany jest też kod pocztowy regułą `regex:/^\d{2}-\d{3}$/` i pliki zdjęć (`image`, `mimes`, `max:5120`).

Wykrywalne błędy: brak wymaganego pola, za krótka treść, ocena spoza 1–5, zły format kodu pocztowego, niepoprawny plik. Co widzi użytkownik: jeśli dane są błędne, Laravel zawraca go na poprzednią stronę z zachowaniem wpisanych wartości (`old(...)`) i wypełnioną tablicą `$errors`. Widok wyświetla je zbiorczo na górze (`@if($errors->any())`) i przy konkretnych polach (`@error('content')`), z czytelnymi polskimi komunikatami zdefiniowanymi w metodzie `messages()` (np. „Treść recenzji musi mieć co najmniej 150 znaków.").

---

## 7. Dobrze zorganizowany widok

Dobrym przykładem jest **komponent `property-card.blade.php`** używany na liście ogłoszeń i na stronie głównej. Jest czytelny, bo każda sekcja (zdjęcie, cena, tytuł, lokalizacja, parametry) jest jasno wyodrębniona komentarzami i ma jedną odpowiedzialność.

Najważniejsza zaleta: **unika powtórzeń przez wyniesienie do komponentu**. Zamiast kopiować kod karty na liście, mapie i dashboardzie, wystarczy wywołać `<x-property-card :property="$property"/>`. Gdyby trzeba było zmienić wygląd karty (np. dodać badge „Nowe"), poprawia się jeden plik, a zmiana pojawia się wszędzie. Widok korzysta też z **akcesorów modelu** (`$property->formatted_price`, `$property->coverImage`) zamiast formatować cenę w szablonie — logika prezentacji jest w jednym miejscu. Rozbudowa jest łatwa: dorzucenie nowego pola to jedna linijka w komponencie. Podobnie wydzielone są `review-card` i `star-rating` — projekt konsekwentnie stosuje komponenty Blade.

---

## 8. Rozróżnianie użytkowników o różnych uprawnieniach

Poziomy uprawnień są opisane **enumem ról** `UserRole` (`Guest`, `User`, `Admin`) zapisanym w kolumnie `users.role`. Model `User` ma metody pomocnicze `isAdmin()`, `isUser()`, `canWriteReviews()`, `isBlocked()`.

Kontrola działa na trzech warstwach:

1. **Middleware na trasach** — blok `admin` chroni cały panel (`AdminMiddleware` rzuca 403, jeśli rola ≠ Admin); `auth`, `verified`, `not.blocked` pilnują, by zalogowany, zweryfikowany i niezablokowany użytkownik mógł dodawać treści.
2. **Policy** (`PropertyPolicy`, `ReviewPolicy`) — decydują o pojedynczych obiektach. Mają metodę `before()`, która adminowi przyznaje wszystko z góry:

```php
public function before(User $user): ?bool {
    return $user->isAdmin() ? true : null;
}
public function delete(User $user, Review $review): bool {
    return $review->user_id === $user->id && ! $user->isBlocked();
}
```

3. **Form Request `authorize()`** — np. blokuje recenzowanie własnego ogłoszenia.

**Konkretny przykład:** użytkownik A napisał recenzję ogłoszenia. Tylko A widzi przy niej przycisk „Usuń" i tylko on może wykonać `DELETE /ogloszenia/{property}/recenzje/{review}` — bo `ReviewPolicy::delete` sprawdza `$review->user_id === $user->id`. Użytkownik B, próbując usunąć cudzą recenzję, dostaje **403**. Wyjątkiem jest administrator: dzięki `before()` zwracającemu `true` może usunąć każdą recenzję. Tej samej akcji A nie może wykonać na recenzji B — i właśnie na tym polega rozróżnienie.

---

## 9. Pełny przepływ jednej akcji: dodanie recenzji

Prześledźmy `POST /ogloszenia/{property}/recenzje` — od kliknięcia „Opublikuj recenzję" do zapisu i zwrócenia widoku. Żądanie przechodzi przez warstwy: **widok → routing → middleware → Form Request (walidacja + autoryzacja) → kontroler → model/Eloquent → baza → zdarzenie modelu → przekierowanie do widoku**.

**Etap 1 — przechwycenie danych z formularza** (`resources/views/properties/_review-form-multi.blade.php`). Formularz wysyła POST z tokenem CSRF i polami ocen/treści:

```blade
<form method="POST" action="{{ route('reviews.store', $property) }}" id="review-form">
    @csrf
    <input type="hidden" name="rating_overall" value="{{ old('rating_overall', 0) }}">
    <textarea name="content" ...>{{ old('content') }}</textarea>
    <button type="submit">🚀 Opublikuj recenzję</button>
</form>
```

**Etap 2 — routing + middleware** (`routes/web.php`). Trasa znajduje się w bloku chronionym `auth + verified + not.blocked`, więc niezalogowany lub zablokowany w ogóle tu nie dotrze:

```php
Route::prefix('ogloszenia/{property}/recenzje')->name('reviews.')->group(function () {
    Route::post('/', [ReviewController::class, 'store'])->name('store');
});
```

**Etap 3 — autoryzacja i walidacja** (`StoreMultiCriteriaReviewRequest`). Zanim kontroler się uruchomi, sprawdzane są uprawnienia (m.in. zakaz recenzowania własnego ogłoszenia i tylko jedna recenzja na ogłoszenie) oraz reguły pól:

```php
public function authorize(): bool {
    $user = $this->user(); $property = $this->route('property');
    if (!$user || !$user->canWriteReviews() || $user->isBlocked()) return false;
    if ($property->isOwnedBy($user)) return false;                          // nie swoje
    if ($property->reviews()->where('user_id',$user->id)->exists()) return false; // jedna recenzja
    return true;
}
```

**Etap 4 — główna logika biznesowa** (`ReviewController::store`). Dane już są zwalidowane; kontroler dolicza brakującą ocenę czystości jako średnią pozostałych, dopisuje ID autora i tworzy rekord przez relację:

```php
public function store(StoreMultiCriteriaReviewRequest $request, Property $property): RedirectResponse {
    $validated = $request->validated();
    $validated['rating_cleanliness'] = (int) round(($validated['rating_overall']
        + $validated['rating_landlord'] + $validated['rating_noise_level']
        + $validated['rating_location'] + $validated['rating_value_for_money']) / 5);
    $validated['user_id'] = $request->user()->id;
    $property->reviews()->create($validated);          // ZAPIS przez relację hasMany
    return redirect()->route('properties.show', $property)
        ->with('success', 'Twoja recenzja została dodana i oczekuje na moderację.');
}
```

**Etap 5 — zapis do bazy + efekt uboczny w modelu** (`app/Models/Review.php`). `create()` wykonuje `INSERT` do tabeli `reviews`. Po zapisie odpala się zdarzenie `saved`, które automatycznie przelicza średnią ocenę ogłoszenia:

```php
protected static function booted(): void {
    static::saved(fn (Review $review) => $review->property?->recalculateRating());
    static::deleted(fn (Review $review) => $review->property?->recalculateRating());
}
```

`recalculateRating()` w modelu `Property` liczy średnią z nieukrytych recenzji i zapisuje ją przez `updateQuietly()` (cicho — żeby nie wywołać kolejnego zdarzenia i nie zapętlić się).

**Etap 6 — zwrócenie widoku.** Kontroler przekierowuje na `properties.show`; przeglądarka robi GET, `PropertyController::show` ładuje ogłoszenie z recenzjami, a użytkownik widzi swoją recenzję oraz komunikat flash „oczekuje na moderację".

---

## 10. Przykład powiązania danych

Najczytelniejsze powiązanie: **ogłoszenie (Property) i jego recenzje (Review)** — relacja jeden-do-wielu.

**W bazie danych:** tabela `reviews` ma klucz obcy `property_id` wskazujący na `properties.id`, z regułą `cascadeOnDelete` (usunięcie ogłoszenia kasuje jego recenzje) i indeksem `['property_id','is_hidden']`.

**W kodzie:** w modelu `Property` relacja jest zadeklarowana w obie strony:

```php
public function reviews(): HasMany { return $this->hasMany(Review::class); }
public function visibleReviews(): HasMany {
    return $this->hasMany(Review::class)->where('is_hidden', false);
}
```

a w `Review` strona odwrotna: `public function property(): BelongsTo { return $this->belongsTo(Property::class); }`. Dzięki temu można pisać `$property->reviews()->create(...)` (tworzenie z automatycznym ustawieniem `property_id`) albo `$review->property` (sięgnięcie do ogłoszenia).

**W widoku:** na liście ogłoszeń pokazywana jest liczba i średnia recenzji (`withCount`, `withAvg`), a na stronie szczegółów pętla `@foreach($property->visibleReviews as $review)` renderuje karty recenzji. To samo powiązanie jest więc widoczne na wszystkich trzech poziomach.

(Analogiczne relacje: `User → properties`, `User → reviews`, `Property → images`.)

---

## 11. Funkcjonalność niewidoczna w interfejsie

Dobrym przykładem jest **automatyczne przeliczanie średniej oceny ogłoszenia** opisane w punkcie 9 — `Review::booted()` + `Property::recalculateRating()`. Użytkownik nigdzie nie klika „przelicz ocenę"; dzieje się to samoczynnie po każdym dodaniu, edycji czy usunięciu recenzji. Bez tego pole `avg_rating` szybko stałoby się nieaktualne, a sortowanie i karty ogłoszeń pokazywałyby błędne wartości. Zapis przez `updateQuietly()` chroni przed nieskończoną pętlą zdarzeń.

Druga taka funkcja to **`AnonymousPseudonymService`** (`app/Services/`): deterministycznie zamienia ID użytkownika na pseudonim typu „Spokojny Bóbr" (CRC32 z ID → indeks w listach przymiotników i zwierząt). Dzięki determinizmowi ten sam autor zawsze ma ten sam pseudonim (czytelnik widzi spójną „osobę"), ale jego tożsamość pozostaje ukryta. To kluczowe dla idei recenzji najmu — ludzie chętniej krytykują wynajmującego, jeśli nie grozi to ujawnieniem nazwiska.

Trzecia — **`TrackPropertyView`**: middleware zliczający unikalne odsłony ogłoszenia raz na sesję.

---

## 12. Dlaczego projekt zasługuje na dobrą ocenę

Projekt jest kompletny i dojrzały technicznie jak na pracę zaliczeniową. Realizuje pełen cykl CRUD dla ogłoszeń i recenzji, z czytelnym, RESTowym routingiem i polskimi, semantycznymi adresami (`/ogloszenia/dodaj`, `/admin/uzytkownicy`). Walidacja jest poprawnie wydzielona do klas Form Request z regułami biznesowymi (jedna recenzja na ogłoszenie, zakaz recenzowania własnego mieszkania) i własnymi komunikatami błędów. Uprawnienia zrobiono wzorcowo na trzech warstwach: middleware, Policy z metodą `before()` dla admina oraz `authorize()` w żądaniach — co daje realne rozdzielenie ról gość/użytkownik/admin. Baza ma przemyślany schemat z kluczami obcymi, indeksami, ograniczeniem `unique`, soft-deletes i kaskadami. Widoczna jest dbałość o jakość: komponenty Blade eliminujące powtórzenia, akcesory i scope'y w modelach, usługi (`Service`) wydzielające logikę pseudonimów i zdjęć, automatyczne przeliczanie ocen przez zdarzenia modelu oraz zabezpieczenie CSRF i obsługa błędów 403/404. Są też testy (`tests/Feature`) i seedery zasilające aplikację realistycznymi danymi. Słabsze punkty (zdublowana akcja `verify`, licznik „pomocne" oparty tylko na sesji) są drobne i lokalne. Całość pokazuje rozumienie architektury Laravela, a nie tylko jej odtworzenie.

---

## 13. Co poprawić w pierwszej kolejności

**Uporządkować weryfikację recenzji do jednej, administracyjnej ścieżki** (opisanej w punkcie 5): usunąć metodę `verify` i trasę z poziomu zwykłego użytkownika, zostawiając wyłącznie `admin.reviews.verify` chronioną przez `auth + admin`.

Dlaczego to najważniejsze: weryfikacja recenzji to operacja zaufania — to ona decyduje, którym opiniom portal nadaje status „zweryfikowana". Trzymanie jej akcji częściowo w przestrzeni zwykłego użytkownika (chronionej dopiero wewnętrznym `authorize`) rozmywa granicę uprawnień i jest najbardziej wrażliwym punktem z perspektywy bezpieczeństwa i wiarygodności danych. Skonsolidowanie tej logiki usuwa duplikację, zmniejsza ryzyko regresji przy zmianach i czyni model uprawnień w pełni spójnym — a właśnie spójność uprawnień jest tu kryterium o największej wadze.

---

## Uwaga końcowa

Pytanie o **porównanie do innego projektu z grupy** (mocniejszy / słabszy) pominięto — w dostarczonym archiwum znajduje się tylko ten jeden projekt, więc brak danych do rzetelnego zestawienia z pracami innych osób.