# RentOpinion — konkretne odpowiedzi na pytania (z fragmentami kodu)

Projekt: aplikacja Laravel (PHP), baza SQLite, widoki Blade, logowanie na Laravel Breeze. Portal z recenzjami wynajmowanych mieszkań.

---

## 1. Struktura bazy danych, przykładowa tabela i dane początkowe

Bazę zbudowano za pomocą **migracji** w katalogu `database/migrations/`. Główne tabele: `users`, `properties`, `property_images`, `reviews`.

Przykładowa tabela — **`reviews`** (`database/migrations/2024_01_01_000004_create_reviews_table.php`):

```php
Schema::create('reviews', function (Blueprint $table) {
    $table->id();
    $table->foreignId('property_id')->constrained()->cascadeOnDelete();
    $table->foreignId('user_id')->constrained()->cascadeOnDelete();

    $table->unsignedTinyInteger('rating_overall');          // oceny 1–5
    $table->unsignedTinyInteger('rating_location');
    $table->unsignedTinyInteger('rating_landlord');
    // ...
    $table->string('title', 200);
    $table->text('content');

    $table->boolean('is_verified')->default(false);          // moderacja
    $table->boolean('is_hidden')->default(false);
    $table->foreignId('verified_by')->nullable()->constrained('users')->nullOnDelete();

    $table->timestamps();
    $table->softDeletes();

    $table->unique(['property_id', 'user_id']);              // jedna recenzja na ogłoszenie
    $table->index(['property_id', 'is_hidden']);
});
```

Najważniejsze pola: `property_id` i `user_id` (do kogo/czego należy recenzja), oceny `rating_*` (1–5), `title` i `content` (treść), `is_verified`/`is_hidden` (moderacja). Kluczowe reguły: `unique(['property_id','user_id'])` — jeden użytkownik = jedna recenzja na ogłoszenie; `cascadeOnDelete` — recenzje znikają wraz z ogłoszeniem; `softDeletes()` — usunięcie tylko oznacza rekord, nie kasuje go fizycznie.

Dane początkowe pochodzą z **seederów** (`database/seeders/`). Np. konto admina (`AdminSeeder.php`):

```php
User::updateOrCreate(
    ['email' => 'admin@rentopinion.pl'],
    [
        'name'     => 'Administrator',
        'password' => Hash::make(env('ADMIN_PASSWORD', 'Admin1234!')),
        'role'     => UserRole::Admin,
        'email_verified_at' => now(),
    ]
);
```

`PropertySeeder` wpisuje gotowe ogłoszenia z polskich miast, a fabryki (`database/factories/ReviewFactory.php`) generują losowe dane testowe (`'rating_overall' => fake()->numberBetween(1, 5)`).

---

## 2. Zabezpieczenie formularzy przed wysłaniem z zewnątrz (CSRF)

Mechanizm to **token CSRF**. Każdy formularz zmieniający dane ma dyrektywę `@csrf`. Przykład — formularz recenzji (`resources/views/properties/_review-form-multi.blade.php`):

```blade
<form method="POST" action="{{ route('reviews.store', $property) }}" id="review-form">
    @csrf
    ...
</form>
```

`@csrf` dokłada do formularza ukryte pole z tokenem zapisanym w sesji. Wbudowany middleware `VerifyCsrfToken` przy każdym żądaniu POST/PATCH/DELETE porównuje token z formularza z tokenem sesji. Jeśli się nie zgadza (żądanie z obcej strony) — Laravel zwraca błąd **419** i dane nie zostają zapisane. Token zna tylko prawdziwa strona aplikacji, więc obca witryna nie podszyje się pod zalogowanego użytkownika.

---

## 3. Reakcja na sytuacje nietypowe — co widzi użytkownik i co dzieje się w kodzie

**Nieistniejące / zablokowane ogłoszenie** → strona 404. Kod (`PropertyController::show`):

```php
public function show(Property $property): View
{
    abort_if($property->isBlocked(), 404);   // zablokowane „udaje" nieistniejące
    // ...
}
```

Gdy podany numer ogłoszenia nie istnieje, mechanizm route model binding sam zwraca 404.

**Brak uprawnień do panelu admina** → strona 403. Kod (`AdminMiddleware.php`):

```php
if (! $request->user() || $request->user()->role !== UserRole::Admin) {
    abort(403, 'Brak dostępu do panelu administracyjnego.');
}
```

**Konto zablokowane** → wylogowanie i komunikat. Kod (`EnsureUserNotBlocked.php`):

```php
if (Auth::check() && Auth::user()->isBlocked()) {
    Auth::logout();
    $request->session()->invalidate();
    return redirect()->route('login')
        ->withErrors(['email' => 'Twoje konto zostało zablokowane. Skontaktuj się z administratorem.']);
}
```

**Pusta lista** → komunikat zamiast pustej strony. Kod (`resources/views/admin/reviews/index.blade.php`):

```blade
@forelse($reviews as $review)
    {{-- wiersz recenzji --}}
@empty
    {{-- komunikat: brak wyników --}}
@endforelse
```

Użytkownik widzi więc albo stronę błędu (403/404/419), albo komunikat na ekranie.

---

## 4. Jak aplikacja rozpoznaje intencję użytkownika (adres, rodzaj żądania, ścieżka, kod)

Po **rodzaju żądania HTTP + adresie**. Trasy są w `routes/web.php`:

```php
Route::prefix('ogloszenia')->name('properties.')->group(function () {
    Route::get('/',           [PropertyController::class, 'index']);   // GET  /ogloszenia        → lista
    Route::post('/',          [PropertyController::class, 'store']);   // POST /ogloszenia        → utwórz
    Route::get('/{property}', [PropertyController::class, 'show']);    // GET  /ogloszenia/{id}   → pokaż

    Route::middleware(['auth', 'verified', 'not.blocked'])->group(function () {
        Route::get('/{property}/edytuj', [PropertyController::class, 'edit']);    // formularz edycji
        Route::patch('/{property}',      [PropertyController::class, 'update']);  // PATCH  → aktualizuj
        Route::delete('/{property}',     [PropertyController::class, 'destroy']); // DELETE → usuń
    });
});
```

Ten sam adres `/ogloszenia/{id}` znaczy „pokaż" (GET), „aktualizuj" (PATCH) lub „usuń" (DELETE) — zależnie od metody. W formularzu HTML PATCH/DELETE realizuje się ukrytym polem `@method('PATCH')` / `@method('DELETE')`. Każda kombinacja wskazuje konkretną metodę kontrolera, np. `store` przy POST.

---

## 5. Miejsce do poprawy technicznej

**Problem:** zatwierdzanie recenzji (`verify`) jest zaimplementowane dwukrotnie. Raz w `ReviewController` (trasa dostępna dla zwykłego użytkownika, chroniona dopiero w środku przez `authorize`):

```php
// app/Http/Controllers/ReviewController.php
public function verify(Review $review): RedirectResponse
{
    $this->authorize('verify', $review);
    $review->update(['is_verified' => true, 'verified_by' => auth()->id(), 'verified_at' => now()]);
    return back()->with('success', 'Recenzja została zweryfikowana.');
}
```

a drugi raz w panelu admina, robiąc niemal to samo:

```php
// app/Http/Controllers/Admin/ReviewModerationController.php
public function verify(Review $review): RedirectResponse
{
    $review->update(['is_verified' => true, 'verified_by' => auth()->id(), 'verified_at' => now()]);
    return back()->with('success', 'Recenzja została zweryfikowana.');
}
```

**Wpływ:** zdublowany kod łatwo się rozjeżdża przy zmianach, a czynność czysto administracyjna jest częściowo dostępna spoza panelu admina — rozmywa to granicę uprawnień.

**Poprawa:** usunąć metodę i trasę `verify` z poziomu użytkownika, zostawić jedną — `admin.reviews.verify` (chronioną przez `auth + admin`), i na nią kierować przycisk w widoku.

---

## 6. Walidacja danych wpisanych przez użytkownika

Walidacja siedzi w klasach Form Request (`app/Http/Requests/`) i uruchamia się automatycznie przed kodem kontrolera. Przykład — `StoreMultiCriteriaReviewRequest`:

```php
public function rules(): array
{
    return [
        'title'             => ['required', 'string', 'min:5', 'max:200'],
        'content'           => ['required', 'string', 'min:150', 'max:5000'],
        'rating_overall'    => ['required', 'integer', 'between:1,5'],
        'rating_landlord'   => ['required', 'integer', 'between:1,5'],
        'tenancy_end'       => ['nullable', 'date', 'before_or_equal:today', 'after_or_equal:tenancy_start'],
        // ...
    ];
}

public function messages(): array
{
    return [
        'content.min'            => 'Treść recenzji musi mieć co najmniej 150 znaków.',
        'rating_overall.between' => 'Ocena musi być w skali 1–5.',
    ];
}
```

Sprawdzane są m.in.: obecność i długość tytułu/treści (treść min. 150 znaków), zakres ocen 1–5, logika dat najmu. W ogłoszeniu dodatkowo kod pocztowy (`'postal_code' => ['required', 'regex:/^\d{2}-\d{3}$/']`) i pliki (`'image', 'mimes:jpeg,jpg,png,webp', 'max:5120'`).

Wykrywalne błędy: brak pola, za krótka treść, ocena spoza zakresu, zły kod pocztowy, zły plik. Gdy dane są błędne, użytkownik wraca do formularza z zachowanymi wartościami (`old(...)`), a błędy pokazują się zbiorczo i przy polach:

```blade
@if($errors->any())
    <ul>@foreach($errors->all() as $error)<li>{{ $error }}</li>@endforeach</ul>
@endif

<input name="title" value="{{ old('title') }}" class="{{ $errors->has('title') ? 'error' : '' }}">
@error('title') <span class="form-error">{{ $message }}</span> @enderror
```

---

## 7. Dobrze zorganizowany widok

Przykład — komponent karty ogłoszenia (`resources/views/components/property-card.blade.php`):

```blade
<div class="property-card">
    <a href="{{ route('properties.show', $property) }}">
        @if($property->coverImage)
            <img src="{{ $property->coverImage->url }}" alt="{{ $property->title }}" loading="lazy">
        @else
            <div class="property-card-img-placeholder"><i class="fa-solid fa-house"></i></div>
        @endif
    </a>
    <div class="property-card-body">
        <div class="property-card-price">{{ $property->formatted_price }}<span>/mies.</span></div>
        <div class="property-card-title">{{ Str::limit($property->title, 55) }}</div>
        <div class="property-card-location">{{ $property->city }}</div>
    </div>
</div>
```

Jest czytelny, bo każda sekcja (zdjęcie, cena, tytuł, lokalizacja) jest osobna i podpisana. **Nie powtarza kodu** — zamiast kopiować kartę na liście, mapie i stronie głównej, wstawia się ją jednym wywołaniem `<x-property-card :property="$property"/>`. Zmiana wyglądu = poprawka w jednym pliku, widoczna wszędzie. Korzysta z gotowych akcesorów (`$property->formatted_price`, `$property->coverImage`), więc formatowanie jest w jednym miejscu. Rozbudowa to dosłownie jedna linijka.

---

## 8. Rozróżnianie uprawnień + przykład

Role są w enumie `UserRole` (`Guest`, `User`, `Admin`) w kolumnie `users.role`. Kontrola działa na trzech poziomach.

Middleware na trasie (cały panel admina):

```php
// routes/web.php
Route::middleware(['auth', 'admin'])->prefix('admin')->name('admin.')->group(function () { ... });
```

Policy decydująca o pojedynczym obiekcie (`app/Policies/ReviewPolicy.php`):

```php
public function before(User $user): ?bool
{
    return $user->isAdmin() ? true : null;     // admin może wszystko
}

public function delete(User $user, Review $review): bool
{
    return $review->user_id === $user->id && ! $user->isBlocked();  // autor może usunąć tylko swoją
}
```

W kontrolerze sprawdzenie wywoływane jest tak:

```php
public function destroy(Review $review): RedirectResponse
{
    $this->authorize('delete', $review);   // tu Policy decyduje
    $review->delete();
    return back()->with('success', 'Recenzja została usunięta.');
}
```

**Przykład:** użytkownik A może usunąć własną recenzję (warunek `$review->user_id === $user->id` spełniony). Użytkownik B przy próbie usunięcia recenzji A dostaje **403**. Administrator może usunąć każdą recenzję — bo `before()` zwraca dla niego `true`. Ta sama akcja jest więc dozwolona dla autora i admina, a zabroniona dla obcego użytkownika.

---

## 9. Pełna droga jednej akcji: dodanie recenzji (warstwy + 6 fragmentów kodu)

Łańcuch: **formularz → trasa/middleware → walidacja + autoryzacja → kontroler → zapis do bazy + zdarzenie modelu → przekierowanie do widoku.**

**1) Formularz — przechwycenie danych** (`resources/views/properties/_review-form-multi.blade.php`):

```blade
<form method="POST" action="{{ route('reviews.store', $property) }}" id="review-form">
    @csrf
    <input type="hidden" name="rating_overall" value="{{ old('rating_overall', 0) }}">
    <textarea name="content">{{ old('content') }}</textarea>
    <button type="submit">Opublikuj recenzję</button>
</form>
```

**2) Trasa + bramki dostępu** (`routes/web.php`) — wymaga zalogowania, weryfikacji e-mail i braku blokady:

```php
Route::middleware(['auth', 'verified', 'not.blocked'])->group(function () {
    Route::prefix('ogloszenia/{property}/recenzje')->name('reviews.')->group(function () {
        Route::post('/', [ReviewController::class, 'store'])->name('store');
    });
});
```

**3) Walidacja i autoryzacja** (`app/Http/Requests/StoreMultiCriteriaReviewRequest.php`):

```php
public function authorize(): bool
{
    $user = $this->user(); $property = $this->route('property');
    if (!$user || !$user->canWriteReviews() || $user->isBlocked()) return false;
    if ($property->isOwnedBy($user)) return false;                                  // nie własne ogłoszenie
    if ($property->reviews()->where('user_id', $user->id)->exists()) return false;  // tylko jedna recenzja
    return true;
}
```

**4) Główna logika** (`app/Http/Controllers/ReviewController.php`):

```php
public function store(StoreMultiCriteriaReviewRequest $request, Property $property): RedirectResponse
{
    $validated = $request->validated();
    $validated['rating_cleanliness'] = (int) round(
        ($validated['rating_overall'] + $validated['rating_landlord'] + $validated['rating_noise_level']
        + $validated['rating_location'] + $validated['rating_value_for_money']) / 5
    );
    $validated['user_id'] = $request->user()->id;
    $property->reviews()->create($validated);          // ZAPIS do bazy przez relację
    return redirect()->route('properties.show', $property)
        ->with('success', 'Twoja recenzja została dodana i oczekuje na moderację.');
}
```

**5) Zapis do bazy + automatyczny efekt uboczny** (`app/Models/Review.php`) — po zapisie przelicza się średnia ocena ogłoszenia:

```php
protected static function booted(): void
{
    static::saved(fn (Review $review) => $review->property?->recalculateRating());
    static::deleted(fn (Review $review) => $review->property?->recalculateRating());
}
```

oraz w `app/Models/Property.php`:

```php
public function recalculateRating(): void
{
    $avg = $this->visibleReviews()->avg('rating_overall');
    $this->updateQuietly(['avg_rating' => $avg ? round($avg, 2) : null]);
}
```

**6) Zwrócenie widoku** — przekierowanie na `properties.show`; `PropertyController::show` ładuje ogłoszenie z recenzjami, użytkownik widzi swoją recenzję i komunikat o oczekiwaniu na moderację.

---

## 10. Powiązanie danych: ogłoszenie ↔ recenzje (relacja jeden-do-wielu)

**W bazie** (`create_reviews_table.php`) — klucz obcy plus kaskada:

```php
$table->foreignId('property_id')->constrained()->cascadeOnDelete();
```

**W kodzie** — obie strony relacji:

```php
// app/Models/Property.php
public function reviews(): HasMany { return $this->hasMany(Review::class); }
public function visibleReviews(): HasMany { return $this->hasMany(Review::class)->where('is_hidden', false); }

// app/Models/Review.php
public function property(): BelongsTo { return $this->belongsTo(Property::class); }
```

**W widoku** — na liście pobieramy liczbę i średnią recenzji jednym zapytaniem (`PropertyController::index`):

```php
Property::query()->withAvg('visibleReviews as avg_overall', 'rating_overall')
                 ->withCount('visibleReviews as reviews_count');
```

a na stronie ogłoszenia wyświetlamy listę:

```blade
@foreach($property->visibleReviews as $review)
    <x-review-card :review="$review"/>
@endforeach
```

To samo powiązanie widać więc w bazie, w kodzie i na ekranie.

---

## 11. Funkcjonalność niewidoczna w interfejsie

**Automatyczne przeliczanie średniej oceny** (kod w punkcie 9: `Review::booted()` + `Property::recalculateRating()`). Nigdzie nie ma przycisku „przelicz" — dzieje się to samo po każdej zmianie recenzji. Bez tego oceny przy ogłoszeniach byłyby nieaktualne. `updateQuietly()` zapobiega zapętleniu zdarzeń.

**Anonimowe pseudonimy** (`app/Services/AnonymousPseudonymService.php`):

```php
public function forUser(int $userId): string
{
    $hash = abs(crc32("anon_{$userId}"));
    $adjIndex    = $hash % count(self::ADJECTIVES);
    $animalIndex = intdiv($hash, count(self::ADJECTIVES)) % count(self::ANIMALS);
    return self::ADJECTIVES[$adjIndex] . ' ' . self::ANIMALS[$animalIndex];
}
```

Zamienia ID użytkownika na stały pseudonim („Spokojny Bóbr"). Ten sam autor zawsze ma ten sam pseudonim, ale jego tożsamość pozostaje ukryta — istotne przy recenzjach najmu.

Trzecia rzecz — `TrackPropertyView` zliczający odsłony ogłoszenia raz na sesję.

---

## 12. Dlaczego projekt zasługuje na dobrą ocenę

Projekt jest kompletny: realizuje pełen CRUD ogłoszeń i recenzji z czytelnym, RESTowym routingiem i polskimi adresami. Walidacja jest wydzielona do klas Form Request i zawiera reguły biznesowe (jedna recenzja na ogłoszenie, zakaz recenzowania własnego mieszkania). Uprawnienia działają na trzech warstwach — middleware, Policy z `before()` dla admina i `authorize()` w żądaniach — co realnie oddziela role. Baza ma przemyślany schemat z kluczami obcymi, indeksami, ograniczeniem `unique`, kaskadami i soft-deletes. Widać dbałość o jakość kodu: komponenty Blade bez powtórzeń, akcesory i scope'y w modelach, usługi wydzielające logikę oraz automatyczne przeliczanie ocen przez zdarzenia modelu. Aplikacja jest zabezpieczona przed CSRF i sensownie reaguje na błędy 403/404/419. Są też seedery i testy. Słabe punkty (zdublowany `verify`, licznik „pomocne" na sesji) są drobne i lokalne. Całość pokazuje rozumienie architektury Laravela, a nie tylko jej odtworzenie.

---

## 13. Co poprawić w pierwszej kolejności

Uporządkować zatwierdzanie recenzji do **jednej, administracyjnej ścieżki** (punkt 5): usunąć metodę `verify` i jej trasę z poziomu użytkownika, zostawiając tylko `admin.reviews.verify` chronioną przez `auth + admin`.

Dlaczego to najważniejsze: weryfikacja recenzji decyduje o wiarygodności serwisu (nadaje status „zweryfikowana"). Trzymanie jej częściowo poza panelem admina rozmywa granicę uprawnień i jest najbardziej wrażliwym punktem z perspektywy bezpieczeństwa. Skonsolidowanie usuwa duplikację, zmniejsza ryzyko błędów przy zmianach i czyni model uprawnień w pełni spójnym.

---

## Uwaga

Pytanie o porównanie z innym projektem z grupy pominięto — w archiwum jest tylko ten jeden projekt.
