# Kronikarz — specyfikacja produktu

Status: **planowanie** (analiza i zbieranie danych zakończone, implementacja nie rozpoczęta).
Data sporządzenia: 2026-09-16.

Kronikarz to plugin do klienta Dargoth (arkadia-web-client-extension), który prowadzi
audytowalny dziennik wypraw postaci: zdarzenia, finanse, paczki, zlecenia, zabici,
postępy, cechy, śmierci, podróże i granice sesji. Docelowy kształt to produkt kompletny:
pełne wykresy, pełne wyszukiwanie, pełny eksport i import, dane wszystkich postaci.
Etapów pośrednich (wariantów okrojonych) nie przewiduje się.

Platforma docelowa: wyłącznie klient Dargoth. Porty (Mudlet, klient WWW) są możliwe
dzięki architekturze, ale poza zakresem tego dokumentu.

---

## 1. Model pojęciowy

- **Zdarzenie** — atomowy wpis kroniki. Każde zdarzenie ma: typ, timestamp (czas
  zdarzenia, nie dotarcia), postać, źródło (live/backfill), surową linię gry (o ile
  pochodzi z tekstu), flagę confidence oraz metadane zależne od typu (kwota, lokacja,
  adresat itd.). Żadne zdarzenie nie jest połykane ani odrzucane cicho.
- **Sesja** — od zalogowania do wylogowania (`client.connect` → `client.disconnect`
  w API pluginów). Bez sztucznej spinki. Wyszukiwanie i podsumowania po czasie są
  możliwe niezależnie od sesji, bo każde zdarzenie ma timestamp.
- **Księga** — trwała baza zdarzeń w IndexedDB (baza globalna, wpisy z polem
  `character`), wspólna dla wszystkich postaci; widoki i podsumowania filtrują per
  postać lub zbiorczo.
- **Audytowalność** — każdą kwotę i klasyfikację można rozwinąć do surowej linii
  gry i źródła decyzji. Zdarzenia wątpliwe trafiają na listę do przeglądu zamiast
  być wykluczane.

---

## 2. Katalog zdarzeń

Legenda kolumny „weryfikacja": **kod** = pattern potwierdzony w kodzie klienta,
skryptów tjurczyka, Towarzysza lub python-toolkit; **korpus** = pattern do potwierdzenia
na rzeczywistych logach gracza; **GMCP** = kanał czysto GMCP (brak backfillu z logów).

### 2.1 Paczki pocztowe

Maszyna stanów przeniesiona z modułu `analizator` repo arkadia-python-toolkit
(zweryfikowana na logach), z trzema łatkami ujawnionymi testami adwersarialnymi
(patrz §3).

| Zdarzenie | Detekcja | Weryfikacja |
|---|---|---|
| Odbiór paczki | `... przekazuje ci jakas paczke.` | kod (klient: PackageHelper, tjurczyk: assistant) |
| Lista ofert (cel, nagroda, limit czasu) | `packageLineRegex` (miasto + zł/sr/mdz + czas), etykieta `Wypisano na niej duzymi literami: <MIASTO>` | kod (klient: PackageHelper) |
| Dostawa | `^Oddajesz pocztowa paczke` + wypłata (patrz niżej) | kod |
| Zwrot | `^Zwracasz pocztowa paczke` (brak wypłaty) | kod (PackageHelper + tjurczyk) |
| Spóźnienie | `Dostarczyles przesylke po terminie` (flaga na otwartej paczce) | kod (toolkit) |
| Wypłata | `wyplaca ci <waluta>` (waluta wymagana) LUB `otrzymujesz <waluta>` w oknie ≤ 2 linie od dostawy, bez frazy ` od <kogoś>` | kod + łatki własne |
| Odmowa odbioru | `Ty juz dla nas dostatecznie ciezko zapracowales`, `Nie ufam ci na tyle, aby powierzyc ci dostarczenie tej przesylki`, `Cos ci sie chyba pomylilo, nie ma takiej oferty`, `nie widzisz tu nikogo, od kogo mozna by wziac zlecenie` | kod (tjurczyk) |
| Nieoddana | paczka otwarta bez domknięcia; persystowana między sesjami, widoczna w raporcie jako „w toku" | projekt |

Statusy paczki: **dostarczona, spóźniona, zwrócona, nieoddana** — wszystkie obsługiwane.
Ekwiwalent `_merge_cross_session` z toolkitu: paczka w toku przetrwa relogin (storage).

Kanały premium (API pluginów klienta):
- event `packageStatus` = `{recipient, seconds, location}` — adresat, odliczanie do
  terminu i ID pokoju celu, rozgłaszane co sekundę przez klient;
- event `npc` + baza `npcStore` ({name, loc}: zdalna `arkadia-mapa/data/npc.json`
  TTL 24 h + wpisy lokalne douczane przy dostawach do nieznanych adresatów);
- eventy `leadTo` / `mapPath` — zaplanowana trasa do adresata;
- IndexedDB `ArkadiaDeliveryStats` (historia klienta: timestamp, late, zł/sr/mdz per
  postać) — dodatkowe źródło backfillu, bez surowych linii.

Raport „plan vs wykonanie": z listy ofert znana jest obiecana nagroda i limit czasu,
więc kronika pokazuje rozbieżności (obiecane vs wypłacone, limit vs rzeczywisty czas).

### 2.2 Pieniądze

Przeliczniki (dwa niezależne źródła: Towarzysz `coins.ts` i toolkit `denominacja`):
**1 mth = 100 zł = 24000 mdz; 1 zł = 20 sr = 240 mdz; 1 sr = 12 mdz.**

| Zdarzenie | Detekcja | Weryfikacja |
|---|---|---|
| Łup / kasa otrzymana | `^Bierzesz ...`, `^Dostajesz (.+)\.$`, `wyplaca ci (.+) monet` | kod (Towarzysz LOOT_PATTERNS) |
| Wydatek | `^Kupujesz `, `^Placisz ` (obejmuje przejazdy), `zgarnia ... monet`, `odbiera od ciebie ... monet ... w zamian za zakupion` | kod (Towarzysz SPEND_PATTERNS) |
| Sprzedaż | `^Sprzedajesz ` + zapłata osobną linią przychodu | kod (Towarzysz SELL_PATTERNS) |
| Liczby słowne | mapowanie liczebników polskich (Towarzysz `polishNumbers`, klient `contracts.ts` POLISH_NUMBERS) | kod |
| Wartość ekwipunku | `Wydaje ci sie, ze (jest/sa) wart... mied` i warianty | kod (klient: priceEvaluation) |

**Twarda zasada:** linia bez jawnego nominału (`monet` + liczba/liczebnik) nie ma wpływu
na księgę. Frazy `daje ci / wrecza ci / przekazuje ci` bez waluty dotyczą rzeczy lub
paczek, nie pieniędzy.

**Wykluczone jako niemierzalne:** denominacja w kantorze (komenda `zdenominuj` nie
drukuje kwot; wykrycie wymagałoby porównania ekwipunku przed/po). Konsekwencja
księgowa znikoma — wymiana nie zmienia majątku poza prowizją 3–8%, która pozostaje
niewidzialnym mikrowydatkiem. Świadomie poza zakresem.

**Nie istnieje w grze:** kradzież/okradzenie — poza katalogiem.

### 2.3 Bank i depozyty

| Zdarzenie | Detekcja | Weryfikacja |
|---|---|---|
| Odczyt depozytu | `Twoj depozyt zawiera ...`, `Twoj depozyt jest pusty`, `Nie posiadasz wykupionego...` | kod (klient: deposits.ts) |
| Wpłata / wypłata | triggery kontekstowe na lokacji z bindem `depozyt` (lista referencyjna, §5) | kod + mapa |
| Stan konta per bank | premium: odczyt storage klienta klucz `deposits` (characterStorage) | kod |

**Zasada księgowa:** wpłata i wypłata to **transfer** (przesunięcie gotówka ↔ bank),
nigdy przychód ani wydatek. Bilans majątku pokazuje gotówkę i depozyty osobno i łącznie.

Backfill: echo `→ depozyt` + odczyt w logach daje historię stanów banków.

### 2.4 Zlecenia (kontrakty NPC)

| Zdarzenie | Detekcja | Weryfikacja |
|---|---|---|
| Zapytanie | `^Pytasz .+ o zlecenie\.$` (otwiera kontekst; klient zapisuje `locationId` pokoju) | kod (klient: contracts.ts) |
| Oferta | `.+? \S+ do [^:]+: Tak, mam pewne pilne zamowienie na ([^.]+)\. Potrzebuje (?:jeszcze )?([^.,]+?)(?:, przynajmniej ([^.]+) jakosci)?\.` | kod |
| Termin | `.+? \S+ do [^:]+: Na realizacje zamowienia mam ... (dni/dzien/godzin/godziny/godzine), pozniej zapewne bede potrzebowac czego innego\.` | kod |
| Brak zlecenia | `.+? \S+ do [^:]+: Nie, w tej chwili niczego mi nie trzeba\. Zajrzyj moze za jakis czas\.` (zamyka kontekst, czyści kontrakty lokacji) | kod |
| Realizacja | kontekst: pokój aktywnego zlecenia + kasa-przychód **bez dowodu sprzedaży** (patrz reguła §4) | projekt + korpus (dokładne linie oddania) |

Premium: odczyt storage klienta klucz `contracts` (aktywne zlecenia z `locationId`,
przedmiotem, liczbą, jakością i deadline). Fallback: własne śledzenie tymi samymi
patternami.

### 2.5 Zabici

| Zdarzenie | Detekcja | Weryfikacja |
|---|---|---|
| Zabójstwo własne | `^[ >]*Zabil(?<v>es|as) (?<name>...)\.$` | kod (klient: kill.ts) |
| Zabójstwo drużyny | `^[ >]*(?<player>...) zabil(?<v>a?) (?<name>...)\.$` | kod (kill.ts) |
| Premium live | eventy API `kill` {killer: ME/TEAM/OTHER} i `enemyKilled` {objNum, killer, hasBody} | kod (plugin-types) |
| Premium historia | IndexedDB `ArkadiaKillsDB` (indeks `character`) | kod |

### 2.6 Postępy

| Zdarzenie | Detekcja | Weryfikacja |
|---|---|---|
| Wbicie postępu (live) | GMCP `char.state.improve` (0–15; 16 stanów) | kod (klient improveCounter, tjurczyk gmcp_handler_improvement, Towarzysz) |
| Linia tekstowa | `Poczynil(?:es|as) (.*) postepy, od momentu kiedy .* gry\.$` + wariant `Nie poczynil(?:es|as) zadnych postepow...` | kod (Ralandil/arkatt, piwa87/arkadia-user-plugins) |
| Skala | 16 poziomów: minimalne … niebotyczne = 1:1 `IMPROVE_STATES` klienta | kod + wiki |

Linia tekstowa jest odpowiedzią na komendę `postepy` — w backfillu daje migawki
(echo `→ postepy` + odpowiedź), na żywo służy jako koroboracja. Licznik postępów
resetuje się przy wylogowaniu — naturalnie sesyjny.

### 2.7 Cechy

Klient przechwytuje komendę `cechy` i parsuje odczyt; Kronikarz używa tych samych
zweryfikowanych patternów (klient: lvlCalc.ts):

- `Jestes <opis> i <ile> ci brakuje, zebys mogla? wyzej ocenic sw(a|oj) <cecze>.` z opcjonalnym suffiksem modyfikatora `( +cos )`,
- `Twoja/Twoj <cecha> osiagnela/al nadludzki poziom.`,
- linia zamykająca `Obecnie do waznych cech zaliczasz...`,
- `Twoje cechy sa oslabione po ostatniej smierci.` (snapshot oznaczany jako osłabiony).

Odczyt z modyfikatorem (sprzęt/zioła) jest odrzucany — nie zapisuje się fałszywej
wartości. Detekcja odczytu: event `command` = `cechy` + własny parsing linii.
Premium: storage klienta klucz `cechy_history` (historia zmian i koszt w postępach).
Backfill: echo `→ cechy` + odczyt w logach.

### 2.8 Śmierć

| Zdarzenie | Detekcja | Weryfikacja |
|---|---|---|
| Śmierć własna | `^Umierasz\.$` (następna linia `Oddalasz sie.` to odejście duszy — ignorowana) | kod (Towarzysz DEATH_PATTERNS) |
| Osłabienie po śmierci | `Twoje cechy sa oslabione po ostatniej smierci\.` (+ wariant z liczbą postępów do odbudowy) | kod (klient: afterDeathProgress, lvlCalc) |
| Śmierć członka drużyny | możliwa wyłącznie live: GMCP `objects.data` flaga `living` przy `team: true` | kod (klient: TeamManager) — **odłożone** |

### 2.9 Poczta (listy)

| Zdarzenie | Detekcja | Weryfikacja |
|---|---|---|
| Nowy list | `^Masz nowa poczte od [A-Za-z]+\.$` | kod (Towarzysz MAIL_PATTERN) |

### 2.10 Apokalipsa i czas IG

Patterny przeniesione z toolkitu (moduły `apokalipsa`, `analizator_czasu`), działające
na echu komendy `→ system` / `→ czas` + oknie odpowiedzi:

- `Swiat odrodzil sie : <dzien> <miesiąc rzymski> <rok>, <hh:mm:ss>` — apokalipsa jako
  zdarzenie kroniki i twarda granica kontekstu sesji w backfillu;
- `Swiat istnieje : ...` — uptime; `<n>% swiata zostalo opanowane` — ciemność;
- `Nadchodzi Czas Apokalipsy ... poprosil ... przyspieszenie` — apokalipsa wymuszona;
- `Jest w przyblizeniu <godzina słownie> ...` — kotwice czasu IG; konwersja RL↔IG
  per sesja; kalendary Imperium/Ishtar zweryfikowane 1:1 z pluginami kalendarzowymi.

Zdarzenia kroniki mogą być prezentowane z czasem RL i IG.

### 2.11 Sesja, trasa, komendy

- Granice sesji: eventy API `client.connect` / `client.disconnect`.
- Trasa wyprawy: GMCP `room.info` przy każdym ruchu + event `enterLocation` {id, room,
  direction}.
- Komendy gracza: event `command` — statystyki aktywności, kontekst odczytów
  (`cechy`, `postepy`, `depozyt`, `zdenominuj`).

---

## 3. Zahartowanie maszyny paczek (testy adwersarialne)

Maszyna z toolkitu osiąga ~100% trafności na logach, ale testy na linach
adwersarialnych ujawniły trzy kolizje. Kronikarz łata wszystkie:

1. **Kolizja `wyplaca ci`:** gałąź toolkit nie wymaga waluty — obca wypłata tuż po
   dostawie zamyka paczkę z kwotą 0 i pochłania prawdziwą. Łata: `wyplaca ci` wymaga
   waluty w linii (paritet z gałęzią `otrzymujesz`).
2. **Kolizja `otrzymujesz od X`:** podział łupu/przelew od gracza w oknie ≤ 2 linii
   księgował się jako wypłata za paczkę. Lokalizacja tego nie naprawia (drużyna może
   stać na poczcie). Łata: gałąź `otrzymujesz` odrzuca linie z ` od <kogoś>` + flaga
   confidence.
3. **Kwoty słownie:** `wyplaca ci kilka miedzianych monet` parsowało się na 0. Łata:
   parser liczebników słownych.

Przypadek nieszkodliwy: questowa `jakas paczke` poza pocztą tworzy wpis wiszący —
bez wpływu na księgę, trafia do „nierozliczonych", audytowalny.

---

## 4. Zasady księgowe i klasyfikacji

1. Przychód/wydatek vs **transfer** (bank): transfer nie zmienia majątku.
2. Linia bez nominału nie rusza księgi.
3. Każda kwota w miedzi (normalizacja przez 24000/240/12).
4. **Klasyfikacja na lokacji zleceniodawcy** (sklep/tawerna — sprzedaż jest możliwa):
   kasa-przychód z linią `Sprzedajesz` w oknie = **sprzedaż**; kasa-przychód bez
   dowodu sprzedaży = **zapłata za zlecenie**. Klasyfikacja po dowodzie, nie po
   samej lokacji.
5. Każda klasyfikacja ma confidence i surową linię; wątpliwe → lista do przeglądu.

---

## 5. Gate'y lokalizacji

**Decyzja: gate miękki — lokalizacja nigdy nie wyklucza zdarzenia.** Uzasadnienia:
(a) maszyna paczek działa na logach bez żadnej lokalizacji, bo łańcuch fraz jest
samowalidujący; twardy gate dawałby fałszywe negatywy (nowe/zmienione poczty, luki
synchronizacji mapy); (b) w backfillu z logów nie ma GMCP room.info — twarde wykluczanie
unicestwiłoby backfill.

Lokalizacja służy jako: metadane (trasy, statystyki), flaga confidence
(znana/nieznana), czujnik odkrywania nowych punktów (zdarzenie poza listą trafia na
listę do weryfikacji, nie jest odrzucane).

Listy referencyjne (plik `docs/lokacje-poczta-banki.json`, mapa sync 430, v0.220.0):
- **Poczty:** 47 nazwanych + 12 nienazwanych z symbolem P = 59 punktów; wykluczenia
  paczkowe: Jaskare (poczta bez paczek), Faroe (tylko lokalne), Biały Most (brak).
  Reguła: ID z listy LUB nazwa `^poczt` minus wykluczenia. Dostawa paczki globalna
  (oddanie w dowolnym mieście).
- **Depozyty:** 15 klasycznych + Biały Kieł (Brugge) + Val'kare + Ard Skellig
  (pokój 10416, suplement — luka bindu na mapie, zgłoszenie upstream odłożone).
  Kantory bez depozytu (13 lokacji) zweryfikowane z wiki — klient słusznie ich nie
  traktuje jako banków.
- **Zleceniodawcy:** pokoje z `locationId` z kontraktów (dynamiczne, z gry).

---

## 6. Źródła danych

### 6.1 Live (API pluginów klienta)

Kanały zweryfikowane w `plugin-types/index.d.ts`:
- linie: `message`; komendy: `command`; GMCP: `gmcp` + `gmcp.<ścieżka>`
  (`room.info`, `char.state`, `char.options`, `objects.data`, `objects.nums`);
- sesja: `client.connect` / `client.disconnect`; mapa: `mapReady`, `enterLocation`,
  `mapMove`, `leadTo`, `mapPath`;
- zdarzenia domenowe: `kill`, `enemyKilled`, `packageStatus`, `npc`, `storage`,
  `teamChange`, `clock.*`.

Tryb **premium** (odczyt struktur klienta, ten sam origin): localStorage
`<postać>:deposits`, `<postać>:cechy_history`, `contracts`; IndexedDB
`ArkadiaKillsDB`, `ArkadiaDeliveryStats`, `npcStore`. Warstwa premium jest
defensywna: przy niezgodności formatu — cichy fallback na własne triggery.

### 6.2 Backfill (historia sprzed instalacji pluginu)

Trzy adaptery do wspólnego formatu `{tekst, timestamp_ms, typ?}`:
1. **IndexedDB `ArkadiaMessagesDB`** (podstawa) — object store per sesja logowa
   (`session_<ts>`); wpisy {text HTML, type?, timestamp}; timestamp liczbowy (czas
   zdarzenia); tekst otagowywany; wpis bywa wieloliniowy (split z balansowaniem
   tagów). Sesja logowa = załadowanie strony klienta ≠ sesja gry.
2. **JSON eksportu klienta** (`LogExportData` v1) — ten sam kształt wpisów.
3. **Pliki HTML** (opcjonalnie) — format toolkitu; DOMParser zamiast BeautifulSoup.

**Echo komend** (`→ czas`, `→ system`, `→ postepy`, `→ cechy`, `→ depozyt`) otwiera
okna atrybucji w backfillu: migawki postępów, historia cech, historia banków,
apokalipsy, kotwice czasu IG. Pole `type` wpisu (dokładny zbiór wartości do
zweryfikowania na korpusie) może rozróżniać komendy/linie walki strukturalnie.

**Dedup po treści** (hash linii + czas), nie po nazwie sesji — ta sama sesja może
wejść z IndexedDB i z JSON-a.

**Ograniczenia:** wyłączone logowanie w kliencie = luka; zdarzenia czysto GMCP
(postępy live) nie mają backfillu poza migawkami z komendy `postepy`.

---

## 7. Architektura

- **Rdzeń pure TS**: zero zależności od DOM/klienta; zależności wstrzykiwane —
  strumień linii, zegar, storage. Ten sam rdzeń przetwarza live i backfill oraz
  działa w harnessie testowym.
- **Cienka warstwa kliencka**: subskrypcje API, popup UI, storage klienta.
- **Jeden plik `index.ts`** z sekcjami (ograniczenie builda repo dargoth-plugins:
  esbuild.transform, nie bundle); guard wersji: nazwa paczki = `PLUGIN_VERSION` =
  `plugin.json`.
- **Storage pluginu**: IndexedDB (baza globalna, pole `character`), wzorzec
  Notatnika/Ethel. Perzystencja paczek w toku i stanów między sesjami.
- **Anonimowość treści repo**: zero danych wrażliwych, zero tokenów.

---

## 8. Interfejs użytkownika

Popup pluginu (registerPersistentPopup + addPopupMenuEntry), zakładki:
- **Oś czasu / dziennik** — chronologiczny strumień zdarzeń z filtrami typu i postaci.
- **Finanse** — bilans gotówka/banki, przychody/wydatki/transfery, wykresy (wszystkie
  kategorie, również pokrywające się z licznikami klienta — celowa pełność).
- **Paczki** — statusy (dostarczona/spóźniona/zwrócona/nieoddana), plan vs wykonanie,
  trasy i rentowność (czas, zarobek/h, min/max), odmowy.
- **Zlecenia** — aktywne z deadline, zrealizowane, rentowność.
- **Zabici i postępy** — tabele i wykresy sesji i lifetime (dane własne + premium).
- **Cechy** — historia odczytów, koszt w postępach.
- **Śmierci** — rejestr z kontekstem (lokacja, osłabienie).
- **Apokalipsy i czas IG** — rejestr restartów, kotwice RL↔IG.
- **Wyszukiwanie** — pełne: po tekście surowych linii, typach, kwotach, czasie,
  postaciach; składnia zwykła i regex.
- **Eksport** — pełny: JSON (format własny + kompatybilny z LogExportData v1), CSV,
  kopiowanie do schowka (tabele, podsumowania).
- **Import** — pełny: IndexedDB klienta, JSON klienta, HTML, ze schowka; dedup po
  treści; raport importu (nowe/duplikaty).
- **Audyt** — rozwijanie zdarzenia do surowej linii i źródła; lista „do przeglądu"
  (niski confidence, nieznane lokacje, kandydaci z trybu capture).
- **Tryb capture** — nieznane linie-kandydaci zapisywane do przeglądu; korpus rośnie
  w trakcie użytkowania.

---

## 9. Testy i jakość

- **Zasada red→green:** najpierw test padający na starym kodzie (dowód lokalny,
  nigdy przez CI), potem implementacja; test i poprawka w jednym commicie; wynik RED
  dokumentowany w komunikacie commita. Merge tylko przy zielonym całym zestawie.
- **Repo arkadia-dargoth-testy** (harness): prawdziwy kod pluginu (TS→esbuild→vm w
  jsdom), stub PluginApi, fake-serwer, wirtualny zegar, atrybucja wysyłek, mutanty
  (negative-controls), walidatory paczek ZIP, e2e Playwright (prawdziwy klient,
  mock-websocket). Wymóg: zegar i storage wstrzykiwane (DI) — uwzględnione w §7.
- **Korpus testowy:** rzeczywiste logi gracza (eksport JSON/HTML) — ekstrakcja linii
  kandydackich (postępy, oddanie zlecenia, zwroty paczek), testy regresji patternów.
- **Mutanty** dla maszyny paczek i klasyfikacji kasy (kolizje z §3 jako stałe testy).

---

## 10. Otwarte kwestie (do domknięcia na korpusie)

1. Dokładne linie realizacji zlecenia (oddanie towaru + zapłata/dialog zamknięcia).
2. Zbiór wartości pola `type` wpisów `ArkadiaMessagesDB` (rozróżnienie strukturalne
   komend/linii walki).
3. Potwierdzenie formy linii zwrotu paczki i jej wariantów na rzeczywistych logach.
4. Zgłoszenie upstream do arkadia-mapa: bind `depozyt` dla pokoju 10416 (Ard Skellig)
   — po stronie mapy, nieblokujące.

---

## 11. Rejestr decyzji

Podjęte:
- Pełny produkt od razu (bez wariantów okrojonych); tylko Dargoth, porty później.
- Gate lokalizacji miękki (§5). Klasyfikacja zleceń po dowodzie sprzedaży (§4).
- Sesja = login→logout; zdarzenia zawsze z timestampem; baza globalna + pole character.
- Rdzeń pure TS z DI; jeden plik index.ts; IndexedDB jako storage pluginu.
- Backfill jako obywatel pierwszej klasy (3 adaptery + echo komend).

Odrzucone / poza zakresem (z uzasadnieniem):
- Kradzież — nie istnieje na Arkadii.
- Denominacja w kantorze — niemierzalna bez porównania ekwipunku; wpływ znikomy.
- Śmierć członka drużyny — możliwa tylko live (GMCP `living`), odłożona.
- Twardy gate lokalizacji — fałszywe negatywy + unicestwia backfill.

---

## 12. Zasady repozytorium

- Treści wyłącznie profesjonalne: bez wulgaryzmów, bez danych wrażliwych, bez tokenów.
- Commity małe, tematyczne; checkbox w dokumentacji domykany w commicie swojej zmiany.
- Nigdy force-push; błędne commity zostają w historii i są dokumentowane.
- Każda zmiana kodu: red→green (§9). Commity dokumentacji nie wymagają testów.
