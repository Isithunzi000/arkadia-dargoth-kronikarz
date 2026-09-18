# Kronikarz — specyfikacja produktu

Status: **planowanie** (analiza korpusu logów zakończona, implementacja nie rozpoczęta).
Data sporządzenia: 2026-09-16. Ostatnia aktualizacja: 2026-09-18 (analiza
zrodel poczty-listow: kod klienta + testy byte-for-byte + pomoc gry + korpus;
domkniecia 59-61, otwarte 23).

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
skryptów tjurczyka, Towarzysza lub python-toolkit; **korpus** = pattern potwierdzony
na rzeczywistych logach gracza (korpus 2026-09-16: 3,86 mln linii); **GMCP** = kanał
czysto GMCP (brak backfillu z logów).

### 2.1 Paczki pocztowe

Maszyna stanów przeniesiona z modułu `analizator` repo arkadia-python-toolkit
(zweryfikowana na logach), z trzema łatkami ujawnionymi testami adwersarialnymi
(patrz §3).

| Zdarzenie | Detekcja | Weryfikacja |
|---|---|---|
| Odbiór paczki | `<NPC> przekazuje ci jakas paczke.` (NPC anonimowy lub nazwany) | kod + korpus (17 wariantów) |
| Lista ofert (cel, nagroda, limit czasu) | `packageLineRegex` (miasto + zł/sr/mdz + czas `N` lub `nieogr.`) — regex niekotwiczony do końca linii, **toleruje kolumnę `Dystans` i linie `> dystans: N` wstrzykiwane przez klienta w logach HTML** (12/12 ofert testowych, łowisko low_zlecen); ciężkie przesyłki oznaczone `*` (`Symbolem * oznaczono przesylki ciezkie.`); komenda `wybierz paczke N`, wariant `wybierz przesylke N` (wiki); odliczanie limitu w kliencie startuje od pokazania tablicy (`listTime`) — Kronikarz kotwiczy na zdarzeniu odbioru | kod + korpus + wiki |
| Etykieta paczki | `Wypisano na niej duzymi literami: <...>` w trzech strukturach: 3-członowa `NAZWA, PROFESJA, MIASTO` (121 próbek, nazwa 1–3 słowa), 2-członowa `NAZWA, MIASTO` (9, np. `ANTONIO, CAMPOGROTTA.`), 1-członowa paczka do samej poczty `POCZTA W <MIEŚCIE>` / `POCZTA MIASTA <MIASTO>` (5); sufiks pilności ` - PILNE!` (24) jako flaga metadanych; opcjonalna linia `Ponizej zas odczytujesz drobniejsze pismo:` | korpus |
| Cudzy odbiór (flavor) | `<NPC> przekazuje <komuś innemu> jakas paczke.` — ignorowane; odbiór własny kotwiczony na `przekazuje ci` | korpus |
| Dostawa | `^Oddajesz pocztowa paczke <opis adresata w dopełniaczu>\.$` + wypłata (patrz niżej); łańcuch potwierdzony kontekstami: `Odkladasz plecak` → `Bierzesz pocztowa paczke...` → `Oddajesz pocztowa paczke X.` → `X wyplaca ci ...` | kod + korpus (16+ wariantów opisów) |
| Zwrot | kotwica na **echu komendy** `→ zwroc paczke` (korpus III: 4×) + linia wynikowa `^Zwracasz pocztowa paczke` w trybie capture (korpus: N=0 w 3,86 mln linii); mechanika potwierdzona wiki: zwrot w urzędzie, z którego pobrano paczkę, niewielki spadek reputacji; helpery klienckie traktują zwrot jak dostawę (`^(Oddajesz|Zwracasz)`) — Kronikarz rozróżnia; brak wypłaty | kod (PackageHelper + tjurczyk) + korpus (echo) + wiki (mechanika); linia wynikowa: capture |
| Spóźnienie | `<NPC> mowi do ciebie: Niestety, ale dostarczyles przesylke po terminie. Dlatego moge ci za nia zaplacic tylko tyle.` — dostawa po terminie, **wypłata pomniejszona**; dotychczasowa flaga `Dostarczyles przesylke po terminie` z toolkitu zostaje jako wariant | kod (toolkit) + **korpus III** (łowisko 2026-09-17: 4+ wystąpień; NPC: Luter/Andolf, Guy/Gautier, Lit, anonimowy) |
| Wypłata | `wyplaca ci <waluta>` (waluta wymagana; multi-nominał ze spójnikiem `i`, końcówki `moneta/monety/monet`) LUB `otrzymujesz <waluta>` w oknie ≤ 2 linie od dostawy, bez frazy ` od <kogoś>` | kod + korpus (15 wystąpień; dominuje 4 sr 2 mdz, występują 3-nominałowe) |
| Nieudany odbiór | `Pocztowa paczka jest zbyt ciezka.`, `Lista przesylek zmienila sie i ta, ktora chcesz podjac byc moze nie jest juz ta, ktora widziales w spisie...` | korpus (15× / 34×) |
| Kamień milowy zaufania | `Uwazam cie za osobe wiarygodna i powierze ci kazda przesylke, ktorej zechcesz sie podjac.`, `Jestes uwazany za naprawde wiarygodna osobe, w zwiazku z tym moge powierzyc ci prawie kazda przesylke.` | korpus (14× / 45×) |
| Odmowa odbioru | `Ty juz dla nas dostatecznie ciezko zapracowales`, `Nie ufam ci na tyle, aby powierzyc ci dostarczenie tej przesylki`, `Cos ci sie chyba pomylilo, nie ma takiej oferty`, `nie widzisz tu nikogo, od kogo mozna by wziac zlecenie`; filtrowanie cudzych odmów: liczą się tylko linie `mowi do ciebie`, nie `mowi do <inny gracz>` | kod (tjurczyk) + korpus |
| Nieoddana | paczka otwarta bez domknięcia; persystowana między sesjami, widoczna w raporcie jako „w toku" | projekt |
| Reputacja pocztowa | licznik heurystyczny per rewir (dostawa +1, spóźnienie/zwrot −1, zagubienie = blokada ofert ~24 h); kalibracja komendą `sprawdz swoja reputacje` — **skala 6 gradacji odpowiedzi pracownika poczty (pelny korpus)**: `Nie znam cie wcale.` / `Moge ci powierzyc jedynie przesylki lokalne i do najblizszych miasteczek.` / `Nie powierze ci zadnej dalszej przesylki, ale ufam na tyle, aby dac jakas blizsza.` / `Tak, ufam ci na tyle, aby powierzyc ci nawet te dalsze przesylki.` / `Uwazam cie za osobe wiarygodna i powierze ci kazda przesylke, ktorej zechcesz sie podjac.` (+ dopisek o przystapieniu do pocztylionow) / wariant formy trzecioosobowej per NPC: `Jestes uwazany za naprawde wiarygodna osobe, w zwiazku z tym moge powierzyc ci prawie kazda przesylke.`; kamienie milowe zaufania (wiersz wyżej) jako pośredni sygnał progu | wiki + korpus (gradacje: pelny korpus 576 sesji) |

Statusy paczki: **dostarczona, spóźniona, zwrócona, nieoddana** — wszystkie obsługiwane.
Ekwiwalent `_merge_cross_session` z toolkitu: paczka w toku przetrwa relogin (storage).
Weryfikacja referencyjna: moduł paczek toolkitu jest operacyjny na logach (pary
odbiór/dostawa, spóźnione `[OPOZ]`, nieoddane, zarobek) — jego raport służy jako
niezależny wzorzec dla testów regresji maszyny Kronikarza.

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

**Mechanika czasowa i reputacji (wiki „Pocztylioni", 2026-09-17):** reputacja liczona
osobno per rewir pocztowy; dostawa ją podnosi, spóźnienie, zwrot i zagubienie
obniżają. Oferty skalują się reputacją: paczki lokalne (~1 zł), bliższe (2–4 zł),
dalsze (5–9 zł), najdalsze (10–12 zł, wszystkie do 45 zł). Ciężar paczki ograniczony
(~50 kg; blokada `Pocztowa paczka jest zbyt ciezka.`, korpus 15×). Paczka nieoddana
znika 6 h po wylogowaniu (liczone od pobrania); zagubienie blokuje nowe oferty na
~24 h. Zwrot możliwy w urzędzie, z którego paczka została pobrana (niewielki spadek
reputacji). Licencja pocztowa (Tilea) poza katalogiem — brak kotwic w korpusie.

**Baza adresatów (kod klienta + korpus, 2026-09-17):** zdalna baza NPC
(`arkadia-mapa/data/npc.json`, 637 rekordów, TTL 24 h, merge z wpisami lokalnymi,
dedupe po `name-loc`) pokrywa 111/111 adresatów-osób z korpusu — douczanie lokalne
potrzebne wyłącznie dla nowych NPC. Etykiety 1-członowe (`POCZTA W X` / `POCZTA
MIASTA X`, 5 próbek) celowo poza bazą NPC: mapowane na lokacje poczt (§5), nie na
osoby. Parser etykiety klienta (PackageHelper) łapie tylko pierwszy człon (nazwę) —
Kronikarz parsuje wszystkie trzy struktury + sufiks ` - PILNE!`.

### 2.2 Pieniądze

Przeliczniki (dwa niezależne źródła: Towarzysz `coins.ts` i toolkit `denominacja`):
**1 mth = 100 zł = 24000 mdz; 1 zł = 20 sr = 240 mdz; 1 sr = 12 mdz.**

| Zdarzenie | Detekcja | Weryfikacja |
|---|---|---|
| Łup / kasa otrzymana | trójpodział `Bierzesz`: `... z ciala/sterty` = łup, `... z otwartej/otwartego <pojemnika>` = transfer (wiersz niżej), bez źródła = łup z ziemi (korpus N=0, pattern gry; Towarzysz świadomie go pomija — Kronikarz jako księga łapie); + `^Dostajesz (.+)\.$`, `^Otrzymujesz (.+)\.$` (**brak w aktualnym LOOT_PATTERNS Towarzysza** — wzorzec trzymany na korpusie), `wyplaca ci (.+)` (waluta wymagana, §3) | kod (Towarzysz LOOT_PATTERNS) + korpus |
| Wydatek | `^Kupujesz `, `^Placisz ` (obejmuje przejazdy: `Placisz <komu> <kwota>` oraz wariant bez kwoty `Placisz woznicy i wspinasz sie...`), `zgarnia ... monet` (z dowolnym wtrętem, np. `drapieznym ruchem zgarnia`), `odbiera od ciebie ... monet ... w zamian za zakupion` | kod (Towarzysz SPEND_PATTERNS) + korpus |
| Wydatek z resztą | `Placisz <kwota> i dostajesz <kwota> reszty.` — wydatek netto = zapłacone − reszta; reszta potrafi zawierać mithryl (`Placisz 1 mithrylowa monete i dostajesz 64 zlote, 47 srebrnych i 27 miedzianych monet reszty.`) | korpus |
| Usługa: naprawa | `Oddajesz <NPC> <przedmiot> ze stojka, placac <kwota słownie>.` — oddanie ubioru krawcowi (Novigrad, Campogrotta, Nuln) lub oreża/zbroi kowalowi do naprawy; wydatek kategorii „usługa", nie zakup | korpus + wiedza domenowa |
| Reszta od NPC | `<NPC> <wtręt dowolny> wrecza ci <kwota>( reszty)?\.` — regex przepuszcza dowolny wtręt między podmiotem a czasownikiem. **Reguła:** obecność słowa `reszty` = reszta od nadpłaty przy zakupie (koszt = zapłacone − reszta); brak słowa `reszty` = niejednoznaczne (reszta albo zapłata za sprzedaż) — klasyfikacja po parze/kontekście | korpus (flavory: `ze sztucznym usmiechem`, `kryjac usmiech`, `z ogromna niechecia`) |
| Usługa: wynajem wozu + kaucja | `^Wynajmujesz (.+?),? placac (.+?) kosztu najmu(?: oraz)? (.+?) (?:zwrotnej )?kaucji\.$` (przecinek, `oraz`, `zwrotnej` opcjonalne), np. `Wynajmujesz lekki woz, placac dwadziescia piec zlotych monet kosztu najmu oraz jedna mithrylowa monete zwrotnej kaucji.` — **dwie kwoty w jednej linii:** koszt najmu = wydatek (usługa), kaucja = depozyt zwrotny (nie wydatek); zwrot pojazdu `Zwracasz <pojazd>`; mechanika z kodu klienta: pełna kaucja do 6h od wynajmu, potem częściowa; linia refundacji kaucji nieznana z kodu (klient jej nie triggeruje — prawdopodobnie generyczna linia przychodu) → tryb capture / pełny korpus. To NIE jest przejazd (`Placisz woznicy`) — osobna gałąź | kod (klient: carriage.ts) |
| Kupno u flavor-sklepikarzy | para/trójka linii: `<NPC> drapieznym ruchem zgarnia <kwota>.` (pobranie zapłaty — wydatek) + `<NPC> kryjac usmiech wrecza ci <towar>.` (wydanie towaru — **bez wpływu na księgę**) + opcjonalnie `... wrecza ci <kwota> reszty.` | korpus (Salithrandir, Zykkis, Adipatus, Myrrhis) |
| Transfer gracz→gracz | `<Gracz> daje ci <moneta>.` (wielka litera imienia, brak tagu `(NPC)`) — przychód oznaczany jako transfer od gracza | korpus (Gwenn, Ulik) |
| Transfer gotówka↔pojemnik | `Wkladasz <kwota> monet\w* do otwartej/otwartego <pojemnika>.` / `Bierzesz <kwota> ... z otwartej/otwartego <pojemnika>.` (sakiewka/plecak — korpus: 126×/111×/62×…; skrzynka depozytowa = gałąź §2.3); **transfer, nigdy przychód/wydatek**; kwota często nieokreślona `wiele` → null (reguła 6 niżej) | korpus |
| Podgląd pojemnika (migawka) | `Rozwiazujesz na chwile rzemyk, sprawdzajac zawartosc swojej ... <pojemnika>. W srodku dostrzegasz <lista monet>.` — migawka stanu na żądanie (rodzina `wiedza`/`cechy`), **nie zdarzenie księgowe**; listy mieszane `wiele X, osiem Y` (gra drukuje `wiele` powyżej progu widoczności) | korpus (25×+ dla sakiewki) |
| Sprzedaż | `^Sprzedajesz ` + zapłata osobną linią przychodu | kod (Towarzysz SELL_PATTERNS) + korpus |
| Liczby słowne | mapowanie liczebników polskich (Towarzysz `polishNumbers`, klient `contracts.ts` POLISH_NUMBERS); **listy mieszane**: słownie i cyfry w jednej linii (`szesc srebrnych monet, 76 zlotych monet i siedem miedzianych monet`) | kod + korpus |
| Wartość ekwipunku | surowa forma gry: `^Wydaje ci sie, ze (jest|sa) wart[aye]? okolo N mied\S+\.$` + warianty stosu z kodu `Sa tu N sztuki warte ...` / `Jest tu N sztuk wartych ...` (korpus N=0) — wariant `Jest tu` potrafi nieść opcjonalny prefiks `Wydaje ci sie, ze ` (klient: stoneValue.ts) — + `nie ma wiekszej wartosci` (bez kwoty); **w logach HTML klient dokleja `, czyli X zl, Y sr, Z mdz`** (priceEvaluation.processItemValue — korpus: 1030× dla 170 mdz) — backfill odcina/toleruje sufiks, bonus: gotowa normalizacja; niezależne potwierdzenie przeliczników (170 mdz = 14 sr 2 mdz; 4600 mdz = 19 zł 3 sr 4 mdz) | kod (klient: priceEvaluation) + korpus |

**Twarda zasada:** linia bez jawnego nominału (`monet` + liczba/liczebnik) nie ma wpływu
na księgę. Frazy `daje ci / wrecza ci / przekazuje ci` bez waluty dotyczą rzeczy lub
paczek, nie pieniędzy. Zakup bez kwoty (`Kupujesz butelke oleju.`) — zero wpływu,
zdarzenie ewentualnie jako statystyka. Antyprzykład korpusowy (łowisko v2): opis
pokoju kantoru `kasjer szybko zgarnia wymieniane monety spogladajac...` łapie się
we wzorzec `zgarnia ... monet` — reguła kwoty z liczbą odfiltrowuje go automatycznie.

**Reguły parsera kwot (dowody z korpusu):**
1. Multi-nominał ze spójnikiem `i` i przecinkami; odmiana `moneta/monety/monet` zależna
   od liczby.
2. Potoczne nazwy denominacji: `miedziaki` (= mdz), `srebrniki` (= sr) — występują
   w liniach `Kupujesz ..., placac N miedziakow/srebrnikow`.
3. Mithryl w obiegu codziennym (wypłaty reszty, przelewy graczy, depozyty).
4. Kwoty **nieznormalizowane** istnieją (`Dostajesz 15 srebrnych i 78 miedzianych
   monet`) — parser nie zakłada, że mdz < 12 ani sr < 20.
5. Zdanie może ciągnąć się po kwocie (`Placisz wlascicielce dwanascie srebrnych i szesc
   miedzianych monet i zamawiasz wybrany smakolyk.`) — kotwica na kwocie, nie na
   końcu linii.
6. **Liczebniki nieokreślone** (`wiele`, `kilka`, `sporo`, `troche`, `duzo`) → kwota
   **nieznana (null)**, nigdy liczba: Towarzysz policzyłby `kilka miedzianych` jako 1
   (`amount ?? 1`), toolkit jako 0 — oba fałszują księgę. Korpus: `wiele` masowo przy
   pojemnikach; `kilka` przy kasie N=0 (dotąd tylko test adwersarialny §3).
7. **Unia tabel liczebników:** mianownik (Towarzysz polishNumbers: `jeden`…`dziewietnascie`
   + dziesiątki/setki/tysiące) + dopełniacz/biernik (klient contracts POLISH_NUMBERS:
   `jednej/jednego/dwoch/dwu/trzech/pieciu…`, dwuwyrazowe `trzydziestu jeden`).
   Towarzysz nie zna dopełniacza dziesiątek (`trzydziestu`) — kopia 1:1 gubiłaby
   kwoty słowne w odmianie. Uzupełnienia z kodu Dargotha (2026-09-17):
   `polishNumberConverter` (257 form: mian.+dopełn. 1–99 wraz ze WSZYSTKIMI złożeniami
   dwuwyrazowymi 21–99 w obu przypadkach + liczebniki zbiorowe `dwoje…dziesiecioro`
   z odmianą) — konwerter NIE zna setek ani tysięcy (pokrywa je Towarzysz);
   contracts.ts dodaje formy spoza konwertera: `dwu`, `dwudziestu/trzydziestu/
   czterdziestu dwu`, złożenia z `jednego/jednej` (`dwudziestu jednego`…) — unia
   Kronikarza obejmuje wszystkie trzy tabele. Anomalia rozstrzygnięta (pelny
   korpus 576 sesji): typo-formy contracts.ts `piedziesiat`, `pieedziesieciu`
   to **martwe klucze** (N=0) — gra jest spojna: 4 warianty 50-tki
   (`piecdziesiat`, `piecdziesieciu`, `piecdziesiata`, `piecdziesiecioma`);
   forma `piescdziesiat` to literowka GRACZA w mowie (kanal comm), nie tekst
   gry — poza parserem kwot. Formy zlozone jednostka+setka slownie
   (`siedem czterysta`, `tysiac szescset`) wystepuja w mowie graczy, nie w
   liniach kasowych — unia pokrywa je zapasowo.

**Kantor — pełne zdarzenie bez kwoty (dowód korpusowy):** komenda `zdenominuj` (81 ech)
drukuje wyłącznie `Twoje pieniadze zostaly zdenominowane.` (61×) lub `Twoje pieniadze
juz sa maksymalnie zdenominowane.` (20×); w kontekstach ±3 linie zero kwot. Zapisujemy
pełnoprawne zdarzenie denominacji: timestamp (log-time w backfillu / czas live),
lokacja (live: GMCP `room.info`; backfill: nazwa pokoju — pokoje korpusowe: `Kantor
banku w Daevon.`, `Kantorek Vimme Vivaldiego.`, `Posterunek celny i kantor wymiany
walut.`, `Niewielki kantorek.`, generyczny `Kantor.`) oraz wynik (wykonana /
maksymalnie zdenominowane). Zdarzenie wchodzi do osi czasu, wyszukiwania i filtrów
jak każde inne; statystyki: łącznie, per kantor, per postać, per sesja. Z tabliczek
kantorów: `Za kazda transakcje pobieramy tylko 8 procent prowizji.` (forma slowna)
oraz druga forma z procentem cyfra `...pobieramy tylko 5% prowizji.` (pelny
korpus: 14 wyst.); tabliczka 3% potwierdzona korpusem u Vimme Vivaldiego —
prowizja pozostaje niewidzialnym mikrowydatkiem (wymiana nie zmienia majątku poza prowizją).
Świadomie poza bilansowaniem — ale nie poza kroniką. **Prowizja jest per bank, nie
globalna** (wiki „Pieniądze"): 3% (Nuln, Novigrad, Wyzima, Ard Skellige, Carbon),
5% (Ebino, Karak Varn, Parravon, Quenelles, Daevon, Baccala, Hagge, Maribor,
Mons Arx, Oxenfurt, Rinde, Craag Ros, Athel-Loren, Campogrotta, Toskania), 8%
(Kraina Zgromadzenia, Kreutzhofen, Ubersreik, Varieno, Averheim, Val'Kare, Twierdza
Slaanesha, Zakon Sigmara — na mapie „Bank, Zamek Sigmara", Averland); Guleta i Scala:
depozyt bez denominacji. Tabliczka „8 procent" z korpusu
to stawka lokalna — stawka znana z góry per lokacja trafia do metadanych zdarzenia.
Kantor spoza tabeli wiki: „Kantor, Bank, Sklep, Eysenlaan" (dane mapy, banki bez
binda) — stawka nieznana, do ustalenia z tabliczki (tryb capture).

**Nie istnieje w grze:** kradzież/okradzenie — poza katalogiem.

**Barter poza księgą:** sklepy z nietypowymi środkami płatniczymi (wiki: esencja
życia, gruczoły pająków, kły wampirów, skalpy driad, zapiski) — płatność towarem
bez nominału nie rusza księgi (twarda zasada); ewentualnie zdarzenie informacyjne.

**GMCP:** brak kanału stanu gotówki — **potwierdzone w oficjalnej specyfikacji GMCP
Arkadii** (forum t=740, akt. 2026-09-17): moduły Core, Char, Room, Objects,
Gmcp_msgs, Mail; `Char.State` = hp/mana/fatigue/improve/form/intox/headache/
stuffed/soaked/encumbrance/panic — żadnego pola pieniężnego; kod klienta (arkadia,
Dargoth) subskrybuje zgodnie 1:1. Kasa zawsze z tekstu lub premium storage.

### 2.3 Bank i depozyty

| Zdarzenie | Detekcja | Weryfikacja |
|---|---|---|
| Odczyt depozytu (migawka) | `Twoj depozyt zawiera ...` (monety w liście — słownie i cyframi mieszanie), `Twoj depozyt jest pusty`, `Nie posiadasz wykupionego...` — **migawka stanu na żądanie** (rodzina podglądu pojemnika §2.2), nie zdarzenie księgowe; w logach linia bywa **zawinięta w środku fraz** (§6.2 sklejanie) | kod (klient: deposits.ts) + korpus |
| Wpłata / wypłata | triggery kontekstowe na lokacji z bindem `depozyt` (lista referencyjna, §5); linie `Wkladasz/Bierzesz <coś> do/z otwartej skrzynki depozytowej.` (w tym monety: `Wkladasz dwie mithrylowe monety do otwartej skrzynki depozytowej.`) | kod + mapa + korpus |
| Wykupienie / rozbudowa skrzynki | **wydatek kategorii „usługa bankowa"**: podstawa 50 złotych, poziomy rozbudowy 2 / 5 / 10 / 20 mithryli (wiki „Skrytki"); raz wykupiona działa **do końca gry postacią** (zero odnawiania; śmierć = utrata depozytu, §2.8); limit 25 przedmiotów w depozycie niezależnie od rozbudowy (stos jednego rodzaju = 1 przedmiot; notka Mistrza Rafgarta na tablicy, korpus). Linia gry nieznana (żaden klient nie triggeruje, korpusy nie pokazały) + komenda pomocy `?depozyt` — **tryb capture** | wiki „Skrytki" + korpus (tablica) |
| Cudze operacje | `<Gracz> bierze ... ze swojej otwartej skrzynki depozytowej.` / `<Gracz> wklada ... do swojej otwartej skrzynki depozytowej.` — marker `swojej` = odfiltrować (nie nasz depozyt) | korpus |
| Stan konta per bank | premium: odczyt storage klienta klucz `deposits` (characterStorage); klient trzyma `wiele` jako pseudo-count `'wie'` poza sumą (zgodne z regułą 6 §2.2); konwerter klienta kończy na 99 — lista z `sto+` słownie nie sparsuje się klientowi, unia Kronikarza pokrywa (§2.2 reguła 7); pelny korpus: forma `sto+` slownie N=0 w listach depozytu i liniach kasowych — unia z setkami zostaje zapasowo | kod + korpus (N=0 na 576 sesjach) |

**Zasada księgowa:** wpłata i wypłata to **transfer** (przesunięcie gotówka ↔ bank),
nigdy przychód ani wydatek. Bilans majątku pokazuje gotówkę i depozyty osobno i łącznie.

Backfill: echo `→ depozyt` + odczyt w logach daje historię stanów banków. Nazwy sal
bankowych **w grze ≠ nazwy mapy** (gra: `Glowna sala banku.`, mapa: `Bank w Daevon`)
— lista korpusowa 18 nazw sal domknieta na pelnym korpusie (dane referencyjne,
klucz `korpus_pelny_2026_09_18`).

**Komendy zarządzania depozytem** (wiki „Skrytki"): `przejrzyj [pobieznie] <co>
[z czego]` (filtry: uszkodzone, naprawialne, typy broni/zbroi), `wybierz <co>` —
echo tych komend w backfillu to markery kontekstu depozytowego (rodzina
`→ przejrzyj depozyt`).

**Komendy `wplac`/`wyplac`/`przelej` — rozstrzygniete (pelny korpus 576 sesji):**
`wplac`/`wyplac` N=0 — martwe legacy po likwidacji kont procentowych w 2011
(wiki „Pieniądze"), **poza katalogiem**; `przelej` to komenda **przelewania
plynow** (`Napelniasz/Dopelniasz <naczynie> <plynem> z <naczynia>`) — NIGDY
operacja bankowa: kolizja nazwy, twardy filtr (zdarzenie nie wchodzi do ksiegi).

**Zasada świata:** zakaz pośredniego i bezpośredniego przekazywania pieniędzy
między własnymi postaciami gracza (wiki „Pieniądze") — kontekst interpretacji
transferów gracz→gracz (§2.2); Kronikarz księguje fakty, nie ocenia.

**Historia (nie istnieje):** konta bankowe z prowizją procentową od wpłaty/wypłaty
(pre-2011, wiki „Pieniądze") — zastąpione skrzynkami depozytowymi; wyjaśnia
potencjalny status legacy komend `wplac`/`wyplac`.

### 2.4 Zlecenia (kontrakty NPC)

| Zdarzenie | Detekcja | Weryfikacja |
|---|---|---|
| Zapytanie | `^Pytasz .+ o zlecenie\.$` (otwiera kontekst; klient zapisuje `locationId` pokoju) | kod (klient: contracts.ts) |
| Oferta | `.+? \S+ do [^:]+: Tak, mam pewne pilne zamowienie na ([^.]+)\. Potrzebuje (?:jeszcze )?([^.,]+?)(?:, przynajmniej ([^.]+) jakosci)?\.` | kod |
| Termin | `.+? \S+ do [^:]+: Na realizacje zamowienia mam ... (dni/dzien/godzin/godziny/godzine), pozniej zapewne bede potrzebowac czego innego\.` Konwersja: **1 dzień IG = 48 minut realnych** (kod: ONE_INGAME_DAY_MS), godzina IG = 2 min RL; `kilka godzin` = 0,5 dnia (korpus 2×), brak liczebnika (`mam dzien`) = 1 dzień; deadline kotwiczony na czasie nadania oferty | kod + korpus |
| Brak zlecenia | `.+? \S+ do [^:]+: Nie, w tej chwili niczego mi nie trzeba\. Zajrzyj moze za jakis czas\.` (zamyka kontekst, czyści kontrakty lokacji) | kod |
| Realizacja | **Sekwencja (konteksty łowiska v2):** echo komendy `→ daj <towaru> <NPC>` (dopełniacz partitywny, np. `→ daj miesiwa mezczyznie`) → **para linii na każdą sztukę**: `<NPC> mowi do ciebie: Dziekuje, potrzebuje jeszcze <pozostała ilość>.` (tracker postępu, warianty: „dwoch kilogramow", „ponad kilogram") + `<NPC> odbiera od ciebie <towar> i wrecza ci <kwota>.` **Brak linii `Dajesz/Oddajesz` po stronie gracza** — dowód to echo + linie NPC. Jedna komenda `daj` może dać N dostaw. Dowód: oferta „czterech kilogramow miesa z zajaca" (00:32:07) → 41 s później dwie pary postęp+zapłata od `Wysoki zwinny mezczyzna` (4 zł 8 sr 4 mdz + 4 zł 13 sr). **Płatność pro-rata (decyzja 2026-09-17):** kasa przychodzi z każdą dostawą osobno, niezależnie od stopnia ukończenia — zdarzenie finansowe = para linii na każdą sztukę/kilogram; osobnej linii finansowej „finalizacji" nie ma i nie trzeba jej capture'ować dla księgi (ewentualny tekst po ostatniej dostawie to co najwyżej informacja). Samo `odbiera od ciebie` jest **trójznaczne** (blok niżej) — kotwica na pełnej formie z `i wrecza ci`; reguła §4 (kolizja ze sprzedażą) bez zmian. Lokalizacja dowodu: Parravon | korpus III + łowisko v2 (2026-09-17) |
| Realizacja give-based (bounty) | `Dajesz/Oddajesz <NPC> <przedmiot>.` + okno: `<NPC> mowi do ciebie: ... daje <kwote> ... .` i/lub `<NPC> wrecza ci monety.` (kwota w komentarzu NPC, linia wręczenia bez kwoty — łączyć w oknie). Potwierdzone: Adler, ciała szczurów | korpus (grepy 2026-09-17) |
| Odmowa dawania | `<NPC> mowi do ciebie: A po co mi to dajesz?` — nieudana próba `daj` (NPC nie chce towaru), osobne zdarzenie | korpus (6× Adler) |
| Reklama kontraktu myśliwskiego | `Mam zlecenie na swieze (skory/ryby/mieso), chetnie za nie zaplace.`, `Mam zlecenie na kilka sztuk broni, chetnie za nie zaplace.` | korpus (Lucciano, Benito, Aubert, Naula/Rudolf, Szczuroslaw 10×, Fuats 6×) |
| Oferta (warianty korpusowe) | towar na wagę: `Potrzebuje trzydziestu jeden kilogramow miesa z sarny. Dobrze zaplace!`; z jakością: `Potrzebuje osmiu tarcz, przynajmniej sredniej jakosci. Dobrze zaplace za kazda sztuke.`; z rozmiarem/typem: `dziesieciu srednich ryb slodkowodnych`; sufiksy `Dobrze zaplace!` / `Dobrze zaplace za kazda sztuke.` | korpus (Anatol, Ferdinand, Mortimer, Ghaadrav) |
| Cudze oferty | oferty kierowane do innych graczy (`mowi do <ktoś>:` zamiast `mowi do ciebie:`) — ignorowane | korpus |
| Tablica bounty | `| Zleceniodawca | Scigana osoba | Data` | korpus |
| Ogłoszenie bounty (przekrzyk) | `<NPC> krzyczy (z oddali )?po bretonsku, ale udaje ci sie zrozumiec tylko czesc: ...` — ogłoszenie bounty na potwory czytane na głos w niektórych miastach (ekwiwalent listu gończego); tekst **urwany**, kwota niewiarygodna → kategoria informacyjna, nie zdarzenie finansowe. Warianty podmiotu: `Jakis mezczyzna krzyczy z oddali ...`, `Szczuply ryzy mezczyzna krzyczy ...` | korpus III |

**Trójznaczność `odbiera od ciebie` (korpus III):**
1. `<NPC> odbiera od ciebie <towar> i wrecza ci <kwota>.` — **realizacja zlecenia**
   (towar+kasa w jednej linii).
2. `<NPC> odbiera od ciebie <przedmiot>.` — **sprzedaż sklepowa** (NPC przejmuje
   sprzedany przedmiot, bez kasy w linii; sklepikarze: Antonietta, Olof, Ernest).
3. `<NPC> odbiera od ciebie <kwota> w zamian za zakupiony towar.` — **zakup** (NPC
   pobiera zapłatę od gracza; §2.2).

**Liczebniki i ilości w ofertach (korpus + kod, 2026-09-17):** parser zleceń
Kronikarza stoi na **unii liczebników** (§2.2 reguła 7), NIE na tabeli contracts.ts
— tamtejsza POLISH_NUMBERS kończy się na 50, a korpus ma ofertę `siedemdziesieciu
dwoch kilogramow miesa z dzika` (72 kg; Dargoth sparsowałby count:1). Unia pokrywa
60–90 z złożeniami (osobny konwerter klienta). Oferty bywają też **cyfrowe ≥100**:
`110 kilogramow miesa z losia` — parser musi łykać czyste cyfry. Typy towarów
korpusowe: ryby (sztuki + rozmiar + typ), mieso (kg, zwierzę), zbroje (sztuki,
jakość opcjonalna), **skóry** (`dwunastu skor susla`). Zleceniodawcą może być NPC
z gołym imieniem bez opisu (korpus: `Petyr`).

**Kolizje kategorii (korpus):**
1. `Pracownik poczty mowi: Mam (calkiem) nowe zlecenia.` to ogłoszenie o **paczkach**,
   nie kontrakt — słowo „zlecenie" bez kontekstu nadawcy jest dwuznaczne.
2. **Zleceniodawcy są jednocześnie adresatami paczek** (etykiety: `LUCCIANO,
   HANDLARZ, EBINO`; `RUDOLF KARCZMARZ, NULN`; `BENITO SANGIOVESI, RESTAURATOR,
   KREUTZHOFEN`; `AUBERT GRIBAUX, RZEZNIK, QUENELLES`). Wypłaty `wyplaca ci` od tych
   NPC w korpusie to wypłaty za **paczki** (dowód: konteksty `Oddajesz pocztowa
   paczke X.` → `X wyplaca ci ...`). Wniosek: klasyfikacja kasy wyłącznie po dowodzie
   (linia `Sprzedajesz` w oknie), nigdy po samym NPC czy kwocie.
3. `odbierz zamowienie` / `zloz zamowienie` to **zamówienia rzemieślnicze** (wytwórcy:
   `Przyjdz tedy i 'odbierz zamowienie'.`, `Przeciez nie skladales zadnego
   zamowienia!`) — odrębna mechanika, nie mieszać ze zleceniami. Ich linia kasowa:
   `Placisz <NPC> <kwota> i skladasz zamowienie.` (korpus 4×) — wydatek
   „zamówienie rzemieślnicze", nie zlecenie i nie zakup sklepowy.

**Bounty za ciała szczurów (korpus III 2026-09-17):** mechanika młodego expa —
zabijasz szczury (zabójstwo liczone **normalnie** w statystykach zabitych, jak każde
inne), a ciała oddajesz za kasę szczurolapom. Pełna sekwencja: `daj <ciała> <NPC>`
→ `<NPC> oglada uwaznie cialo.` (lub `... sterte szczatkow szczura.`) → `<NPC> mowi
do ciebie: Dorodny okaz! Za takiego slicznego szczurka daje trzy srebrne monety.`
→ `<NPC> wrecza ci monety.` → `<NPC> usmiecha sie z zadowoleniem.` Cennik: szczur
3 sr, **mysz 8 pensów** (`Adler mowi: ... mysz - osiem pensow`). Potoczne nazwy
monet w cennikach: **szyling = srebrna moneta** (`Ratan mowi: Kazdy martwy szczur
wart trzy srebrne szylingi.` = 3 sr), **pens = miedziana moneta** (8 pensów za
mysz, taniej niż szczur). Klasyfikacja kasy:
przychód **bounty „zapłata za szczury"**, NIE realizacja zlecenia towarowego.
NPC-e: **Adler Winck → Nuln** (potwierdzone korpusem: who-lista `Adler Winck, Nuln`,
opis `Koscisty wysoki mezczyzna (Adler NPC)`, pokój `Biuro szczurolapa.`) i **Ratan**
(`Obdarty brudny mezczyzna (Ratan NPC)`; szept do gracza: `zapytaj szczurolapa o
prace`). Wykluczenia szumu: „Szczurolap" to także **tytuł zawodowy graczy** na who
(`Szczurolap, halfling`, `Doswiadczony Szczurolap` — 419×/187×, np. Krux Wald, Dove
Livett) — nie mylić z NPC; `szczury ladowe` to obelga żeglarska; maskotka `szary
szmaciany szczur` (prezenty dla graczy) — separacja od ciał potwierdzona oknem
kasowym (9 dań graczom, zero kasy w 90 s).

**Linia dawania ciał nie istnieje (łowisko v2, E10):** w całym korpusie zero linii
`Dajesz/Oddajesz ... cial*` do jakiegokolwiek NPC (jedyna taka linia to ciało
skavena wręczone **graczowi**) — sekwencja bounty kotwiczy na **echu komendy**
(`→ daj ciala szczurolapowi`, liczba mnoga) + liniach NPC (`oglada uwaznie cialo`
lub `... sterte szczatkow szczura.` → komentarz z ceną → `wrecza ci monety` →
`usmiecha sie z zadowoleniem.`). Bounty obejmuje też **ciala skavenów** (wpis
wiedzy `Dostarczyles cialo skavena szczurolapowi z Nuln.` — §6.2). Obydwa NPC-e
zaczepiają gracza szeptem: `Jesli chcesz zarobic troche grosza, to szukam kogos do
pomocy. <zapytaj szczurolapa o prace>...` (Adler dodatkowo wariant bezpośredni:
`Zapytaj mnie o prace, to wyjasnie szczegoly.`). Sesja korpusowa: polowanie
2025-12-14 23:03–23:09 (15 szczurów, licznik (1/1)→(15/15)), potem 7× odmowa
`A po co mi to dajesz?` (23:12:53 — spam `daj` niewłaściwym przedmiotem).

**Katalog wykluczeń dla linii `Dajesz/Oddajesz` (korpus, grepy 2026-09-17):**
1. `Oddajesz pocztowa paczke ...` — paczki (§2.1).
2. `Oddajesz ... ze stojka, placac ...` — naprawa u krawca/kowala (§2.2).
3. Transfery do graczy — odbiorca to imię własne / tag GP (`Dajesz elfi chleb Gwenn.`,
   `Dajesz przepyszny makowiec Gwenn.`; echa `daj szczura/wianek/jedzenie elfce` =
   Gwenn, gracz-elfka) — transfer gracz→gracz (§2.2).
4. Bilety transportowe — `Dajesz zatluszczony czerwony bilet starszemu spokojnemu
   mezczyznie.`, `Dajesz niewielki szary bilet ciemnowlosemu przyjacielskiemu
   mezczyznie.` — osobna kategoria: odprawa transportu.
5. `Nie dajesz rady ...` (np. `uniesc drewnianej klapy`) — negacja-zdolność, nie
   zdarzenie; twardy filtr.

**Anomalia tagów (korpus):** formułę oferty `Mam zlecenie na swieze ...` wygłaszają
też postaci z tagiem **GP** (gracz: Anatol, Ghaadrav) — kolizja tagu lub relacja
gracza; parser kotwiczy wyłącznie na tagu `NPC`, GP idzie do przeglądu.

Premium: odczyt storage klienta klucz `contracts` (aktywne zlecenia z `locationId`,
przedmiotem, liczbą, jakością i deadline). Fallback: własne śledzenie tymi samymi
patternami.

### 2.5 Zabici

| Zdarzenie | Detekcja | Weryfikacja |
|---|---|---|
| Zabójstwo własne | forma surowa (live, z kodu): `^[ >]*Zabil(?<v>es|as) (?<name>...)\.$`; **forma w logach HTML (backfill): klient przepisuje linię — `[ ZABILES ] Zabiles <name>. (<n> / <m>)`, postać żeńska: `[  ZABILAS  ] Zabilas <name>. (n / m)` (kod; korpus N=0 — postać męska)** | kod (kill.ts) + korpus (241 wyst., 103 unikalne) |
| Zabójstwo drużyny | forma surowa: `^[ >]*(?<player>...) zabil(?<v>a?) (?<name>...)\.$`; w logach: `[ ZABIL ] <ktoś> zabil <name>. (n / m)`, `[ ZABILA ] <ktoś> zabila <name>. (n / m)`. **Zabójca spoza drużyny (OTHER) dostaje ten sam prefix, ale BEZ suffixu licznika** (kod: `formatPrefix` z pustym licznikiem; korpus: `Feril zabil wscieklego groznego wilka.` 3×) — obce zabójstwo to **filtr, nie zdarzenie kroniki**: nigdy do statystyk własnych/drużyny | kod (kill.ts) + korpus (174+150 wyst., 81+77 unikalnych) |
| Suffix licznika | ` (n / m)` na końcu przepisanej linii — **n = moje zabójstwa tego moba w sesji, m = moje + drużyny w sesji** (kod: `mySession / mySession + teamSession`, zakres sesyjny, NIE lifetime; korpus: `(0 / 1)`, `(5 / 17)`); metadane gratis, parser toleruje i wykorzystuje | kod + korpus |

**Uwaga korpusowa (zabici):** surowa forma `Zabiles X.` w logach HTML **nie występuje**
— klient przepisuje linię przed zapisem (prefix `[ ZABILES ]` / `[ ZABIL ]` /
`[ ZABILA ]`, suffix licznika). Parser backfillu kotwiczy na przepisanej formie.
**Znacznik ma dopełnienie spacjami, ale szerokość dryfuje między wersjami klienta**
(łowisko v2: 1796 linii `[ ZABIL* ]` w korpusie; digesty 2026-09-18: **1 spacja
dominuje** — 115+81+77, 2 spacje 34, 3 spacje 14) — regex kotwiczy na
`^\[\s*ZABIL(?:ES|AS|A|)\s*\]` (sztywna pojedyncza spacja gubi trafienia; `AS` =
postać żeńska, wyżej). Forma trzecioosobowa drużyny też
dotyczy szczurów (`[   ZABILA   ]  Gwenn zabila brudnego smierdzacego szczura.`).
Szum do odfiltrowania: plotki NPC (`mowi: A wczoraj to... zabil`), opisy lokacji
(`kosciotrup`, trupy), nazwy własne (Trupa Trupi Trup), **fałszywe dopasowanie
regexu klienta** `Szerokie drzwi wejsciowe do rezydencji ktos zabil solidnie
kilkoma deskami.` (korpus 3× — klient sam to oznaczył prefixem, bez licznika;
twardy filtr: biernik narzędzi + „zabójca" będący opisem obiektu). Linie walki
otoczenia noszą tagi `[1/6]`, `[par]`, `[unk]`. Zabójstwa szczurów (exp + bounty u
szczurolapów) bez specjalnego traktowania — normalne wpisy statystyk; kasa za ciała
to osobne zdarzenie bounty (§2.4).
**Tagi gildii w liniach zabójstw (kod + korpus, 2026-09-18):** nazwy zabójców bywają
obwieszone tagami klienta — korpus: `Spiczastouchy dlugonogi elf (Aynne GP)
(Crevan ES) zabil brudnego brunatnego kobolda.` Parser ścina `(<imię> <KOD>)`
z nazw zabójców i ofiar. Pełny alfabet 22 kodów (peopleGuilds.ts): CKN, ES, SC, KS,
KM, OS, OHM, SGW, BK, WKS, LE, KG, KGKS, MC, OK, RA, GL, ZT, ZS, ZH, NPC, GP.
**Normalizacja nazw mobów (upstream):** klucz = ostatnie słowo małą literą
(zostaje w dopełniaczu: „szczura", „wilka") albo dwuwyrazowy wyjątek z listy 26
(identycznej w Dargoth i Mudlecie) albo nazwa własna (1 słowo, wielka litera);
tag przy nazwie ofiary zanieczyszcza klucz upstream — pelny korpus potwierdza
N=0 (0/1796 linii zabojstw z tagiem `(NPC)` przy OFIERZE; tagi wystepuja
wylacznie przy zabojcy) — Kronikarz ścina tagi przed normalizacją prewencyjnie.

| Premium live | eventy API `kill` {killer: ME/TEAM/OTHER} i `enemyKilled` {objNum, killer, hasBody}; eventBus `zabici.updated` (sesja) i `zabici2.updated` (lifetime); aliasy klienta: `/zabici`, `/zabiciw`, `/zabici2`, `/zabici2 [rrrr/m/d]`, `/zabici2w`, `/zabici2!`, `/zabici_reset` | kod (kill.ts, plugin-types) |
| Premium historia | characterStorage: `kill_counter` (lifetime), `kill_counter_session`, `kill_counter_team` (per gracz); IndexedDB `ArkadiaKillsDB` — rekord per postać+mob+data, zapis **tylko własnych** zabójstw, odczyt per data/grupowanie/statystyki globalne, import rekordów | kod (kill.ts, killLifetimeStorage.ts) |

### 2.6 Postępy

| Zdarzenie | Detekcja | Weryfikacja |
|---|---|---|
| Wbicie postępu (live) | GMCP `char.state.improve` (0–15; 16 stanów) | kod (klient improveCounter, tjurczyk gmcp_handler_improvement, Towarzysz) |
| Linia tekstowa (sesyjna) | `Poczynil(?:es|as) (.*) postepy, od momentu kiedy .* gry\.$` + wariant bez „momentu": `..., od kiedy wszedles do gry.` + zerowy `Nie poczynil(?:es|as) zadnych postepow...` | kod + korpus (197 wyst. przy 9 echach — **linia jest spontanicznym pushem gry**, nie tylko odpowiedzią na komendę) |
| Linia eksploracji | `Masz wrazenie, iz ostatnimi czasy poczynil(?:es|as) (.*) postepy w poznawaniu swiata.` (+ wariant zerowy) | korpus (155 wyst.) |
| Linia nauki | `Wydaje ci sie, ze poczynil(?:es|as) (.*) postepy w nauce.` | korpus (18 wyst.) |
| Skala | 16 poziomów (0–15): minimalne, nieznaczne, bardzo małe, małe, nieduże, zadowalające, spore, dość duże, znaczne, duże, bardzo duże, ogromne, imponujące, wspaniałe, gigantyczne, niebotyczne = 1:1 `IMPROVE_STATES` klienta | kod (Dargoth improveCounter + Mudlet improve/progress) + wiki + korpus (wszystkie 16 gradacji obecne) — **errata 2026-09-18**: wcześniejsza tabela miała 2 zamiany (7↔8 i 12–15), poprawiona wg 4 zgodnych źródeł |

Linia tekstowa pojawia się samoistnie (push) — **pełny backfill historii postępów
z samych logów jest możliwy**, bez zależności od ech `→ postepy`. Na żywo służy jako
koroboracja GMCP. Licznik postępów resetuje się przy wylogowaniu — naturalnie sesyjny.

Mechanika poziomów (kod Dargotha improveCounter + Mudlet + wiki „Doświadczenie"):

- **Poziom 0** = „zadne/mini" (teksty `zadnych` i `minimalne` oba mapują na 0);
  liczniki go ignorują — to nie jest zdarzenie kroniki.
- **Skok delta > 1**: klient emituje każdy poziom pośredni osobno (pętla `record`
  per poziom) — kronika robi tak samo: jeden wpis per poziom, każdy z własnym
  timestampem, nigdy zbiorczy „skok +N".
- **Reconnect / absorb**: rozróżnienie per `object_num` — `recordInitial` (cichy
  catch-up po reconnect lub nowej sesji, bez duplikowania wpisów) vs `record`
  (zdarzenie); spadek poziomu przy fresh login = absorb między sesjami (reset
  poziomu bazowego). Spadek w trakcie sesji klient ignoruje → otwarte 14–15 (§10).
- **Postępy przy niepełnej formie** liczone osobno (`optionsForm === 1 &&
  stateForm < 3`, licznik `noFormCount`) — adnotacja w wpisie.
- **Prezentacja**: 15 postępów = 1 „niebotyczne" (`N niebotycznych + <stan>`) —
  dotyczy wyłącznie widoku statystyk, nie formatu wpisu kroniki.
- **Semantyka wiki**: 1 poziom = ułamek **całkowitego** doświadczenia postaci —
  czas między wbiciami NIE jest porównywalny między postaciami o różnej sile;
  kronika nie traktuje go jako metryki tempa.
- **Linia kliencka wbicia**: klient drukuje własny komunikat (tab +
  `Wlasnie wbiles postepy: <stan> (czas: m:ss)`) — to NIE jest linia gry, ale
  **println klienta jest logowany** (pelny korpus: 273 wyst. w 56 sesjach) —
  backfill postepow zyskuje dokladne timestampy wbic; MUD nie drukuje wlasnej
  linii wbicia (kanal czysto GMCP; towarzyszy jej linia sesyjna `Poczyniles
  <stan> postepy...`). Korpusowy dowod emisji poziomow posrednich (delta > 1):
  sekwencje `gigantyczne (czas: 24:52)` → `niebotyczne (czas: 0:00)` — kazdy
  poziom osobno, zgodnie z mechanika `record` w petli (wyzej).
- **Kontekst wpisu — decyzja 2026-09-18**: wpis prosty (stan + timestamp);
  czas od poprzedniego wbicia (m:ss) i snapshot zabójstw (my+team) trafiają do
  **metadanych audytowych** zdarzenia i zasilają weryfikator krzyżowy (§11).
- Technikalia klienta (referencja): aliasy `/postepy*` (10), storage
  `improve_counter` / `improve_counter_lifetime` (per `rrrr/m/d`), eventBus
  `postepy.updated` / `postepy2.updated`.

### 2.7 Cechy

Klient przechwytuje komendę `cechy` i parsuje odczyt; Kronikarz używa tych samych
zweryfikowanych patternów (klient: lvlCalc.ts; Mudlet lvl_calc.lua — tablice 1:1):

- `Jestes <opis> i <ile> ci brakuje, zebys mogla? wyzej ocenic sw(a|oj) <cecze>.` z opcjonalnym suffiksem modyfikatora `( +N )`,
- `Twoja/Twoj <cecha> osiagnela/al nadludzki poziom.`,
- linia zamykająca `Obecnie do waznych cech zaliczasz...` z **opcjonalnym sufiksem** ` Mozesz to zmienic podczas medytacji w gildii podrozniczej.` (oba warianty w korpusie),
- `Twoje cechy sa oslabione po ostatniej smierci.` (snapshot oznaczany jako osłabiony; grace 500 ms po linii zamykającej — kod).

**Tablice kanoniczne** (kod ×2 + wiki „Cechy"):

- 5 cech × 9 poziomów + `nadludzki` = 10; anomalia rozstrzygnieta (pelny korpus):
  `tchorzliwy` 10× / `thorzliwy` 0× — **forma wiki jest kanoniczna** (gra ja
  drukuje), forma kodowa `thorzliwy` zostaje w parserze zapasowo;
- kroki do następnego poziomu: bardzo duzo=0, duzo=1, troche=2, niewiele=3,
  bardzo niewiele=4; suma cechy = (poziom−1)×5 + krok;
- progi poziomu postaci: LEVEL_THRESHOLDS 58–190 (12 progów, 13 etykiet od
  „ktos niedoswiadczony" do „osoba owiana legenda"); rzeczownik `intelekt`
  mapowany na inteligencję;
- komenda `poziomy` (pomoc gry) = referencja opisów poziomów — walidator skal;
  output korpusowy: 35 list opisow poziomow (np. `→ poziomy sily`) + kompletna
  skala cech zrodlem gry (sila: slabiutki, watly, slaby, krzepki, silny, mocny,
  potezny, mocarny, epicko silny) — tablice kanoniczne potwierdzone Z GRY,
  nie tylko z kodu klienta.

Odczyt z modyfikatorem (sprzęt/zioła) jest odrzucany — nie zapisuje się fałszywej
wartości; gate: `gmcp.char.options.state_modifiers===1` (wiki „Opcje": `opcje
modyfikatory wlacz`). Detekcja odczytu: event `command` = `cechy` + własny parsing
linii; komenda `cechy` jest **bezargumentowa** (pomoc gry) — subkomenda `cechy um`
nie istnieje (echo w korpusie = literówka gracza, §10: 39). Linia `Twoj aktualny
poziom to ...` w logach = generowana przez klienta (calculateLvl), nie przez grę
— **ERRATA (pelny korpus): linia WYSTEPUJE w logach wielokrotnie** (println
klienta jest logowany razem z liniami gry; wniosek bez zmian: to nie jest linia
gry; wczesniejsze digesty korpusu mylnie raportowaly N=0 — §10: 58).
Premium: kanał live `cechy.read` (snapshot: odczyty/suma/poziom/osłabienie) +
storage klienta `cechy_history` (MAX 500 wpisów, tylko zmiany, null per cecha
zmodyfikowana, flaga estimated; pole `postepy` = lifetime postępów przy odczycie,
czyli koszt zmiany cechy wyrażony w postępach). Rozszerzona linia osłabienia
(pelny korpus, 116 wyst.): `Twoje cechy sa oslabione po ostatniej smierci. By je
odbudowac potrzebujesz zdobyc jeszcze <gradacja> postepy.` — gradacje = skala
6-stopniowa z §2.8 (wszystkie 6 obecne w korpusie), NIE skala 16-stopniowa.
Backfill: echo `→ cechy` + odczyt w logach. **Uwaga korpusowa:** logi HTML zawierają
linie cech w wersji **wzbogaconej przez klienta** — `[18] Jestes krzepki [4/10] i
niewiele [3/5] ci brakuje, zebys mogl wyzej ocenic swa sile.` — parser backfillu musi
tolerować prefiks `[N]` i wstawki `[x/y]` (bonus: wartości liczbowe dostępne wprost).
Po smierci klient dokleja przy prefiksie `[N]` wstawke `(±N)` = delta sumy cech
od poprzedniego odczytu (korpus: `[19] (-4) Jestes krzepki ...`, `[23] (+1) ...`)
— metadane gratis, parser toleruje; po smierci klient loguje tez wlasne paski
cech `[---- ... ----]` i znacznik `[Szczescie wroci?]` (rozpoznawane i pomijane).

### 2.8 Śmierć

| Zdarzenie | Detekcja | Weryfikacja |
|---|---|---|
| Śmierć własna | `^Umierasz\.$` (następna linia `Oddalasz sie.` to odejście duszy — ignorowana); przyczyna bywa środowiskowa, nie tylko walka (korpus: upadek — `Odpadasz od sciany i lecisz w dol...`) | kod (Towarzysz DEATH_PATTERNS) + korpus |
| Osłabienie po śmierci | `Twoje cechy sa oslabione po ostatniej smierci\.` (pelna forma rozszerzona: `... By je odbudowac potrzebujesz zdobyc jeszcze <gradacja> postepy.`) + **6 gradacji** wymaganych postępów: minimalne / bardzo małe / nieduże / nieznaczne / małe / zadowalające — **pelny korpus potwierdza tabele 1:1** (116+ wyst. linii rozszerzonej — counter jednoslowowy: minimalne 55, nieduze 25, nieznaczne 22, male 12, zadowalajace 2; dwuslowowa „bardzo male" dodatkowo 14+ w blokach — wszystkie 6 gradacji obecne); 11 smierci z pelnym blokiem posmiertnym (sekwencja zaswiatow = flavor, nie ksiega; `Jestes ledwo zywy` po odrodzeniu) | kod (klient: afterDeathProgress, lvlCalc) + korpus (143 odczyty przy 10 śmierciach + pelny korpus: 116 wyst.) |
| Śmierć członka drużyny | **„kto" rozstrzygnięte kodem**: diff `objects.nums` (przed→po) + akumulowane `objects.data` (id → {desc, team, hp}) identyfikuje ubyłego drużynowego w 100% — oba klienty tak akumulują (Dargoth `accumulatedObjectsData`, Mudlet `ateam.objs`); **„dlaczego ubył" w capture** (otwarte 18): dyskryminator śmierci vs wyjście/quit/teleport = brak linii odejścia + pojawienie się ciała w pokoju (wiki „Śmierć": ciało zostaje z dobytkiem, duch w zaświaty na kilka minut, odrodzenie wg opcji); forma ciała gracza, zachowanie `hp` i powrót ducha (ten sam id?) — nieznane, 1 obserwacja live; errata: flaga `living` NIE jest flagą śmierci (spec t=740: „istota żywa", zawsze true); linia tekstowa śmierci osób: korpus N=0 (576 sesji, zero śmierci drużynowych) | spec GMCP + korpus + kod ×2 + wiki „Śmierć" — zdarzenie live-only, brak backfillu; **zasięg: tylko ta sama lokacja** (GMCP milczy o innych pokojach — śmierć podzielonej drużyny poza zasięgiem, jawne ograniczenie); widok statystyk: „Zmarli czlonkowie druzyny" (decyzja 2026-09-18) |

### 2.9 Poczta (listy)

Listy = zdalna komunikacja gracz-gracz (bezplatna), odrebna od paczek
kurierskich (§2.1); wysylka/odbior na poczcie albo przez zwierze pocztowe
(`wyslij zwierze`, 15 gatunkow — flavor). Klientami z modulem pocztowym sa
Dargoth i tjurczyk/arkadia (fork Mudleta: mail_creator + triggery);
upstream Delwinga arkadia-skrypty: 0 trafien, brak modulu.

| Zdarzenie | Detekcja | Weryfikacja |
|---|---|---|
| Nowy list | `^Masz nowa poczte od [A-Za-z]+\.$` — nadawca w **dopelniaczu** (korpus: 21 unikalnych, m.in. Eldura, Tahiry, Yoany, Gvidona, Kalkofoniksa, Ulika, Ulvhedina, Selene, Kruxa); w logach HTML linia występuje z **prefiksem klienta** `[ POCZTA ] ` (println newMail.ts jest logowany; **padding wariantowy**: Dargoth `[ POCZTA ] `, tjurczyk `[  POCZTA   ] ` — regex `^\[\s*POCZTA\s*\]`, regula jak przy `[ ZABIL* ]`, §10: 9) — parser backfillu akceptuje opcjonalny prefiks; koroboracja premium live: GMCP `Mail.State.unread` | kod (newMail.ts pattern 1:1) + korpus (21 nadawcow) + spec GMCP |
| Wyslany list | potwierdzenie wysylki z edytora: regex tjurczyka `zostal.* pomyslnie wyslan` (**dokladna forma capture** — korpus N=0, otwarte 23); towarzyszaca linia po `**`: `List jest gotowy do wyslania.`; koroboracja: znikniecie GMCP `unsent`; wpis prosty {timestamp} (decyzja §11) | kod (tjurczyk Arkadia.xml) + korpus (N=0) |
| Sygnaly tekstowe statusu | login: `Czeka na ciebie nieprzeczytana poczta` (tekstowy odpowiednik GMCP `unread`; trigger tjurczyka masz_poczte_login) i `^Wysylasz .* na poczte.$` (potwierdzenie `wyslij zwierze` — kontekst odbioru) — **metadane/kontekst, nie wpisy**; korpus N=0 | kod (tjurczyk Arkadia.xml) |
| Status skrzynek (GMCP) | `Mail.State` {unread, unreceived, unsent} — booleany, push automatyczny przy logowaniu i przy kazdej zmianie (otrzymanie/czytanie/wysylka); **metadane sesji live-only, nie wpis kroniki**; badge klienta pokazuje tylko unreceived („Nowa") i unsent („Niewyslana"), unread ignorowany (decyzja UI Dargotha, potwierdzona e2e); klik badge = `wyslij zwierze`; opcja `Char.Options.mail_hidden`; typy Gmcp_msgs: `mail`, `editor.mail`, `notification.mail` | spec GMCP forum t=740 + kod (MailStatus.tsx, e2e mail-status.spec.ts) |
| Migawka indeksu `listy` | naglowek `^Listy (nieprzeczytane\|odebrane\|wyslane\|niewyslane)(?: \([^)]*\))?:$` — warianty skrotu: `(prezentowane jest pierwsze 20)` / `(prezentowane jest pierwszych 50)` / `(prezentowanych jest pierwszych 50)`; wpis `N. [*R* ]Temat: <temat>` (`*R*` = przeczytany; numeracja dowolna, nie od 1; ≤50 najnowszych) + druga linia `Nadawca: <imie>` (skrzynki odbiorcze) albo `Odbiorcy:`/`Odbiorca:` (wyslane/niewyslane) z **data IG w drugiej kolumnie** po 2+ spacjach (`Pn, 31 VIII 2026`); dlugie listy odbiorcow zawijane (kontynuacja z wcieciem); korpus: **suffix `(forwardowany)` po temacie** = list doslany (`doslij`); **metadane audytowe, nie wpis** — indeks z popupa klient UKRYWA (trigger zwraca null), w logach tylko z komend manualnych | kod (poczta.ts + test byte-for-byte z sesji live) + korpus (wpisy 4×, forward 1×) |
| Czytanie listu | naglowki `List : N`, `Od   :`, `Temat:`, `Do   :` (opcjonalnie), `DW   :` (opcjonalnie), `Data : <Pn, 31 VIII 2026, 21:16:12>` — pola zawijane (kontynuacja wciecie 2+); tresc dowolna (takze ASCII-art pergaminu); koniec = pager `^\[<zakres> <klawisze>\] \(aktualny: N\) -- $`; negatyw `Nie ma takiego listu.`; **kontekst/zalacznik do zdarzenia, nie wpis** — popup ukrywa jak indeks | kod (poczta.ts + test byte-for-byte) + korpus (tresc raportu kurierskiego) |
| List-raport dostawy paczki | temat `opis doreczenia przesylki` (indeks/list) + tresc `Dnia <n>. pory <miesiac IG>, wedlug rachuby czasu Starszego Ludu paczka zostala doreczona do adresata - <Imie>...` — **weryfikator krzyzowy z §2.1** (koroborant dostawy paczki, nie osobne zdarzenie); drugi temat korpusowy: `przesylka kurierska - PILNE` | korpus (wpis indeksu + tresc) |
| Negatywy | `Nie masz zadnych (nieprzeczytanych \|odebranych \|wyslanych \|niewyslanych )?listow.` (korpus: wariant nieprzeczytanych ×2 — komenda `poczta` / `listy nieprzeczytane`) i `Nie ma takiego listu.` — filtr, nie zdarzenie | kod (emptyPattern) + korpus |

Komenda `listy` (pomoc gry `?listy` = strona „Poczta"; `?poczta` nie istnieje
— 404): 4 skrzynki, ≤50 najnowszych wg daty wyslania, filtry `do`/`od <kogo>`,
`dzis`/`wczoraj`/`przedwczoraj`, `dnia`/`przed`/`po <data>`, `o temacie
<fragment>` (argumenty laczone, temat ostatni); pozostale komendy: `napisz
list` (edytor: `**` wysyla, `~q` porzuca, `~l` lista, `~dw`/`~udw` DW),
`przeczytaj list <n>` (alias `list <n>`), `doslij`, `aliasy pocztowe`,
`poczta` (status nieodebranych/nieprzeczytanych).

**Rozbieznosc nazewnicza**: pomoc gry i wiki dokumentuja skrzynke `otrzymane`,
gra drukuje `odebrane`/`odebranych` (naglowek indeksu + negatyw — test klienta
byte-for-byte z sesji live); klient wysyla `listy odebrane` i dostaje indeks
— parser naglowka i negatywu akceptuje prewencyjnie OBIE formy.

Edytor i wysylka: `napisz list` otwiera konwersacje **adresat -> temat ->
DW** (teksty promptow nieznane — oba klienty odpowiadaja pozycyjnie, na
slepo; capture), potem prompt edytora `Wpisz ~?, zeby uzyskac pomoc, lub
**, by zakonczyc edycje.` (**zgodny w dwoch klientach**: Dargoth
PROMPT_PATTERN, tjurczyk tempTrigger); w edytorze: `~udw <imie>` per ukryty
odbiorca, `**` wysyla, `~q` porzuca (wiki: tez `~l`, `~dw`); po `**` linia
`List jest gotowy do wyslania.`, a po faktycznej wysylce potwierdzenie
`zostal.* pomyslnie wyslan` (oba z kodu tjurczyka, korpus N=0 — otwarte 23);
znikniecie flagi GMCP `unsent` to koroborant wysylki.

Szablony klientow: Dargoth (none/plain/parchment/parchment2/parchment3/raw,
szerokosc konfigurowalna 20-120) i tjurczyk (plain/plain_border/parchment
x3, sztywne 55) — **ta sama rodzina ASCII-art** (naglowek pergaminu Dargotha
= art z listu w tescie poczta.test.ts); tresci listow bywaja preformatowane
przez klienty — parser tresci bierny (dowolne linie). Linia kliencka
podgladu `Podglad listu (szerokosc N, szablon <bez szablonu|Ramka|
pergamin|pergamin 2|pergamin 3|bez formatowania>)` (+ `(brak tresci)`) —
println logowany, backfill pomija (zasada §11). Alias `/list` jest kliencki
w obu klientach (Dargoth: kompozytor; tjurczyk: tez `/list <kto> <tresc>`
= szybki list) — echo `-> /list` bez odbicia w grze.

### 2.10 Apokalipsa i czas IG

Patterny przeniesione z toolkitu (moduły `apokalipsa`, `analizator_czasu`), działające
na echu komendy `→ system` / `→ czas` + oknie odpowiedzi:

- `Swiat odrodzil sie : <dzień tygodnia>, <dzien> <miesiąc rzymski> <rok>, <hh:mm:ss>`
  (korpus: `Swiat odrodzil sie : Pt, 19 XII 2025, 07:27:33` — **z dniem tygodnia**,
  czego nie łapał regex toolkitu) — apokalipsa jako zdarzenie kroniki i twarda granica
  kontekstu sesji w backfillu;
- `Swiat istnieje : ...` — uptime z pełną odmianą (dzień/dni, godzina/godziny/godzin,
  minuta/minuty/minut, sekunda/sekundy/sekund) i wariantem bez dni (`6 godzin 30 minut
  8 sekund`); `<n>% swiata zostalo opanowane` — ciemność;
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
- Drużyna: przekazanie prowadzenia `[ DRUZYNA ] <ktoś> przekazuje ci prowadzenie
  druzyny.` (prefix klienta; korpus: 25 wyst.) + eventy `teamChange`.

### 2.12 Wiedza

| Zdarzenie | Detekcja | Weryfikacja |
|---|---|---|
| Wzrost wiedzy (tick) | `Wydaje ci sie, ze twoja wiedza o <dziedzina> wzrosla nieznacznie\.` — gradacja zawsze „nieznacznie" (wiki: przekroczenie minipoziomu co 1%, 100 minipoziomow = pelna wiedza); wildcard z kodu Dargotha (`wzrosla .*`) jako fallback | wiki „Wiedza" + kod (Dargoth knowledge.ts KNOWLEDGE_TICK_PATTERN; Mudlet knowledge.lua: brak parsowania ticka) + korpus (8 dziedzin, wszystkie „nieznacznie") |
| Pozyskanie fragmentu wiedzy | `Dowiadujesz sie czegos wiecej o <dziedzina>\.` — **pelny korpus: 266 wyst. w 68 sesjach** — promocja z capture na pelne zdarzenie kroniki (decyzja, §11); linia push gry, czesto w parze z linia czynu `Widziales/Ogladales/Sluchales opowiesci o/Analizowales <cos>.` (tez push; korpus: `Widziales kobolda.`, `Ogladales smocza luske.`, `Analizowales szczatki wippera.`); gated opcja gry WIEDZA (wiki „Opcje") | wiki „Wiedza" + korpus (266 wyst.) |
| Pelna wiedza w dziedzinie | brak osobnej linii — rozstrzygalne z migawki komendy `wiedza o <dziedzinie>` (poziom `pelna`) lub z odczytu tytulu; tytul „Znawca <dziedziny>"; wariant korpusowy na who: „Znawca Wiedzy Wszelakiej" | wiki „Wiedza" + korpus |

**14 dziedzin kanonicznych** (kod Dargoth `knowledgeCategories.ts` = Mudlet
`knowledge.lua`, 1:1): Chaos i jego twory, goblinoidy, golemy, istoty demoniczne,
jaszczuroludzie, magia i jej twory, nieumarli, pajaki i pajakowate, ryboludzie,
smoki i smokowate, starsze rasy, stwory pokoniunkcyjne, szczuroludzie, wampiry.
Uwaga: lista wiki „Wiedza" pisze „istoty magiczne" — gra/korpus/kod: `magii i jej
tworach` (korpus 6 wyst.); wiki nadrzedne semantycznie, kod/korpus nadrzedne
tekstowo.

**Zrodla wiedzy per dziedzina** (trop Delwinga potwierdzony): trzy typy —
`walki` / `ksiazek i bibliotek` / `eksploracji` (Dargoth KNOWLEDGE_TYPE_IDENTIFIERS
= wiki „Rozwijanie wiedzy"). Bazy zrodlowe:
- **ksiazki**: `tjurczyk/arkadia-data/master/knowledge_data.json` (repo danych
  Delwinga, version 1) — 44 ksiazki z pelna odmiana (mianownik/dopelniacz/
  biernik/mnoga) i lista dziedzin per ksiazka;
- **biblioteki**: ten sam JSON — 44 biblioteki z `location_id` (mapa), nazwa
  i lista dziedzin per biblioteka; pokrycie 14/14 dziedzin w obu zbiorach;
- **eksploracja**: API ethel.pl `wp-admin/admin-ajax.php?action=wiedza_data`
  (to z niego korzysta Dargoth wiedzaStore) — 14 list wpisow eksploracyjnych
  (teksty `Byles/Ogladales/Dowiedziales...`), kolejnosc list = kolejnosc
  KNOWLEDGE_CATEGORY_CONFIG (mapowanie indeksowe, bez nazw w payloadzie); API
  NIE niesie typu zrodla — Dargoth przypisuje wszystkie wpisy do `exploration`.
  Wiki: „ponad szescset" zdarzen eksploracyjnych.

**Poziomy stanu wiedzy** (migawka `wiedza`): 10 gradacji gry — znikoma, niewielka,
czesciowa, niezla, dosc dobra, dobra, bardzo dobra, doskonala, prawie pelna,
pelna (wiki + Mudlet knowledge_desc [1/10..10/10]); Dargoth dodaje `brak` na
indeksie 0 (11 etykiet). Migawka rozbija dziedzine na 3 typy; linia ticka NIE
mowi ktorego typu dotyczy — wpis kroniki = dziedzina + timestamp, typ nieznany.

**Filtr szumu `porownaj` (pelny korpus):** linia `Wydaje ci sie, ze jestes <...>
niz <kto>.` (komenda `porownaj`, z doklejkami klienta `(±N)`) dzieli prefiks
`Wydaje ci sie, ze` z tickiem wiedzy i linia nauki (§2.6) — twardy filtr:
kotwica ticka na pelnej frazie `twoja wiedza o ... wzrosla`, nigdy na samym
prefiksie.

Wpis kroniki: `12:03:44 — Wzrost wiedzy: goblinoidy.` Backfill mozliwy z logow
(linia ticka jest pushem gry); koroboracja z migawki komendy `wiedza` (poziomy
per dziedzina x typ). Decyzja 2026-09-18: wzrost wiedzy = zdarzenie kroniki (§11).

### 2.13 Umiejętności

| Zdarzenie | Detekcja | Weryfikacja |
|---|---|---|
| Zmiana poziomu umiejętności | **brak linii push w grze** — zmiana rozstrzygana diffem kolejnych migawek komendy `um` / `umiejetnosci [typ]` (model `cechy_history`, §2.7); wpis = nazwa + stary→nowy poziom + timestamp | kod (Dargoth skills.ts, skillTable.ts; Mudlet skill.lua) + wiki „Umiejętności" + pomoc `umiejetnosci` + próbka logów HTML (um ×4) |

**Skala kanoniczna** (3 źródła 1:1 — Dargoth skillsDesc, Mudlet skills_desc,
wiki): 1 ledwo, 2 troche, 3 pobieznie, 4 zadowalajaco, 5 niezle, 6 dobrze,
7 znakomicie, 8 doskonale, 9 perfekcyjnie, 10 mistrzowsko.

**Typy** (wiki „Umiejętności"): ogólne (11), bojowe (16), magiczne (3),
złodziejskie (6), praktyczne, językowe, specjalne. Komendy: `um`,
`umiejetnosci [typ]`, `maksymalne [typ]` (pomoc gry). Pelny korpus: echa
`→ umiejetnosci [typ]` N=0 (gracz uzywa golego `um`, 623 wyst.) — filtry
typow zostaja z pomocy gry (kod), bez potwierdzenia korpusowego; output
`maksymalne` potwierdzony: surowa tabela 38 pozycji BEZ wstawek `[N/10]`
(w przeciwienstwie do tabeli `um`).

Modyfikacje klienta w logach: tabela `um` przepisana z wstawkami `[N/10]`
(skillTable) — parser backfillu toleruje i odcina; wiersz tabeli rozpoznawany
po słowniku poziomów (isSkillRow), obce linie w ramach tabeli przepuszczane.
Modyfikator na wierszu `um` (suffix jak w cechach) — pelny korpus N=0
(576 sesji): zostaje wylacznie z kodu, parser przygotowany. Mechanika
diff-migawki potwierdzona korpusem: zmiana `topory: ledwo → troche` miedzy
kolejnymi migawkami `um` w odstepie 5 minut (brak jakiejkolwiek linii push).

**Koszt treningu** (decyzja 2026-09-18, §11; mechanika z pelnego korpusu):
trening u mistrzów zawodu księgowany jako wydatek „usługa/trening" —
wyłącznie łączna kwota wydana na treningi. Sygnatury (korpus):
`→ trenuj` (bez argumentu) drukuje **cennik**: `Oto umiejetnosci, w jakich
mozesz sie szkolic:` + tabela dwukolumnowa `Umiejetnosc: / Koszt sesji
treningowej:` z kwotami slownie per umiejetnosc (`walka dwiema bronmi
5 srebrnych i 1 miedziana moneta`); sesja treningowa `→ trenuj <um>` →
`Przechodzisz szkolenie w <opis>.` — **gra NIE drukuje linii kasowej przy
pobraniu oplaty** (zero linii kasowej w 10 echach treningowych w korpusie);
limit mistrza: `Obawiam sie, ze w tej dziedzinie nie naucze cie juz nic
nowego. Moze jednak uda ci sie znalezc innego mistrza...` Księgowanie:
koszt = liczba sesji `Przechodzisz szkolenie ...` × stawka z ostatniego
znanego cennika tej lokacji (stawki per um dostepne jako metadane; suma
łączna wg decyzji); sam trening jako kategoria kroniki pozostaje poza
zakresem; trening meczy (`Jestes bardzo zmeczony.`).

### 2.14 Języki

| Zdarzenie | Detekcja | Weryfikacja |
|---|---|---|
| Zmiana poziomu języka | jak umiejętności: diff migawek komendy `jezyki` (brak push); wpis = język + stary→nowy poziom + timestamp | kod (Dargoth languageSkills.ts) + wiki „Języki" + pomoc `jezyki` + próbka logów HTML (jezyki ×4) |

**Skala** = skala wiedzy (10 gradacji: znikoma, niewielka, czesciowa, niezla,
dosc dobra, dobra, bardzo dobra, doskonala, prawie pelna, pelna — Dargoth
languageLevels = Mudlet knowledge_desc = wiki „Wiedza").

**20 języków** (wiki „Języki"): 14 Imperium + 6 Ishtar; nauka od innych graczy
lub z pochodzenia postaci; Mroczna Mowa = mutacja. Linia negatywna komendy
`jezyki`: `Nie znasz zadnych jezykow obcych.` (pelny korpus: 297 wyst.).
Komenda `jezyki maksymalne` — output korpusowy: wiersze `<jezyk>: <poziom>`
BEZ paskow (`starsza mowa: dobra`, `tileanski: pelna`, `reikspiel: pelna`);
klient trzyma poziomy maksymalne
w storage `language_max_levels` (koroboracja premium). Pomoc gry
(`pomoc jezyki` → `Dostepne jezyki: ...`) potwierdza liste 14 jezykow
Imperium zrodlem gry.

Modyfikacje klienta w logach: wiersz tabeli `jezyki` ma forme
`<jezyk>:  <poziom>  [#---------]` — pasek klienta (gauge) o **zmiennej
dlugosci skalowanej do poziomu maksymalnego** jezyka (korpus: `[#---------]`
przy max pelna, `[#-----]` przy max dobra) — parser backfillu kotwiczy na
poziomie slownym i odcina wszystko po nim, nigdy nie parsuje paska.

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

**Echo komend** (`→ czas`, `→ system`, `→ postepy`, `→ cechy`, `→ depozyt`,
`→ zdenominuj`, `→ wybierz paczke N`, `→ oddaj paczke`, `→ wloz/wez monety ...`)
otwiera okna atrybucji w backfillu: migawki postępów, historia cech, historia banków,
apokalipsy, kotwice czasu IG, maszyna paczek, markery intencji transferu monet.
Pole `type` wpisu — zbiór wartości **potwierdzony** na eksporcie JSON klienta
(LogExportData v1, 1892 wpisy, 2026-09-17): `script`, `trigger-echo`, `echo`, `mud`,
`other`, `prompt`, `info`, `room.short`, `room.exits`, `room.contents.living`,
`comm`. Umożliwia **strukturalny backfill** nazw pokoi (`room.short`), NPC w pokoju
(`room.contents.living`) i mowy NPC (`comm`) zamiast heurystyk; większa próbka
potwierdzi kompletność zbioru.

**Modyfikacje klienta w logach (korpus):** zalogowana linia bywa wersją **przepisaną
przez klienta**, nie surowym tekstem gry — parser backfillu toleruje: `[ POCZTA ] `
(poczta), `[N] ` (licznik przy liniach cech i przybyciach), `[unk] ` (linie walki bez
rozpoznanego typu), wstawki `[x/y]` w liniach cech, doklejkę `, czyli X zl, Y sr,
Z mdz` (wycena, §2.2) oraz **własne tabele klienta** drukowane po liniach gry:
pretty-print depozytu (nagłówek `| DEPOZYT |`, wiersze kategorii, markery `*...*`,
liczności cyframi) i analogiczne tabele pojemników — redundantne wobec linii gry,
rozpoznawane i pomijane (detekcja po nagłówku ramki). W szeptach pomocy poczty komendy
są osadzone jako klikalne elementy i w spłaszczonym tekście znikają — nie traktować
takich linii jako dowodu braku komendy.

**Sklejanie zawiniętych linii (korpus):** gra zawija długie linie do szerokości
okna klienta — w logu jedna linia logiczna rozpada się na kilka wizualnych, łamana
w środku fraz, z wiszącymi przecinkami (dowód: `Twoj depozyt zawiera ... dwie
mithrylowe` ⏎ `monety,` ⏎ `czarna zdobiona ksiege` ⏎ `, piec` ⏎ `garsci ...`).
Parser backfillu **najpierw skleja wizualne linie wpisu, dopiero potem stosuje
regexy** — dotyczy wszystkich długich list (depozyt, pojemniki, wyceny stosu).

**Echo transferów monet:** `→ wloz monety do swojej sakiewki/plecaka`, `→ wez monety
ze swojej sakiewki/plecaka`, `→ wez <denominacja> monety z N. ciala` — transfery
między pojemnikami i looting monet z ciał; nie są przychodem/wydatkiem (loot z ciała
księguje się z linii `Bierzesz/Dostajesz`), ale są markerami kontekstu.

**Echo `→ wiedza` — migawka wiedzy postaci (łowisko v2):** komenda `wiedza` drukuje
kategorie wiedzy i wpisy z prefiksem `* ` — stworzenia widziane (`* Widziales
szczuroczleka.`), przedmioty oglądane (`* Ogladales amulet kultystow Rogatego
Szczura.`), czyny wykonane (`* Dostarczyles cialo skavena szczurolapowi z Nuln.`).
To **migawka na żądanie** (jak `postepy`/`cechy`), nie dziennik zdarzeń: liczniki
wystąpień wpisów = częstotliwość komendy, nie zdarzeń. Wpis-czyn to **jednorazowy
koroborant** („zdarzenie zaszło kiedyś przed tą migawką") — nigdy licznik ani
znacznik czasu wykonania. Pokrewne linie: `Wiedza o <kategorii>:` (nagłówek
kategorii) i `Wydaje ci sie, ze twoja wiedza o <kategorii> wzrosla ...` (wzrost
wiedzy — kandydat na zdarzenie postępu wiedzy, decyzja odłożona).

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
- **Oś czasu / dziennik** — chronologiczny strumień zdarzeń. **Filtry globalne
  (obowiązują we wszystkich zakładkach): zakres dat (od–do), typy zdarzeń, postać.**
  Każde zdarzenie ma timestamp (log-time z HTML w backfillu albo czas live), więc
  każde bez wyjątku jest przyporządkowane do czasu i filtrowalne — nic nie jest
  redukowane do gołego licznika.
- **Finanse** — bilans gotówka/banki, przychody/wydatki/transfery, wykresy (wszystkie
  kategorie, również pokrywające się z licznikami klienta — celowa pełność);
  denominacje kantorowe: statystyki łącznie, per kantor, per postać, per sesja.
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

## 10. Otwarte kwestie

Domknięte na korpusie 2026-09-16 (3,86 mln linii, 576 sesji HTML; dwa przeloty —
drugi z poprawionymi filtrami):
1. ~~Potwierdzenie formy linii zwrotu paczki~~ — **zero** `Zwracasz pocztowa paczke`
   w korpusie; status „zwrócona" zostaje w katalogu (pattern z kodu) jako ścieżka
   rzadka bez potwierdzenia korpusowego. (ERRATA korpus III: `po terminie` jednak
   występuje — patrz pkt 4 niżej.)
2. ~~Zabójstwa: próbki korpusowe~~ — domknięte drugim przelotem: klient przepisuje
   linię zabójstwa (`[ ZABILES ]` / `[ ZABIL ]` / `[ ZABILA ]` + suffix `(n / m)`),
   565 wystąpień, 261 unikalnych próbek; surowa forma w logach nie występuje.
3. Weryfikacja korpusowa katalogu: paczki (z etykietami 1/2/3-członowymi
   i ` - PILNE!`), postępy, cechy, śmierci, bank, kantor, poczta, czas, zabici —
   potwierdzone (szczegóły w tabelach §2).

Domknięte na łowisku low_zlecen 2026-09-17 (korpus III, sekcje A–K, 576 sesji):
4. ~~Status paczki „spóźniona"~~ — potwierdzony: `<NPC> mowi do ciebie: Niestety,
   ale dostarczyles przesylke po terminie. Dlatego moge ci za nia zaplacic tylko
   tyle.` (4+ wystąpień, różni NPC) — wypłata pomniejszona (§2.1).
5. ~~Dokładna linia realizacji standardowego zlecenia~~ — `<NPC> odbiera od ciebie
   <towar> i wrecza ci <kwota>.`; dowód: oferta (mięso z zająca) → 41 s → dwie
   dostawy częściowe tego samego NPC; rozróżnione 3 warianty `odbiera od ciebie`
   (§2.4).
6. ~~Adler → Nuln~~ — potwierdzone korpusem (who-lista `Adler Winck, Nuln`, opis
   `(Adler NPC)`, pokój `Biuro szczurolapa.`); drugi szczurolap: Ratan; cennik:
   szczur 3 sr, mysz 8 pensów (§2.4).
7. ~~Zbiór wartości pola `type`~~ — wstępnie domknięte: 11 wartości z eksportu JSON
   klienta (LogExportData v1, 1892 wpisy) — strukturalny backfill pokoi/NPC/mowy
   (§6.2); większa próbka potwierdzi kompletność.

Domknięte na łowisku v2 2026-09-17 (sekcje B+, D+, E1/E10, konteksty):
8. ~~Surowa forma linii dawania ciał~~ — **linia nie istnieje**: zero `Dajesz/
   Oddajesz ... cial*` do NPC w korpusie; `daj` do NPC questowych drukuje wyłącznie
   echo komendy + linie NPC (§2.4 realizacja i bounty).
9. ~~Anomalia E1=0~~ — znacznik zabójstwa ma **dopełnienie spacjami** (`[  ZABILES  ]`,
   `[   ZABIL   ]`, `[   ZABILA   ]`); sztywna spacja w regexie gubiła 100% trafień;
   korpus ma 1796 linii `[ ZABIL* ]`, w tym 37 zabójstw szczurów (23 unikalne) i
   zabójstwa drużynowe (Gwenn).
10. ~~Pełna sekwencja realizacji~~ — echo `→ daj <towaru> <NPC>` → para {postęp
    `Dziekuje, potrzebuje jeszcze ...` + zapłata `odbiera od ciebie ... i wrecza
    ci ...`} ×N dostaw; jedna komenda = N dostaw (§2.4).
11. ~~System wpisów `* `~~ — to listing komendy `wiedza` (migawka wiedzy), nie
    dziennik questów; wpis-czyn = jednorazowy koroborant (§6.2).

Domknięte na pełnej analizie źródeł paczek 2026-09-17 (korpus III + kod klienta
PackageHelper/deliveryStats/npcStore + tjurczyk assistant.lua + wiki „Pocztylioni"):
12. ~~Zwrot paczki — status dowodowy~~ — linia wynikowa `Zwracasz pocztowa paczke`
    nadal N=0 w korpusie, ale mechanika potwierdzona wiki, a echo `→ zwroc paczke`
    występuje (4×); detekcja kotwiczy na echu, linia w trybie capture (§2.1).
13. ~~Czy backfill ofert działa na starych logach~~ — tak: tablice w logach są już
    zmodyfikowane przez klienta (kolumna `Dystans`, linie `> dystans: N`); regex
    niekotwiczony parsuje 12/12 ofert testowych (§2.1).
14. ~~Pokrycie adresatów w bazie NPC~~ — 111/111 adresatów-osób korpusu obecnych
    w zdalnej bazie npc.json (637 rekordów); 5 etykiet pocztowych mapowanych na
    lokacje poczty (§2.1).

Domknięte na pełnej analizie źródeł pieniędzy 2026-09-17 (Towarzysz coins/
polishNumbers/sources + klient coinColors/priceEvaluation/deposits/contracts +
toolkit denominacja + wiki „Pieniądze" + korpus; tjurczyk: brak modułu kasy):
15. ~~Wycena EQ — forma w logach~~ — `, czyli X zl, Y sr, Z mdz` to wstawka klienta
    (priceEvaluation), nie tekst gry; backfill odcina; + 2 warianty stosu z kodu
    (§2.2).
16. ~~Kwoty `wiele`/`kilka`~~ — liczebniki nieokreślone nieparsowalne: kwota null,
    księga nietknięta (żaden z trzech parserów nie robi tego poprawnie — §2.2
    reguła 6).
17. ~~Monety w pojemnikach własnych~~ — transfer gotówka↔pojemnik (jak bank), nie
    przychód/wydatek; linie wynikowe + migawka podglądu skatalogowane (§2.2).
18. ~~Prowizja denominacji~~ — per bank 3/5/8% (wiki), nie globalna 8% (§2.2).

Domknięte na analizie źródeł pieniędzy II 2026-09-17 (kod Dargotha: polishNumber-
Converter/carriage/stoneValue/contracts/smith/shop/bagManager/prettyContainers/
itemCollector/bilety + oficjalna specyfikacja GMCP forum t=740 + re-fetch wiki
„Pieniądze" z kolumną Depozyt + korpus-próbka 33 logi; wiki „LPC": brak treści
pieniężnych):
19. ~~Pokrycie tabeli prowizji~~ — luka domknięta: Zakon Sigmara 8% (wiki) =
    „Bank, Zamek Sigmara" (mapa, Averland); „Toscania" w tabeli wiki to literówka
    — gra/mapa: Toskania (nasze dane poprawne); kantor Eysenlaan spoza wiki —
    stawka nieznana → otwarte 4 (§2.2).
20. ~~Wynajem wozu + kaucja~~ — osobny typ zdarzenia: dwie kwoty (najem = wydatek,
    kaucja = depozyt zwrotny), pełna kaucja do 6h (kod carriage.ts); NIE przejazd
    (§2.2).
21. ~~Unia liczebników vs kod Dargotha~~ — konwerter Dargotha bez setek/tysięcy
    (pokrywa Towarzysz); formy `dwu`- i `jednego/jednej`-złożenia tylko w contracts;
    zbiorowe skatalogowane; typo-formy `piedziesiat/pieedziesieciu` → otwarte 5
    (§2.2 reguła 7).
22. ~~GMCP a gotówka~~ — definitywnie: oficjalna specyfikacja (forum t=740) nie ma
    modułu/pola pieniężnego; księga zawsze tekstowa (§2.2).

Nadal otwarte:
1. ~~Wzrost wiedzy jako osobne zdarzenie kroniki~~ — decyzja 2026-09-18: TAK
   (§2.12, §11); kanal, 14 dziedzin, bazy zrodlowe (ksiazki/biblioteki/eksploracja)
   i poziomy domkniete na wiki + kod x2 + korpus + JSON Delwinga + API ethel.pl.
2. ~~Zgłoszenie upstream do arkadia-mapa~~ — odrzucone decyzją gracza
   2026-09-18: suplement lokalny (klucz `suplement_depozyty`, pokój 10416
   Ard Skellig) wystarczy na stałe (§2.3, §5).
3. ~~Linia wynikowa komendy `sprawdz swoja reputacje`~~ — domkniete na pelnym
   korpusie: skala 6 gradacji odpowiedzi pracownika poczty (§2.1, §10: 46).
4. Stawka prowizji kantoru Eysenlaan — tabliczka nieznana (wiki milczy) — tryb
   capture; pelny korpus: nikt nie odwiedzil kantoru Eysenlaan w 576 sesjach
   (N=0 wizyt); wzorce tabliczki gotowe (forma slowna + forma z procentem
   cyfra, §2.2).
5. ~~Typo-formy `piedziesiat/pieedziesieciu` (contracts.ts)~~ — domkniete na
   pelnym korpusie: martwe klucze (N=0), gra spojna (4 warianty 50-tki);
   `piescdziesiat` = literowka gracza w mowie (§2.2 reguła 7, §10: 47).
6. Linia refundacji kaucji wozu — format nieznany z kodu (klient nie triggeruje);
   `Wynajmujesz` N=0 rowniez na pelnym korpusie 576 sesji — capture (potwierdzone
   N=0 jako dowod, nie brak danych).

Domknięte na analizie źródeł banków 2026-09-17 (Dargoth deposits.ts + pretty-
Containers parseItems + Mudlet boxes.lua + tjurczyk boxes.lua (ta sama rodzina;
Towarzysz: brak modułu) + wiki „Skrytki" i „Pieniądze" + korpus-próbka 33 logi
(realna sesja bankowa) + mapa/JSON (bindem 24 + suplement + anomalie) + GMCP):
23. ~~Koszty skrzynek depozytowych~~ — 50 zł podstawa + poziomy 2/5/10/20 mithryli,
    do końca gry postacią, limit 25 przedmiotów (stos = 1); wydatek „usługa
    bankowa" (§2.3); linia gry → otwarte 7.
24. ~~Pokrycie lokalizacji depozytów~~ — trzy źródła zgodne: valid_banks Mudleta
    16/16 = nasze dane (15 z bindem + skellige suplement), wiki 15/15, ponad to
    Brugge (10 pokoi) i Val'Kare; wiki „Skrytki" nieaktualna (13 miejsc) — dane
    mapy źródłem nadrzędnym (§2.3, §5).
25. ~~Zawijanie długich linii w logach~~ — linia depozytu łamana w środku fraz;
    reguła ogólna: sklejanie przed regexami (§6.2).
26. ~~Tabela DEPOZYT klienta w logach~~ — pretty-print po linii gry, redundantny;
    akapit „prefixy" rozszerzony do „modyfikacje klienta" (§6.2).
27. ~~Komendy `wplac`/`wyplac`/`przelej`~~ — gra je zna (walidator Dargotha +
    lista Mudleta), status rozstrzygnięty jako nieznany → otwarte 8 (§2.3).

Nadal otwarte (uzupełnienie):
7. Linia wykupienia/rozbudowy skrzynki depozytowej + wyjście `?depozyt` — format
   nieznany (żaden klient nie triggeruje); pelny korpus: N=0 potwierdzone —
   tryb capture.
8. ~~Status komend `wplac`/`wyplac`/`przelej`~~ — domkniete na pelnym korpusie:
    `wplac`/`wyplac` N=0 (martwe legacy po 2011, poza katalogiem); `przelej` =
    przelewanie plynow, kolizja nazwy — twardy filtr (§2.3, §10: 48).
9. ~~Korpusowe nazwy sal bankowych~~ — domkniete na pelnym korpusie: 18 nazw
    sal (dane referencyjne, klucz `korpus_pelny_2026_09_18`) (§2.3, §10: 49).
10. ~~`sto+` słownie w listach depozytu~~ — domkniete na pelnym korpusie: N=0
    w depozytach i liniach kasowych; unia z setkami zostaje zapasowo; formy
    zlozone `siedem czterysta` tylko w mowie graczy (§2.3, §10: 50).
11. Eysenlaan: czy „Kantor, Bank, Sklep" oferuje depozyt (wiki milczy) — capture;
    pelny korpus: N=0 wizyt (jak otwarte 4).
12. Zabójstwa followerów/charmów drużynowych — czy przechodzą przez bramkę drużyny
    i jak wygląda ich linia: korpus N=0 (drużyna = sami gracze) — capture.
13. ~~Tag `(NPC)` przy nazwie OFIARY w linii zabójstwa~~ — domkniete na pelnym
    korpusie: 0/1796 linii zabojstw — potwierdzenie N=0; parser scina tagi
    prewencyjnie (§2.5, §10: 51).
14. Zachowanie GMCP `improve` na szczycie skali (15): cap (zostaje 15) czy wrap
    (spada do 0)? Kod klienta spadek w trakcie sesji ignoruje — przy wrap licznik
    zamiera do końca sesji; pelny korpus: `niebotyczne` ×5 (osiagalne; dowod
    emisji poziomow posrednich `gigantyczne → niebotyczne (czas: 0:00)`), ale
    brak dowodu cap/wrap — capture live.
15. Czy śmierć lub inne zdarzenie obniża `improve` w trakcie sesji — kod traktuje
    spadek wyłącznie przy fresh login (absorb między sesjami) — capture live.
16. ~~Linia kliencka wbicia `Wlasnie wbiles postepy: ... (czas: m:ss)`~~ —
    domkniete na pelnym korpusie: 273 wyst. w 56 sesjach — println klienta JEST
    logowany (dokladne timestampy wbic w backfillu); MUD nie drukuje wlasnej
    linii wbicia (kanal czysto GMCP) (§2.6, §10: 52).
17. ~~Format wpisu postępu~~ — decyzja 2026-09-18: wpis prosty (stan +
    timestamp); czas od poprzedniego wbicia i snapshot zabójstw = metadane
    audytowe zdarzenia zasilające weryfikator krzyżowy; polityka rozbieżności:
    flaga + diagnoza, zero auto-korekty (§11).
18. Śmierć członka drużyny — „kto" ROZSTRZYGNIĘTE kodem (diff `objects.nums` +
    akumulowane `objects.data`: id → desc/team, §2.8); capture dotyczy wyłącznie
    „dlaczego": dyskryminator śmierci (brak linii odejścia + ciało w pokoju) vs
    wyjście/quit/teleport; nieznane: forma ciała gracza, `hp` przy zgonie, powrót
    ducha (id). Protokół capture: surowe snapshoty `objects.nums`+`objects.data`
    przed/w trakcie/po + pełny tekst pokoju ±30 s + timestampy; możliwa śmierć
    wymuszona z drugą postacią. Opcjonalnie: ring buffer surowego GMCP (tryb
    diagnostyczny pluginu) = capture samoczynny. Flaga `living` odpada (spec:
    zawsze true = „istota żywa", nie „żyje"); linia tekstowa korpus N=0 (§2.8).

Domknięte na analizie źródeł zleceń 2026-09-17 (Dargoth contracts.ts +
deliveryStats.ts + polishNumberConverter + Mudlet/tjurczyk/Towarzysz (brak modułu
zleceń) + wiki (brak strony o systemie zleceń) + korpus-łowisko 576 sesji (oferty
22, pytania 30, odmowy „niczego mi nie trzeba" 4) + korpus-próbka 33 logi):
28. ~~Linia finalizacji zlecenia~~ — nie istnieje jako osobne zdarzenie kasowe:
    płatność pro-rata z każdą dostawą (para postęp+zapłata na sztukę/kg, dowód
    łowiska v2: 2 kg dostarczone z 4 kg zamówionych, dwie wypłaty) — księga
    pokrywa 100% kasy zleceń liniami par; ewentualny tekst po ostatniej dostawie
    to informacja, nie kasa (§2.4 Realizacja).
29. ~~Rola Szczuroslawa i Fuatsa~~ — reklamodawcy kontraktów myśliwskich
    (`Mam zlecenie na swieze (ryby/mieso), chetnie za nie zaplace.`, Szczuroslaw
    10×, Fuats 6×) — ta sama kategoria co Lucciano/Benito/Aubert (§2.4 Reklama
    kontraktu).
30. ~~Skala liczebników w ofertach~~ — oferty przekraczają 50 słownie
    (`siedemdziesieciu dwoch kilogramow` = 72 kg) i 100 cyframi (`110
    kilogramow`); tabela contracts.ts ślepa powyżej 50 — parser Kronikarza na
    unii liczebników + cyfry (§2.4 blok liczebników).

Domknięte na analizie źródeł zabici 2026-09-18 (Dargoth kill.ts +
killLifetimeStorage.ts w całości + Mudlet counter/counter2/utils + tjurczyk +
Towarzysz (brak modułu) + wiki (brak strony) + korpus 576 sesji (dwa digesty) +
próbka logów HTML):
31. ~~Komplet form prefixów zabójstw~~ — ZABILES/ZABILAS/ZABIL/ZABILA (`AS` =
    postać żeńska, kod; korpus N=0) + OTHER bez licznika; padding 1–3 spacje,
    dominanta 1 (korpus 576) — regex `^\[\s*ZABIL(?:ES|AS|A|)\s*\]` (§2.5).
32. ~~Semantyka `(n / m)`~~ — sesyjne liczniki per mob (mySession /
    mySession+teamSession), NIE lifetime (§2.5 Suffix licznika).
33. ~~Obce zabójstwa~~ — prefix bez licznika = zabójca spoza drużyny (kod +
    korpus); decyzja: filtr, nie zdarzenie kroniki (§2.5).

Domknięte na analizie źródeł postępów 2026-09-18 (Dargoth improveCounter.ts w
całości + Mudlet improve/progress/gmcp_handler_improvement/improve2 + tjurczyk +
Towarzysz (brak modułu) + wiki „Doświadczenie" + korpus 576 sesji (digesty)):
34. ~~Kanoniczna kolejność skali gradacji~~ — 4 źródła zgodne (kod Dargoth
    `IMPROVE_STATES`, Mudlet improve.lua, Mudlet progress.lua, wiki): 0 minimalne,
    1 nieznaczne, 2 bardzo małe, 3 małe, 4 nieduże, 5 zadowalające, 6 spore,
    7 dość duże, 8 znaczne, 9 duże, 10 bardzo duże, 11 ogromne, 12 imponujące,
    13 wspaniałe, 14 gigantyczne, 15 niebotyczne; dawna tabela SPEC miała 2 zamiany
    (7↔8, 12–15) — poprawiona (§2.6 Skala).
35. ~~Semantyka poziomu 0~~ — „zadne/mini" (teksty `zadnych`/`minimalne` = 0);
    liczniki ignorują 0 — nie zdarzenie kroniki (§2.6).
36. ~~Skok delta > 1~~ — klient emituje każdy poziom pośredni osobno; kronika:
    wpis per poziom z własnym timestampem (§2.6).
37. ~~Reconnect / absorb~~ — rozróżnienie per `object_num`: `recordInitial` (cichy
    catch-up, zero duplikatów) vs `record`; spadek przy fresh login = absorb między
    sesjami, reset poziomu bazowego (§2.6); spadek mid-session → otwarte 14–15.

Domknięte na analizie źródeł cech 2026-09-18 (Dargoth lvlCalc.ts w całości +
cechyHistory.ts + afterDeathProgress.ts + Mudlet misc/lvl_calc.lua (tablice 1:1)
+ wiki „Cechy" i „Opcje" + pomoc arkadia.rpg.pl/help/command/cechy + korpus 576
sesji):
38. ~~Tablice kanoniczne cech~~ — 5 cech × 9 poziomów + nadludzki (10); kroki
    bardzo duzo…bardzo niewiele = 0–4; suma cechy = (poziom−1)×5 + krok; progi
    poziomu postaci 58–190 (12 progów, 13 etykiet) (§2.7).
39. ~~Subkomenda `cechy um`~~ — **nie istnieje**: pomoc gry definiuje `cechy`
    jako komendę bezargumentową; echo w korpusie to literówka gracza (§2.7).
40. ~~Linia `Twoj aktualny poziom to ...`~~ — generowana przez klienta
    (calculateLvl), nie przez grę; korpus N=0 zgodne z kodem (§2.7).
41. ~~Gate modyfikatorów cech~~ — odczyt z suffixem `( +N )` odrzucany tylko
    przy włączonych modyfikatorach (`gmcp.char.options.state_modifiers===1`;
    wiki „Opcje") (§2.7).

Domknięte na analizie źródeł umiejętności i języków 2026-09-18 (Dargoth
skills.ts + skillTable.ts + languageSkills.ts + Mudlet misc/skill.lua + wiki
„Umiejętności" i „Języki" + pomoc arkadia.rpg.pl/help/command/umiejetnosci,
jezyki, poziomy, trenuj + próbka logów HTML: um ×4, jezyki ×4):
42. ~~Skala umiejętności~~ — 10 gradacji zgodnych w 3 źródłach (ledwo…
    mistrzowsko) (§2.13).
43. ~~Kanał zmiany umiejętności/języków~~ — brak linii push w grze; zmiana =
    diff kolejnych migawek komend (model cechy_history) (§2.13, §2.14).
44. ~~Komenda `poziomy`~~ — komenda-referencja opisów poziomów cech (pomoc
    gry), walidator skal; output nieznany → otwarte 20.
45. ~~Skala i lista języków~~ — 10 gradacji (= skala wiedzy); 20 języków:
    14 Imperium + 6 Ishtar; Mroczna Mowa = mutacja (§2.14).

Domkniete na pelnym korpusie 2026-09-18 (skrypty ekstrakcyjne v1-v5 na 576
sesjach HTML, 3,86 mln linii; piec przelotow tematycznych):
46. ~~Reputacja pocztowa — linia wynikowa~~ — skala 6 gradacji odpowiedzi
    pracownika poczty po `sprawdz swoja reputacje` (od `Nie znam cie wcale.`
    do kamieni milowych zaufania; wariant formy trzecioosobowej per NPC)
    (§2.1).
47. ~~Typo-formy 50-tki~~ — contracts.ts `piedziesiat/pieedziesieciu` = martwe
    klucze (N=0); gra spojna: `piecdziesiat/piecdziesieciu/piecdziesiata/
    piecdziesiecioma` (1502 trafienia); `piescdziesiat` = literowka gracza
    w mowie, poza parserem (§2.2 reguła 7).
48. ~~Komendy `wplac`/`wyplac`/`przelej`~~ — `wplac`/`wyplac` N=0 (martwe
    legacy po 2011, poza katalogiem); `przelej` = przelewanie plynow
    (`Napelniasz/Dopelniasz ... z ...`), NIGDY bankowe — twardy filtr (§2.3).
49. ~~Nazwy sal bankowych~~ — 18 nazw korpusowych w danych referencyjnych
    (klucz `korpus_pelny_2026_09_18`) (§2.3).
50. ~~`sto+` slownie~~ — N=0 w depozytach i kasie; formy zlozone
    jednostka+setka (`siedem czterysta`, `tysiac szescset`) tylko w mowie
    graczy (§2.3, §2.2 reguła 7).
51. ~~Tag `(NPC)` przy ofierze~~ — 0/1796 linii zabojstw: potwierdzenie N=0;
    scinanie tagow prewencyjne (§2.5).
52. ~~Linia kliencka wbicia~~ — 273 wyst. w 56 sesjach: println klienta jest
    logowany; MUD nie drukuje linii wbicia; dowod delta > 1: sekwencje
    `gigantyczne (czas: X)` → `niebotyczne (czas: 0:00)` (§2.6).
53. ~~`thorzliwy` vs `tchorzliwy`~~ — korpus: 0× / 10× — forma wiki
    kanoniczna, kodowa zapasowa (§2.7).
54. ~~Output komendy `poziomy`~~ — 35 list opisow poziomow + kompletna skala
    cech zrodlem gry (sila: slabiutki … epicko silny); walidator tablic
    potwierdzony Z GRY (§2.7).
55. ~~Modyfikator na wierszu `um`~~ — N=0 w 576 sesjach; mechanika zostaje
    z kodu, parser przygotowany (§2.13).
56. ~~Output `umiejetnosci maksymalne` / `jezyki maksymalne`~~ — um: surowa
    tabela 38 pozycji bez wstawek `[N/10]`; jezyki: wiersze `<jezyk>:
    <poziom>` bez paskow (dobra/pelna) (§2.13, §2.14).
57. ~~Rozszerzona linia oslabienia~~ — potwierdzona 1:1 z kodem: 116 wyst.,
    wszystkie 6 gradacji (minimalne/bardzo male/nieznaczne/male/nieduze/
    zadowalajace); 11 smierci z blokiem posmiertnym (zaswiaty = flavor)
    (§2.8).
58. ~~ERRATA §10:40~~ — linia `Twoj aktualny poziom to ...` WYSTEPUJE w
    logach (println klienta logowany, jak `Wlasnie wbiles` — pkt 52);
    wniosek bez zmian: to nie jest linia gry; wczesniejsza notka „korpus
    N=0 zgodne z kodem" bledna (§2.7). Bonusy przelotow: linia negatywna
    jezykow `Nie znasz zadnych jezykow obcych.` (297×, §2.14); fragment
    wiedzy `Dowiadujesz sie czegos wiecej o ...` (266×/68 sesji) — promocja
    na pelne zdarzenie (§2.12, §11); filtr szumu `porownaj` (§2.12);
    paski klienta `[#---]` w tabeli jezykow (§2.14); wstawka `(±N)` przy
    `[N]` po smierci (§2.7); echa `→ umiejetnosci [typ]` N=0 (§2.13);
    cennik treningu + brak linii kasowej (§2.13); druga forma tabliczki
    prowizji z procentem cyfra (§2.2); tytul „Znawca Wiedzy Wszelakiej"
    (§2.12).

Domkniete na analizie zrodel poczty-listow 2026-09-18 (Dargoth newMail.ts +
poczta.ts w calosci + MailStatus.tsx + PocztaPopup.tsx + testy poczta.test.ts
i mail-status.spec.ts (fixture byte-for-byte z sesji live) + Mudlet
arkadia-skrypty (brak modulu) + spec GMCP forum t=740 + wiki „Poczta" i
„Zwierzeta pocztowe" + pomoc arkadia.rpg.pl/help/command/listy + korpus 576
sesji):
59. ~~Format indeksu i listu~~ — kompletny z testu klienta (byte-for-byte):
    naglowek z 3 odmianami skrotu, wpis `N. [*R*] Temat:`, `Nadawca:`/
    `Odbiorcy:` + data IG w drugiej kolumnie, zawijanie odbiorcow; list:
    6 pol naglowka + pager `(aktualny: N)` + `Nie ma takiego listu.`;
    korpus doklada suffix `(forwardowany)` i raport kurierski `opis
    doreczenia przesylki` (§2.9).
60. ~~Rozbieznosc `otrzymane` vs `odebrane`~~ — pomoc gry i wiki:
    `otrzymane`; gra drukuje `odebrane`/`odebranych` (naglowek + negatyw);
    parser akceptuje obie formy prewencyjnie (§2.9).
61. ~~Konsumenci GMCP Mail~~ — `Mail.State` {unread, unreceived, unsent},
    push przy logowaniu i zmianach; Dargoth: badge (ignoruje unread) +
    popup; indeks i list z popupa sa UKRYWANE (trigger null) — w logach
    tylko z komend manualnych. ERRATA (pkt 62): „Mudlet: brak modulu"
    dotyczy wylacznie upstreamu Delwinga — fork tjurczyk/arkadia modul
    pocztowy MA (§2.9).

Domkniete na analizie zrodel poczty-listow II 2026-09-18 (tjurczyk/arkadia:
mail_creator + 5 szablonow + footer + triggery Arkadia.xml; Dargoth
letter.ts + types/letter + LetterComposer/LetterViewPopup + letterRenderer +
e2e; pomoc ?napisz/?przeczytaj/?list):
62. ~~Moduly pocztowe klientow~~ — ERRATA §2.9 i §10: 61: modul pocztowy
    maja DWA klienty — Dargoth i tjurczyk/arkadia (fork Mudleta, aktywny);
    upstream Delwinga arkadia-skrypty: 0 trafien, brak modulu (§2.9).
63. ~~Prompt edytora i przeplyw `napisz list`~~ — prompt `Wpisz ~?, zeby
    uzyskac pomoc, lub **, by zakonczyc edycje.` zgodny w dwoch klientach;
    konwersacja adresat -> temat -> DW (teksty promptow nieznane — oba
    klienty odpowiadaja na slepo) -> edytor (`~udw` per ukryty odbiorca,
    `**` wysyla) (§2.9).
64. ~~Linie wysylki i statusu~~ — z kodu tjurczyka: `List jest gotowy do
    wyslania.`, `zostal.* pomyslnie wyslan`, `Czeka na ciebie
    nieprzeczytana poczta` (login), `Wysylasz .* na poczte.`, negatywy
    `Nie otrzymal(e)s zadnych...`; korpus N=0 calej partii (Arahi nie
    pisal listow) — dokladne formuly capture (otwarte 23); alias `/list`
    kliencki w obu klientach; padding prefixu `[ POCZTA ]` wariantowy
    (Dargoth 1+1, tjurczyk 2+3) -> `^\[\s*POCZTA\s*\]`; linia kliencka
    podgladu `Podglad listu (szerokosc N, szablon X)` (§2.9).

Nadal otwarte (poczta-listy):
23. Wysylka listu — formuly capture: dokladna forma potwierdzenia (kod
    tjurczyka: `zostal.* pomyslnie wyslan`, korpus N=0), teksty trzech
    promptow konwersacji `napisz list` (adresat/temat/DW), tresc typow
    Gmcp_msgs `mail`/`editor.mail`/`notification.mail`, mechanika
    znacznika `(forwardowany)` (strona nadawca/odbiorca; korpus 1x w
    indeksie odbiorczym), output `aliasy pocztowe`, bare `listy`,
    pozytywny `poczta` — capture live (ring buffer surowego GMCP,
    protokol jak otwarte 18).

Nadal otwarte (cechy, umiejętności, języki):
19. ~~Forma przy odwadze 1~~ — domkniete (§10: 53).
20. ~~Output komendy `poziomy`~~ — domkniete (§10: 54).
21. ~~Forma modyfikatora na wierszu `um`~~ — domkniete N=0 (§10: 55).
22. ~~Output komend `umiejetnosci maksymalne` / `jezyki maksymalne`~~ —
    domkniete (§10: 56).

---

## 11. Rejestr decyzji

Podjęte:
- Pełny produkt od razu (bez wariantów okrojonych); tylko Dargoth, porty później.
- Gate lokalizacji miękki (§5). Klasyfikacja zleceń po dowodzie sprzedaży (§4).
- Sesja = login→logout; zdarzenia zawsze z timestampem; baza globalna + pole character.
- Rdzeń pure TS z DI; jeden plik index.ts; IndexedDB jako storage pluginu.
- Backfill jako obywatel pierwszej klasy (3 adaptery + echo komend).
- (2026-09-16, korpus) Linia postępu traktowana jako spontaniczny push — backfill
  postępów z samych logów, GMCP live jako koroboracja.
- (2026-09-16, korpus) Parser monet: multi-nominał, miedziaki/srebrniki, mithryl,
  liczby mieszane (słownie+cyfry), kwoty nieznormalizowane, kotwica na kwocie nie na
  końcu linii, wtręty dowolne przy `wrecza/zgarnia`.
- (2026-09-16, korpus) Reszta jako osobna gałąź księgowa: wydatek netto = zapłacone
  − reszta; reszta potrafi zawierać mithryl.
- (2026-09-16, korpus) Parser backfillu toleruje prefixy/wstawki klienta
  (`[ POCZTA ] `, `[N] `, `[unk] `, `[x/y]`).
- (2026-09-16, korpus) „Zlecenia" na poczcie = paczki (kolizja kategorii rozstrzygana
  po nadawcy/kontekście).
- (2026-09-16, korpus II) Backfill zabójstw kotwiczy na formie przepisanej przez
  klienta (`[ ZABIL(ES|A|) ] ... (n / m)`); surowa forma w logach nie występuje.
- (2026-09-16, korpus II) Klasyfikacja kasy nigdy po NPC/kwocie: zleceniodawcy są
  adresatami paczek, a ich `wyplaca ci` to wypłaty paczkowe (dowód: konteksty).
- (2026-09-16, korpus II) `Oddajesz ... ze stojka, placac ...` = usługa naprawy
  (krawcy/kowale), osobna kategoria wydatku, nie zakup.
- (2026-09-17, decyzja) Kantor NIE jest degradowany do licznika: denominacja to pełne
  zdarzenie {timestamp, lokacja, wynik} w osi czasu; statystyki łącznie + per kantor
  + per postać + per sesja (§2.2, §8).
- (2026-09-17, decyzja) Mandat uniwersalny: każde zdarzenie kroniki ma timestamp
  (log-time z HTML albo czas live) i podlega globalnym filtrom — zakresy dat i typy
  zdarzeń; cel: 100% pokrycia i wykorzystania danych, zero degradacji do samych
  liczników.
- (2026-09-17, korpus + deklaracja gracza) Szczury: zabójstwo = normalna statystyka
  zabitych; kasa za oddanie ciała = przychód bounty „zapłata za szczury" (wzorzec
  give-and-pay: komentarz NPC z kwotą + `wrecza ci monety`), nie realizacja zlecenia.
  Adler → Nuln, szczurolap → Novigrad (deklaracja).
- (2026-09-17, korpus III) Adler → Nuln **potwierdzone korpusem** (who-lista `Adler
  Winck, Nuln`, pokój `Biuro szczurolapa.`); Ratan drugi szczurolap; myszy 8 pensów;
  sekwencja bounty rozszerzona do 4 linii (oglądanie → komentarz → wręczenie →
  uśmiech); „Szczurolap" tytuł zawodowy graczy (who), `szczury ladowe` obelga,
  maskotka oddzielona od ciał (okno kasowe).
- (2026-09-17, korpus) Linie `Dajesz/Oddajesz`: pełny katalog wykluczeń (paczka,
  naprawa, gracz, bilet transportowy, „nie dajesz rady"); odmowa NPC `A po co mi to
  dajesz?` jako osobne zdarzenie nieudanej próby.
- (2026-09-17, korpus III) Realizacja zlecenia = `<NPC> odbiera od ciebie <towar>
  i wrecza ci <kwota>.` (dostawy częściowe możliwe); `odbiera od ciebie` jest
  trójznaczne (realizacja / sprzedaż sklepowa / zakup „w zamian za towar") — parser
  kotwiczy na pełnej formie (§2.4).
- (2026-09-17, decyzja) Zlecenia rozliczane **pro-rata**: kasa przychodzi z każdą
  dostawą osobno (para postęp+zapłata na sztukę/kg), bez osobnego zdarzenia
  finansowego „finalizacji" — księga zleceń kompletna na liniach par; dawna
  kwestia „linii finalizacji" domknięta jako nieistniejąca kasowo (§2.4, §10: 28).
- (2026-09-18, decyzja) Obce zabójstwa (OTHER — prefix bez licznika) to filtr,
  nie zdarzenie kroniki: nie trafiają do statystyk własnych ani drużyny (§2.5).
- (2026-09-18, kod ×2 + Mudlet ×2 + wiki) Korekta skali gradacji postępów:
  kanoniczna kolejność 0–15 wg czterech zgodnych źródeł; dawna tabela §2.6 miała
  2 zamiany (7↔8 i 12–15) przy deklaracji „1:1 IMPROVE_STATES" — poprawiona;
  mapowanie GMCP liczba→nazwa kotwiczy na kodzie klienta (§2.6, §10: 34).
- (2026-09-18, kod) Skok `improve` o delta > 1 = osobny wpis per poziom pośredni
  (jak `record` w pętli klienta), każdy z własnym timestampem; nigdy wpis
  zbiorczy (§2.6).
- (2026-09-18, decyzja) Wzrost wiedzy = zdarzenie kroniki (tick „wzrosla
  nieznacznie" + fragment „Dowiadujesz sie czegos wiecej" w capture); wpis =
  dziedzina + timestamp (typ zrodla nieznany z linii); bazy zrodlowe: JSON
  Delwinga (44 ksiazki, 44 biblioteki) + API ethel.pl (wpisy eksploracyjne)
  (§2.12, §10: 1).
- (2026-09-18, decyzja + spec GMCP) Śmierć członka drużyny = zdarzenie kroniki
  live-only, brak backfillu; widok statystyk nazwany „Zmarli czlonkowie
  druzyny"; KOREKTA: flaga `living` NIE jest flagą śmierci (spec t=740: zawsze
  true) — kanał detekcji w capture (§2.8, §10: 18).
- (2026-09-18, decyzja) Trening i drużyna jako kategorie kroniki — odrzucone
  (poza zakresem); mechanika zostaje odnotowana w §2.11; śmierć drużynowa
  (§2.8) NIE jest objęta odrzuceniem — to pojedynczy typ zdarzenia, nie
  kategoria.
- (2026-09-18, decyzja) Postępy — format wpisu i audyt: wpis prosty (stan +
  timestamp, wierny GMCP); czas od poprzedniego wbicia (m:ss) i snapshot
  zabójstw (my+team) = **metadane audytowe** zdarzenia, poza tekstem wpisu.
  Weryfikator krzyżowy (live, w obrębie sesji): delta snapshotów między wbiciami
  vs liczba wpisów zabójstw (własne+drużyna) kroniki oraz czas klienta vs
  różnica timestampów kroniki. Rozbieżność (delta != 0): NIGDY auto-korekta
  (wpisy kroniki = dowód, licznik klienta = metadany); wpis audytowy + znacznik
  sesji „pokrycie niepełne"; statystyki pokazują obie wartości z flagą; capture
  diagnostyczny surowych linii ±20 wokół wbicia (paliwo red→green); korekta
  statystyk wg klienta dla klasy przypadków dopiero po danych z flag, osobną
  decyzją, nigdy dla wpisów-zdarzeń. Ograniczenia: zakres sesyjny, snapshot =
  my+team, docięcie na absorb między sesjami, alarm wyłączony przy poziomie 15
  do rozstrzygnięcia otwartego 14, brak działania na backfillu (§2.6, §10: 17).
- (2026-09-18, kod ×2 + wiki + pomoc gry) Cechy: tablice kanoniczne przyjęte
  z kodu klienta ×2 (zgodne z wiki poza formą „tchórzliwy" — otwarte 19);
  kanał premium live `cechy.read` + historia `cechy_history`; koszt zmiany
  cechy wyrażany w postępach (pole lifetime przy odczycie); `cechy um` nie
  istnieje (pomoc) — echo korpusu to literówka gracza (§2.7, §10: 38–41).
- (2026-09-18, decyzja) Umiejętności i języki = kategorie kroniki (§2.13,
  §2.14); brak linii push w grze → zmiana poziomu rozstrzygana diffem migawek
  komend `um`/`jezyki` (model cechy_history); wpis = nazwa + stary→nowy
  poziom + timestamp; modyfikatory jak w cechach (capture, otwarte 21).
- (2026-09-18, decyzja) Koszt treningu księgowany jako wydatek
  „usługa/trening" — wyłącznie łączna kwota wydana na treningi, bez atrybucji
  per umiejętność (indywidualne treningi nierozróżnialne kasowo); trening
  jako kategoria kroniki pozostaje odrzucony (§2.13).
- (2026-09-17, korpus III) Paczka spóźniona potwierdzona: `Niestety, ale
  dostarczyles przesylke po terminie. Dlatego moge ci za nia zaplacic tylko tyle.`
  — wypłata pomniejszona (§2.1).
- (2026-09-17, korpus III) Reszta od NPC: kotwica na słowie `reszty` (nadpłata przy
  zakupie; koszt = zapłacone − reszta); `wrecza ci <kwota>` bez `reszty`
  klasyfikowane po parze/kontekście; flavor kupna: `drapieznym ruchem zgarnia` +
  `kryjac usmiech wrecza ci <towar>` (§2.2).
- (2026-09-17, korpus III) Przekrzyki bretońskie = ogłoszenia bounty czytane na głos
  (ekwiwalent listu gończego); kwota urwana/niewiarygodna — kategoria informacyjna,
  nie zdarzenie finansowe (§2.4).
- (2026-09-17, korpus III) Pole `type` eksportu JSON (LogExportData v1): potwierdzony
  zbiór 11 wartości — strukturalny backfill pokoi/NPC/mowy (§6.2).
- (2026-09-17, łowisko v2) Realizacja zlecenia = sekwencja echo + para {postęp
  `Dziekuje, potrzebuje jeszcze ...` + `odbiera od ciebie ... i wrecza ci ...`}
  ×N dostaw; **brak linii `Dajesz` po stronie gracza** (§2.4).
- (2026-09-17, łowisko v2) Bounty: kotwica na echu komendy + liniach NPC (linia
  dawania ciał nie istnieje — E10); bounty obejmuje też **ciala skavenów** (wpis
  wiedzy `Dostarczyles cialo skavena szczurolapowi z Nuln.`).
- (2026-09-17, łowisko v2) Wpisy `* ` to listing komendy `wiedza` (migawka wiedzy
  postaci) — jednorazowy koroborant czynów, nie licznik ani dziennik zdarzeń (§6.2).
- (2026-09-17, łowisko v2) Znacznik zabójstwa: regex `^\[\s*ZABIL(?:ES|A|)\s*\]`
  (dopełnienie spacjami); forma trzecioosobowa drużyny objęta (§2.5).
- (2026-09-17, korpus III + wiki + kod) Zwrot paczki: kotwica na echu `→ zwroc
  paczke` (4× korpus), linia wynikowa capture; mechanika wiki (urząd źródłowy,
  niewielki spadek reputacji); helpery klienckie nie rozróżniają zwrotu — Kronikarz
  rozróżnia (§2.1).
- (2026-09-17, kod klienta + korpus III) Parser ofert paczek toleruje modyfikacje
  klienta w logach HTML (kolumna `Dystans`, linie `> dystans: N`) — regex
  niekotwiczony, 12/12 ofert testowych; odliczanie limitu kotwiczone na odbiorze,
  nie na pokazaniu tablicy (§2.1).
- (2026-09-17, wiki) Reputacja pocztowa: licznik heurystyczny per rewir (dostawa +1,
  spóźnienie/zwrot −1, zagubienie = blokada ~24 h), kalibracja komendą `sprawdz
  swoja reputacje` (capture); mechaniki: 6 h po wylogowaniu, progi cenowe, limit
  ~50 kg (§2.1).
- (2026-09-17, kod klienta + korpus) Baza adresatów paczek: zdalna npc.json (637
  rekordów, TTL 24 h) + douczanie lokalne; pokrycie korpusu 111/111; etykiety
  pocztowe mapowane na lokacje poczty (§2.1).
- (2026-09-17, korpus + kod klienta) Wycena EQ: surowa forma gry kończy na kwocie
  w miedziakach; `, czyli ...` = doklejka klienta (backfill odcina, bonus
  normalizacji); warianty stosu `Sa tu/Jest tu N sztuk(i) warte...` z kodu (§2.2).
- (2026-09-17, korpus + Towarzysz) Liczebniki nieokreślone (`wiele`/`kilka`/…)
  → kwota null, nigdy 0/1; parser liczebników = unia tabel mianownik+dopełniacz
  (§2.2 reguły 6–7).
- (2026-09-17, korpus) Monety w pojemnikach własnych (sakiewka/plecak) = transfer
  gotówka↔pojemnik; podgląd pojemnika (`Rozwiazujesz… dostrzegasz`) = migawka
  stanu, nie kasa (§2.2).
- (2026-09-17, wiki) Prowizja denominacji per bank: 3/5/8%; stawka lokalna jako
  metadane zdarzenia kantorowego (§2.2).
- (2026-09-17, kod Towarzysza) Atrybucja `Otrzymujesz`: brak w aktualnym
  LOOT_PATTERNS — wzorzec trzymany na korpusie (§2.2).
- (2026-09-17, wiki + mapa) Tabela prowizji kompletna: 8% obejmuje też Zakon
  Sigmara („Bank, Zamek Sigmara", Averland); „Toscania" w tabeli wiki = literówka,
  gra/mapa: Toskania; kantor Eysenlaan poza wiki — stawka capture (§2.2).
- (2026-09-17, kod Dargotha) Wynajem wozu + kaucja: osobne zdarzenie usługowe —
  koszt najmu = wydatek, kaucja = depozyt zwrotny (pełna do 6 h, potem częściowa);
  NIE mylić z przejazdem `Placisz woznicy`; linia refundacji — capture (§2.2).
- (2026-09-17, kod Dargotha) Parser liczebników = unia TRZECH tabel: Towarzysz
  (mianownik + setki/tysiące) + contracts.ts (dopełniacz, `dwu`, złożenia
  `jednego/jednej`) + polishNumberConverter (złożenia 21–99 oba przypadki,
  zbiorowe); typo-formy na liście obserwowanej (§2.2 reguła 7).
- (2026-09-17, spec GMCP forum t=740 + kod) Gotówka NIE istnieje w GMCP —
  potwierdzone specyfikacją (moduły Core/Char/Room/Objects/Gmcp_msgs/Mail) i kodem;
  księga zawsze tekstowa lub premium storage (§2.2).
- (2026-09-17, wiki „Skrytki" + korpus) Wykupienie/rozbudowa skrzynki = wydatek
  „usługa bankowa" (50 zł + 2/5/10/20 mtr, do końca gry postacią, limit 25
  przedmiotów); śmierć = utrata depozytu; linia gry — capture (§2.3).
- (2026-09-17, kod ×3 + wiki + mapa) Lokalizacje depozytów domknięte trzema
  źródłami (Mudlet 16/16, wiki 15/15, +Brugge/Val'Kare z mapy); wiki „Skrytki"
  nieaktualna — mapa źródłem nadrzędnym lokalizacji depozytów (§2.3, §5).
- (2026-09-17, korpus-próbka) Backfill: sklejanie zawiniętych linii przed regexami;
  „modyfikacje klienta" (prefixy, doklejki, tabele pretty DEPOZYT/pojemników)
  rozpoznawane i pomijane (§6.2).
- (2026-09-17, kod) Komendy `wplac`/`wyplac`/`przelej` istnieją w grze, status
  nieznany (konta zlikwidowane 2011) — capture, nie modelujemy na ślepo (§2.3).
- (2026-09-18, pelny korpus v1-v5) Fragment wiedzy `Dowiadujesz sie czegos
  wiecej o <dziedzina>.` = **pelne zdarzenie kroniki** (promocja z capture;
  266 wyst. w 68 sesjach); linie czynu `Widziales/Ogladales/Sluchales
  opowiesci o/Analizowales ...` to towarzyszacy push gry (§2.12).
- (2026-09-18, pelny korpus v1-v5) Komendy `wplac`/`wyplac` = martwe legacy
  (N=0 w 576 sesjach) — poza katalogiem; `przelej` = przelewanie plynow,
  twardy filtr kolizji, nigdy zdarzenie bankowe (§2.3).
- (2026-09-18, pelny korpus v1-v5) Koszt treningu: gra nie drukuje linii
  kasowej przy pobraniu oplaty — ksiegowanie = liczba sesji `Przechodzisz
  szkolenie ...` × stawka z ostatniego cennika `Oto umiejetnosci, w jakich
  mozesz sie szkolic:` tej lokacji; suma laczna wg decyzji „usługa/trening",
  stawki per umiejetnosc dostepne jako metadane (§2.13).
- (2026-09-18, pelny korpus v1-v5) Zasada ogolna po erracie §10:40: **println
  klienta JEST logowany** w logach HTML (linia poziomu, linia wbicia
  postepow, paski cech/jezykow, wstawki `[x/y]`, `(±N)`, `[N]`) — backfill
  rozpoznaje linie klienckie i nie myli ich z liniami gry; digesty korpusu
  raportujace N=0 dla takich linii byly bledne (§2.6, §2.7, §10: 52, 58).
- (2026-09-18, kod + testy + pomoc gry + korpus) Zakres poczty-listow:
  zdarzeniem kroniki jest **nowy list** {timestamp, nadawca}; indeks
  skrzynki, tresc listu i status GMCP `Mail.State` = metadane audytowe,
  nie wpisy; tresc listu trafia do kroniki tylko gdy widoczna w logach
  (komenda manualna — popup klienta ukrywa indeks i list, backfill ich
  nie widzi) (§2.9).
- (2026-09-18, korpus) List-raport `opis doreczenia przesylki` = koroborant
  dostawy paczki (weryfikator krzyzowy z §2.1), nie osobne zdarzenie;
  suffix `(forwardowany)` przy temacie = list doslany (metadane) (§2.9).
- (2026-09-18, pomoc gry + test klienta) Rozbieznosc nazewnicza skrzynki:
  pomoc/wiki `otrzymane` vs gra `odebrane`/`odebranych` — parser akceptuje
  obie formy naglowka i negatywu prewencyjnie (§2.9).
- (2026-09-18, kod tjurczyka + decyzja) **Wyslany list = zdarzenie
  kroniki** (potwierdzenie `zostal.* pomyslnie wyslan` — dokladna forma
  capture, korpus N=0); wpis prosty {timestamp}; `List jest gotowy do
  wyslania.` = linia towarzyszaca; loginowe `Czeka na ciebie
  nieprzeczytana poczta` i `Wysylasz .* na poczte.` = sygnaly statusu
  (metadane), nie wpisy (§2.9).
- (2026-09-18, kod x2) Prefix `[ POCZTA ]` z wariantowym paddingiem —
  parser `^\[\s*POCZTA\s*\]` (regula jak `[ ZABIL* ]`, §10: 9); ASCII-art
  w tresciach listow = szablony klientow (Dargoth/tjurczyk, ta sama
  rodzina pergaminow) — parser tresci bierny (§2.9).

Odrzucone / poza zakresem (z uzasadnieniem):
- Kradzież — nie istnieje na Arkadii.
- Komendy `wplac`/`wyplac` — martwe legacy po likwidacji kont procentowych
  (2011); N=0 na pelnym korpusie 576 sesji (decyzja 2026-09-18, §2.3).
- Kwota denominacji w kantorze — niemierzalna bez porównania ekwipunku (potwierdzone
  na korpusie: zero kwot w kontekstach `zdenominuj`); samo zdarzenie pełnoprawne
  (§2.2); prowizja 8% (tabliczki kantorów) poza bilansowaniem.
- Śmierć członka drużyny — możliwa tylko live (GMCP `living`), odłożona.
- Twardy gate lokalizacji — fałszywe negatywy + unicestwia backfill.
- Zamówienia rzemieślnicze (torby/plecaki/zbroje/miecze na zamówienie u wytwórców) —
  decyzja gracza 2026-09-17: poza katalogiem; sekcja F łowiska zostaje wyłącznie jako
  filtr szumu.
- Zwroty książek do biblioteki (`Oddajesz ksiazke Kerii...`) — decyzja gracza
  2026-09-17: poza katalogiem.
- Trening i drużyna jako kategorie kroniki — odłożone; mechanika odnotowana
  (`trenuj` / `trenuj intensywnie` u mistrzów zawodu; komenda `um` = umiejętności
  + modyfikatory chwilowe; tag `[   DRUZYNA   ]`, składy, przekazanie prowadzenia).
  Koszt treningu księgowany jako wydatek „usługa/trening" (decyzja 2026-09-18,
  §2.13); umiejętności i języki przyjęte jako kategorie (§2.13, §2.14).

---

## 12. Zasady repozytorium

- Treści wyłącznie profesjonalne: bez wulgaryzmów, bez danych wrażliwych, bez tokenów.
- Commity małe, tematyczne; checkbox w dokumentacji domykany w commicie swojej zmiany.
- Nigdy force-push; błędne commity zostają w historii i są dokumentowane.
- Każda zmiana kodu: red→green (§9). Commity dokumentacji nie wymagają testów.
