# Analiza projektu RentOpinion — pytania i odpowiedzi

RentOpinion to strona internetowa, na której najemcy wystawiają opinie o wynajmowanych mieszkaniach i o wynajmujących. Można dodać ogłoszenie, przeglądać oferty na liście i na mapie, a także napisać recenzję. Projekt napisano w technologii Laravel (język PHP), dane trzyma w bazie SQLite, a wygląd stron robią szablony Blade.

Poniżej każde pytanie w pełnym brzmieniu i odpowiedź wytłumaczona możliwie prosto.

---

## 1. Jak przygotowano strukturę bazy danych? Wskaż przykładową tabelę, opisz jej najważniejsze pola i wyjaśnij, skąd biorą się przykładowe/początkowe dane.

Bazę danych zbudowano za pomocą tzw. migracji. Migracja to po prostu plik z przepisem: „utwórz taką tabelę, z takimi kolumnami". Dzięki temu każdy, kto pobierze projekt, jednym poleceniem odtworzy całą bazę od zera — nie trzeba jej klikać ręcznie. W projekcie są cztery główne tabele: użytkownicy (`users`), ogłoszenia (`properties`), zdjęcia ogłoszeń (`property_images`) i recenzje (`reviews`).

Weźmy tabelę **recenzji**. Przechowuje ona m.in.: do którego ogłoszenia należy recenzja, kto ją napisał, oceny w skali 1–5 (osobno za standard, kontakt z właścicielem, sąsiedztwo, okolicę, cenę), tytuł, treść, a także informacje dla administratora — czy recenzja jest zweryfikowana i czy została ukryta.

Trzy rzeczy warte podkreślenia, bo to one decydują o porządku w danych:

- jest reguła pilnująca, że **jedna osoba może dodać tylko jedną recenzję do danego ogłoszenia** (nie da się zaspamować jednej oferty),
- jeśli usunie się ogłoszenie, **jego recenzje znikają razem z nim** (nie zostają „sieroty" bez ogłoszenia),
- usunięcie recenzji to tak naprawdę tylko jej **schowanie** (zostaje w bazie z datą usunięcia), więc nic nie ginie bezpowrotnie.

Skąd biorą się dane na start? Z dwóch źródeł. Pierwsze to tzw. seedery — gotowe paczki danych wpisywane przy uruchomieniu projektu (np. konto administratora `admin@rentopinion.pl` i kilka przykładowych ogłoszeń z polskich miast). Drugie to fabryki, które generują dużo losowych, ale sensownych danych (np. losowe oceny i polskie teksty recenzji) — używa się ich głównie do testów.

---

## 2. Jak projekt zabezpiecza formularze przed niepożądanym lub przypadkowym wysłaniem danych z zewnątrz? Wskaż konkretny formularz i opisz mechanizm.

Chodzi o ochronę przed sytuacją, w której obca strona internetowa próbuje „w tle" wysłać formularz w imieniu zalogowanego użytkownika (atak typu CSRF). Zabezpieczeniem jest tzw. token CSRF — można go porównać do tajnego znaczka, który strona wkłada do każdego swojego formularza.

W praktyce wygląda to tak: w formularzu dodawania recenzji jest jedna linijka `@csrf`, która niewidocznie dokłada ten znaczek do wysyłanych danych. Gdy formularz trafia na serwer, system sprawdza, czy znaczek się zgadza z tym zapisanym w sesji użytkownika. Jeśli tak — żądanie jest przyjmowane. Jeśli nie (bo np. wysłała je obca strona, która znaczka nie zna) — serwer odrzuca je z błędem i nic się nie zapisuje.

Najważniejsze, że ten znaczek zna tylko prawdziwa strona projektu. Obca witryna nie ma jak go zdobyć, więc nie podszyje się pod użytkownika.

---

## 3. Jak aplikacja reaguje na nietypową sytuację? Co zobaczy użytkownik i co dzieje się w kodzie?

Aplikacja przewiduje kilka takich sytuacji:

**Ktoś wchodzi na ogłoszenie, którego nie ma albo które zostało zablokowane.** Wtedy zamiast zawartości pokazuje się strona „nie znaleziono" (błąd 404). Co ważne, zablokowane ogłoszenie też udaje nieistniejące — żeby nie zdradzać, że w ogóle było.

**Ktoś próbuje zrobić coś, do czego nie ma prawa** (np. usunąć cudzą recenzję albo wejść do panelu administratora). Wtedy dostaje stronę „brak dostępu" (błąd 403).

**Konto zostało zablokowane przez administratora.** Przy najbliższej próbie zrobienia czegokolwiek użytkownik jest automatycznie wylogowywany i widzi komunikat: „Twoje konto zostało zablokowane. Skontaktuj się z administratorem."

**Lista jest pusta** (np. brak recenzji). Zamiast pustej, mylącej strony pojawia się zwykły komunikat „brak wyników".

W kodzie odpowiadają za to krótkie „bramki" — np. polecenie „jeśli ogłoszenie jest zablokowane, przerwij i pokaż 404". Użytkownik widzi natomiast albo czytelną stronę błędu, albo zielony/czerwony komunikat na górze ekranu.

---

## 4. Jak aplikacja rozpoznaje, co użytkownik chce zrobić (pobrać dane, edytować, usunąć, utworzyć)? Odnieś się do adresu strony, rodzaju żądania, formularza, ścieżki i fragmentu kodu.

Aplikacja rozpoznaje to po dwóch rzeczach naraz: po **adresie strony** i po **rodzaju żądania**. To trochę jak zamawianie w restauracji — liczy się i to, co mówisz, i jak to mówisz.

Najlepiej widać to na ogłoszeniach:

| Co użytkownik chce zrobić | Rodzaj żądania | Adres |
|---|---|---|
| Zobaczyć listę | otwarcie strony (GET) | `/ogloszenia` |
| Zobaczyć jedno ogłoszenie | otwarcie strony (GET) | `/ogloszenia/{numer}` |
| Dodać nowe | wysłanie formularza (POST) | `/ogloszenia` |
| Edytować | aktualizacja (PATCH) | `/ogloszenia/{numer}` |
| Usunąć | usunięcie (DELETE) | `/ogloszenia/{numer}` |

Ten sam adres `/ogloszenia/{numer}` może więc znaczyć „pokaż", „zmień" albo „usuń" — zależnie od rodzaju żądania. W pliku z trasami (`routes/web.php`) każda taka kombinacja jest przypisana do konkretnej funkcji w kodzie, np.:

```php
Route::post('/', [PropertyController::class, 'store'])->name('store'); // dodaj nowe ogłoszenie
```

Ciekawostka: przeglądarka z formularza umie wysłać tylko „otwórz" i „wyślij", więc dla „edytuj" i „usuń" projekt dokłada ukryte pole, które mówi serwerowi, o jaką operację naprawdę chodzi.

---

## 5. Wskaż jedno miejsce, które powinno zostać poprawione technicznie. Opisz problem, jego wpływ i zaproponuj poprawę.

**Problem:** zatwierdzanie (weryfikowanie) recenzji jest zrobione w dwóch miejscach naraz — raz „od strony użytkownika", a raz w panelu administratora. To dwie prawie identyczne funkcje robiące to samo. Dodatkowo ta „użytkownikowa" droga jest dostępna dla każdego zalogowanego, a dopiero w środku sprawdza się, czy to administrator.

**Dlaczego to przeszkadza:** gdy kiedyś trzeba będzie coś w tym poprawić, łatwo zmienić jedną wersję, a o drugiej zapomnieć — i zrobi się bałagan. To też niepotrzebne ryzyko: czynność typowo administracyjną lepiej trzymać wyłącznie po stronie administratora.

**Propozycja:** zostawić tylko jedną drogę — tę w panelu administratora, chronioną od samego wejścia — a tę drugą usunąć. Mniej kodu, jeden punkt odpowiedzialności, mniejsza szansa na błąd.

(Drobniejszy przykład: licznik „ta recenzja była pomocna" pilnuje pojedynczego kliknięcia tylko przez pamięć przeglądarki, więc wyczyszczenie ciasteczek pozwala kliknąć ponownie. Docelowo warto zapisywać to w bazie.)

---

## 6. Gdzie aplikacja sprawdza poprawność danych wpisanych przez użytkownika? Jakie dane są sprawdzane, jakie błędy mogą zostać wykryte i co zobaczy użytkownik?

Sprawdzanie danych jest wydzielone do osobnych plików (tzw. Form Requests), a nie wmieszane w resztę kodu. To dobre rozwiązanie, bo wszystkie reguły dla danego formularza są w jednym miejscu. Co ważne, sprawdzenie odbywa się **zanim** dane w ogóle dotrą do głównej logiki — jeśli coś jest nie tak, dalej nic się nie dzieje.

Na przykładzie recenzji sprawdzane jest m.in.:

- czy podano tytuł i treść (treść musi mieć co najmniej 150 znaków, żeby recenzja była konkretna),
- czy oceny mieszczą się w skali 1–5,
- czy daty najmu mają sens (koniec nie wcześniej niż początek i nie w przyszłości).

Przy dodawaniu ogłoszenia sprawdzany jest też m.in. format kodu pocztowego (XX-XXX) oraz to, czy wgrane zdjęcia naprawdę są obrazkami i nie są za duże.

Co widzi użytkownik, gdy się pomyli? Strona nie znika i nie traci wpisanych danych — wraca do formularza z zachowaniem tego, co już napisał, i pokazuje czerwone komunikaty po polsku, np. „Treść recenzji musi mieć co najmniej 150 znaków." Komunikaty są i zbiorczo na górze, i przy konkretnym polu, którego dotyczą.

---

## 7. Wskaż dobrze zorganizowany widok. Dlaczego jest czytelny, czy unika powtórzeń i czy łatwo byłoby go zmienić lub rozbudować?

Dobrym przykładem jest **kafelek ogłoszenia** (`property-card`) — czyli ta prostokątna karta ze zdjęciem, ceną, tytułem i miastem, którą widać na liście ofert.

Jest czytelny, bo każdy element (zdjęcie, cena, tytuł, lokalizacja, parametry) jest wyraźnie oddzielony i podpisany komentarzem — od razu wiadomo, co jest czym.

Co najważniejsze, **nie powtarza się**. Zamiast kopiować kod tej karty na liście ofert, na mapie i na stronie głównej, zrobiono ją raz jako gotowy „klocek" i wstawia się ją jednym wywołaniem: `<x-property-card .../>`. Dzięki temu, jeśli ktoś będzie chciał zmienić wygląd karty (np. dodać oznaczenie „Nowość"), poprawia tylko jeden plik, a zmiana pojawia się od razu wszędzie. To czyni go bardzo łatwym do rozbudowy — dorzucenie nowej informacji to dosłownie jedna linijka. Podobnie zrobiono kartę recenzji i gwiazdki ocen.

---

## 8. Jak aplikacja rozróżnia użytkowników o różnych uprawnieniach? Podaj przykład, gdy jedna osoba może coś zrobić, a inna nie. Jak projekt to kontroluje?

Każdy użytkownik ma przypisaną **rolę**: gość, zwykły użytkownik albo administrator. Po roli aplikacja wie, na co dana osoba może sobie pozwolić.

Kontrola działa na kilku poziomach, niczym kolejne bramki: najpierw przy wejściu na adres (np. cały panel administratora jest zamknięty dla osób bez roli administratora), a potem przy konkretnych czynnościach (np. „czy ta recenzja na pewno należy do ciebie?").

**Konkretny przykład:** użytkownik A napisał recenzję. Tylko on widzi przy niej przycisk „Usuń" i tylko on może ją skasować — bo system sprawdza, czy autor recenzji to ta sama osoba, która klika. Użytkownik B, gdyby spróbował usunąć recenzję A, dostanie „brak dostępu" (403). Wyjątkiem jest administrator — on z założenia może usunąć każdą recenzję, bo do moderacji treści jest powołany. Czyli ta sama czynność („usuń recenzję") jest dozwolona dla autora i dla administratora, a zabroniona dla kogoś obcego. Na tym właśnie polega rozróżnianie uprawnień.

---

## 9. Wybierz jedną akcję i opisz krok po kroku drogę danych — od kliknięcia w interfejsie aż po zapis i zwrócenie wyniku. Zidentyfikuj warstwy i dołącz co najmniej 5 fragmentów kodu z podpisami.

Wybieram **dodanie recenzji**. Prześledźmy, co się dzieje od momentu, gdy użytkownik klika „Opublikuj recenzję", aż do chwili, gdy widzi efekt.

Najprościej mówiąc, dane przechodzą przez taki łańcuch: **formularz → sprawdzenie adresu i uprawnień → sprawdzenie poprawności danych → główna logika → zapis do bazy → odświeżenie strony z wynikiem.**

**Krok 1 — formularz (skąd biorą się dane).** Użytkownik wypełnia pola i klika przycisk. To plik widoku z formularzem:

```blade
<form method="POST" action="{{ route('reviews.store', $property) }}" id="review-form">
    @csrf
    <input type="hidden" name="rating_overall" value="{{ old('rating_overall', 0) }}">
    <textarea name="content" ...>{{ old('content') }}</textarea>
    <button type="submit">Opublikuj recenzję</button>
</form>
```

**Krok 2 — trasa i bramki dostępu (czy w ogóle wolno tu wejść).** Adres jest w grupie chronionej: trzeba być zalogowanym, mieć potwierdzony e-mail i nie być zablokowanym.

```php
Route::prefix('ogloszenia/{property}/recenzje')->name('reviews.')->group(function () {
    Route::post('/', [ReviewController::class, 'store'])->name('store');
});
```

**Krok 3 — sprawdzenie uprawnień i danych (czy ta osoba może i czy dane są poprawne).** Tu pilnujemy m.in., że nie recenzujesz własnego ogłoszenia i że nie dodajesz drugiej recenzji do tej samej oferty:

```php
public function authorize(): bool {
    $user = $this->user(); $property = $this->route('property');
    if (!$user || !$user->canWriteReviews() || $user->isBlocked()) return false;
    if ($property->isOwnedBy($user)) return false;                          // nie swoje ogłoszenie
    if ($property->reviews()->where('user_id',$user->id)->exists()) return false; // tylko jedna recenzja
    return true;
}
```

**Krok 4 — główna logika (co system robi z danymi).** Dolicza brakującą ocenę, podpina autora i tworzy recenzję:

```php
public function store(StoreMultiCriteriaReviewRequest $request, Property $property): RedirectResponse {
    $validated = $request->validated();
    $validated['user_id'] = $request->user()->id;
    $property->reviews()->create($validated);   // tu powstaje nowa recenzja
    return redirect()->route('properties.show', $property)
        ->with('success', 'Twoja recenzja została dodana i oczekuje na moderację.');
}
```

**Krok 5 — zapis do bazy i automatyczny efekt uboczny.** Po zapisaniu recenzji system sam przelicza średnią ocenę ogłoszenia — użytkownik nic w tej sprawie nie klika:

```php
protected static function booted(): void {
    static::saved(fn (Review $review) => $review->property?->recalculateRating());
    static::deleted(fn (Review $review) => $review->property?->recalculateRating());
}
```

**Krok 6 — zwrócenie wyniku.** Na koniec użytkownik zostaje przeniesiony z powrotem na stronę ogłoszenia, widzi tam swoją recenzję i zielony komunikat, że czeka ona na zatwierdzenie przez administratora.

---

## 10. Opisz jeden przykład powiązania danych. Wybierz dwa typy danych ze sobą powiązane i wyjaśnij, jak to widać w bazie, kodzie i widoku.

Najprostszy przykład to **ogłoszenie i jego recenzje**. Jedno ogłoszenie może mieć wiele recenzji — czyli „jeden do wielu", tak jak jeden produkt w sklepie może mieć wiele opinii.

**W bazie danych:** w tabeli recenzji jest kolumna wskazująca, do którego ogłoszenia należy dana recenzja. Dodatkowo ustawiono, że skasowanie ogłoszenia usuwa też jego recenzje.

**W kodzie:** ogłoszenie „wie", że ma swoje recenzje, a recenzja „wie", do którego ogłoszenia należy. Dzięki temu można łatwo pobrać jedno z drugiego:

```php
public function reviews(): HasMany { return $this->hasMany(Review::class); }   // ogłoszenie → jego recenzje
public function property(): BelongsTo { return $this->belongsTo(Property::class); } // recenzja → jej ogłoszenie
```

**W widoku:** na liście ofert przy każdym ogłoszeniu widać liczbę i średnią ocen, a na stronie konkretnego ogłoszenia wyświetla się lista wszystkich jego recenzji. To samo powiązanie widać więc na wszystkich trzech poziomach: w bazie, w kodzie i na ekranie.

(Podobnie powiązani są: użytkownik i jego ogłoszenia, użytkownik i jego recenzje, ogłoszenie i jego zdjęcia.)

---

## 11. Wskaż jedną funkcjonalność, której nie widać od razu w interfejsie, ale która jest ważna. Gdzie się znajduje i dlaczego jest potrzebna?

Dobrym przykładem jest **automatyczne przeliczanie średniej oceny ogłoszenia**. Nigdzie nie ma przycisku „przelicz ocenę" — dzieje się to samo, w tle, za każdym razem gdy ktoś doda, zmieni albo usunie recenzję. Gdyby tego nie było, oceny pokazywane przy ogłoszeniach szybko przestałyby się zgadzać z rzeczywistością.

Druga taka rzecz to **anonimowe pseudonimy**. Gdy ktoś wystawia recenzję anonimowo, system zamienia go na stały pseudonim w stylu „Spokojny Bóbr". Co ważne, ten sam autor zawsze dostaje ten sam pseudonim — czytelnik widzi więc spójną „osobę", ale nie pozna jej nazwiska. To istotne akurat przy opiniach o najmie, bo ludzie chętniej napiszą szczerze (także krytycznie) o właścicielu mieszkania, jeśli nie grozi to ujawnieniem tożsamości.

Jest też cichy licznik odsłon ogłoszeń, który zlicza unikalne wejścia raz na sesję.

---

## 12. Napisz 6–10 zdań, dlaczego projekt zasługuje na wskazaną ocenę. Odwołaj się do konkretnych przykładów.

Projekt jest kompletny i przemyślany jak na pracę zaliczeniową. Pozwala wykonać pełen zestaw działań — dodawanie, przeglądanie, edycję i usuwanie ogłoszeń oraz recenzji — a adresy stron są czytelne i po polsku (np. `/ogloszenia/dodaj`). Sprawdzanie poprawności danych jest porządnie wydzielone i zawiera sensowne reguły, takie jak zakaz recenzowania własnego mieszkania czy limit jednej recenzji na ofertę. Uprawnienia zrobiono solidnie i na kilku poziomach, dzięki czemu role gościa, użytkownika i administratora są naprawdę od siebie oddzielone. Baza danych ma sensowną strukturę z powiązaniami między tabelami i zabezpieczeniami przed bałaganem w danych. Widać też dbałość o jakość: powtarzające się elementy wyglądu zrobiono jako gotowe „klocki", a część logiki (np. pseudonimy, przeliczanie ocen) działa automatycznie w tle. Aplikacja jest też zabezpieczona przed typowymi atakami na formularze i sensownie reaguje na błędy oraz brak uprawnień. Do tego dochodzą testy i gotowe dane startowe. Słabsze punkty, jak zdublowana funkcja zatwierdzania recenzji, są drobne i łatwe do naprawienia. Całość pokazuje, że autor rozumie, jak taka aplikacja działa od środka, a nie tylko skopiował gotowy schemat.

---

## 13. Co należy poprawić w pierwszej kolejności, aby projekt zasługiwał na wyższą ocenę? Podaj jedną konkretną zmianę i wyjaśnij, dlaczego jest najważniejsza.

Najpierw uporządkowałbym **zatwierdzanie recenzji do jednej drogi — wyłącznie po stronie administratora** (opisane w punkcie 5): usunąć tę zdublowaną, „użytkownikową" wersję i zostawić jedną, zamkniętą dla zwykłych użytkowników.

Dlaczego akurat to? Bo zatwierdzanie recenzji to czynność, która decyduje o wiarygodności całego serwisu — to ona nadaje opiniom status „sprawdzonej". Trzymanie jej częściowo poza panelem administratora rozmywa granicę między tym, co wolno użytkownikowi, a co administratorowi, i jest najbardziej wrażliwym miejscem z punktu widzenia bezpieczeństwa. Poprawienie tego usuwa powielony kod, zmniejsza ryzyko pomyłek przy późniejszych zmianach i sprawia, że zasady dostępu stają się w pełni spójne — a to tutaj najważniejsze.

---

## Uwaga końcowa

Pytanie o porównanie z innym projektem z grupy (mocniejszy / słabszy) pominięto, ponieważ w dostarczonym archiwum znajduje się tylko ten jeden projekt — nie ma więc z czym go rzetelnie zestawić.
