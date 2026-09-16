# Kronikarz — specyfikacja produktu

Status: **planowanie** (analiza korpusu logów zakończona, implementacja nie rozpoczęta).
Data sporządzenia: 2026-09-16. Ostatnia aktualizacja: 2026-09-16 (analiza korpusu:
576 plików HTML klienta Dargoth, 3 865 554 linii, 13 kategorii patternów).

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
| Lista ofert (cel, nagroda, limit czasu) | `packageLineRegex` (miasto + zł/sr/mdz + czas); ciężkie przesyłki oznaczone `*` (`Symbolem * oznaczono przesylki ciezkie.`); komenda `wybierz paczke N` | kod + korpus |
| Etykieta paczki | `Wypisano na niej duzymi literami: <...>` w trzech strukturach: 3-członowa `NAZWA, PROFESJA, MIASTO` (121 próbek, nazwa 1–3 słowa), 2-członowa `NAZWA, MIASTO` (9, np. `ANTONIO, CAMPOGROTTA.`), 1-członowa paczka do samej poczty `POCZTA W <MIEŚCIE>` / `POCZTA MIASTA <MIASTO>` (5); sufiks pilności ` - PILNE!` (24) jako flaga metadanych; opcjonalna linia `Ponizej zas odczytujesz drobniejsze pismo:` | korpus |
| Cudzy odbiór (flavor) | `<NPC> przekazuje <komuś innemu> jakas paczke.` — ignorowane; odbiór własny kotwiczony na `przekazuje ci` | korpus |
| Dostawa | `^Oddajesz pocztowa paczke <opis adresata w dopełniaczu>\.$` + wypłata (patrz niżej); łańcuch potwierdzony kontekstami: `Odkladasz plecak` → `Bierzesz pocztowa paczke...` → `Oddajesz pocztowa paczke X.` → `X wyplaca ci ...` | kod + korpus (16+ wariantów opisów) |
| Zwrot | `^Zwracasz pocztowa paczke` (brak wypłaty) | kod (PackageHelper + tjurczyk); **zero wystąpień w korpusie** — ścieżka rzadka, pattern bez potwierdzenia korpusowego |
| Spóźnienie | `<NPC> mowi do ciebie: Niestety, ale dostarczyles przesylke po terminie. Dlatego moge ci za nia zaplacic tylko tyle.` — dostawa po terminie, **wypłata pomniejszona**; dotychczasowa flaga `Dostarczyles przesylke po terminie` z toolkitu zostaje jako wariant | kod (toolkit) + **korpus III** (łowisko 2026-09-17: 4+ wystąpień; NPC: Luter/Andolf, Guy/Gautier, Lit, anonimowy) |
| Wypłata | `wyplaca ci <waluta>` (waluta wymagana; multi-nominał ze spójnikiem `i`, końcówki `moneta/monety/monet`) LUB `otrzymujesz <waluta>` w oknie ≤ 2 linie od dostawy, bez frazy ` od <kogoś>` | kod + korpus (15 wystąpień; dominuje 4 sr 2 mdz, występują 3-nominałowe) |
| Nieudany odbiór | `Pocztowa paczka jest zbyt ciezka.`, `Lista przesylek zmienila sie i ta, ktora chcesz podjac byc moze nie jest juz ta, ktora widziales w spisie...` | korpus (15× / 34×) |
| Kamień milowy zaufania | `Uwazam cie za osobe wiarygodna i powierze ci kazda przesylke, ktorej zechcesz sie podjac.`, `Jestes uwazany za naprawde wiarygodna osobe, w zwiazku z tym moge powierzyc ci prawie kazda przesylke.` | korpus (14× / 45×) |
| Odmowa odbioru | `Ty juz dla nas dostatecznie ciezko zapracowales`, `Nie ufam ci na tyle, aby powierzyc ci dostarczenie tej przesylki`, `Cos ci sie chyba pomylilo, nie ma takiej oferty`, `nie widzisz tu nikogo, od kogo mozna by wziac zlecenie`; filtrowanie cudzych odmów: liczą się tylko linie `mowi do ciebie`, nie `mowi do <inny gracz>` | kod (tjurczyk) + korpus |
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
| Łup / kasa otrzymana | `^Bierzesz ...`, `^Dostajesz (.+)\.$`, `^Otrzymujesz (.+)\.$`, `wyplaca ci (.+)` | kod (Towarzysz LOOT_PATTERNS) + korpus |
| Wydatek | `^Kupujesz `, `^Placisz ` (obejmuje przejazdy: `Placisz <komu> <kwota>` oraz wariant bez kwoty `Placisz woznicy i wspinasz sie...`), `zgarnia ... monet` (z dowolnym wtrętem, np. `drapieznym ruchem zgarnia`), `odbiera od ciebie ... monet ... w zamian za zakupion` | kod (Towarzysz SPEND_PATTERNS) + korpus |
| Wydatek z resztą | `Placisz <kwota> i dostajesz <kwota> reszty.` — wydatek netto = zapłacone − reszta; reszta potrafi zawierać mithryl (`Placisz 1 mithrylowa monete i dostajesz 64 zlote, 47 srebrnych i 27 miedzianych monet reszty.`) | korpus |
| Usługa: naprawa | `Oddajesz <NPC> <przedmiot> ze stojka, placac <kwota słownie>.` — oddanie ubioru krawcowi (Novigrad, Campogrotta, Nuln) lub oreża/zbroi kowalowi do naprawy; wydatek kategorii „usługa", nie zakup | korpus + wiedza domenowa |
| Reszta od NPC | `<NPC> <wtręt dowolny> wrecza ci <kwota>( reszty)?\.` — regex przepuszcza dowolny wtręt między podmiotem a czasownikiem. **Reguła:** obecność słowa `reszty` = reszta od nadpłaty przy zakupie (koszt = zapłacone − reszta); brak słowa `reszty` = niejednoznaczne (reszta albo zapłata za sprzedaż) — klasyfikacja po parze/kontekście | korpus (flavory: `ze sztucznym usmiechem`, `kryjac usmiech`, `z ogromna niechecia`) |
| Kupno u flavor-sklepikarzy | para/trójka linii: `<NPC> drapieznym ruchem zgarnia <kwota>.` (pobranie zapłaty — wydatek) + `<NPC> kryjac usmiech wrecza ci <towar>.` (wydanie towaru — **bez wpływu na księgę**) + opcjonalnie `... wrecza ci <kwota> reszty.` | korpus (Salithrandir, Zykkis, Adipatus, Myrrhis) |
| Transfer gracz→gracz | `<Gracz> daje ci <moneta>.` (wielka litera imienia, brak tagu `(NPC)`) — przychód oznaczany jako transfer od gracza | korpus (Gwenn, Ulik) |
| Sprzedaż | `^Sprzedajesz ` + zapłata osobną linią przychodu | kod (Towarzysz SELL_PATTERNS) + korpus |
| Liczby słowne | mapowanie liczebników polskich (Towarzysz `polishNumbers`, klient `contracts.ts` POLISH_NUMBERS); **listy mieszane**: słownie i cyfry w jednej linii (`szesc srebrnych monet, 76 zlotych monet i siedem miedzianych monet`) | kod + korpus |
| Wartość ekwipunku | `Wydaje ci sie, ze (jest/sa) wart... mied` i warianty (w tym `nie ma wiekszej wartosci`); niezależne potwierdzenie przeliczników (170 mdz = 14 sr 2 mdz; 4600 mdz = 19 zł 3 sr 4 mdz) | kod (klient: priceEvaluation) + korpus |

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

**Kantor — pełne zdarzenie bez kwoty (dowód korpusowy):** komenda `zdenominuj` (81 ech)
drukuje wyłącznie `Twoje pieniadze zostaly zdenominowane.` (61×) lub `Twoje pieniadze
juz sa maksymalnie zdenominowane.` (20×); w kontekstach ±3 linie zero kwot. Zapisujemy
pełnoprawne zdarzenie denominacji: timestamp (log-time w backfillu / czas live),
lokacja (live: GMCP `room.info`; backfill: nazwa pokoju — pokoje korpusowe: `Kantor
banku w Daevon.`, `Kantorek Vimme Vivaldiego.`, `Posterunek celny i kantor wymiany
walut.`, `Niewielki kantorek.`, generyczny `Kantor.`) oraz wynik (wykonana /
maksymalnie zdenominowane). Zdarzenie wchodzi do osi czasu, wyszukiwania i filtrów
jak każde inne; statystyki: łącznie, per kantor, per postać, per sesja. Z tabliczek
kantorów: `Za kazda transakcje pobieramy tylko 8 procent prowizji.` — prowizja 8%
pozostaje niewidzialnym mikrowydatkiem (wymiana nie zmienia majątku poza prowizją).
Świadomie poza bilansowaniem — ale nie poza kroniką.

**Nie istnieje w grze:** kradzież/okradzenie — poza katalogiem.

### 2.3 Bank i depozyty

| Zdarzenie | Detekcja | Weryfikacja |
|---|---|---|
| Odczyt depozytu | `Twoj depozyt zawiera ...` (monety w liście — słownie i cyframi mieszanie), `Twoj depozyt jest pusty`, `Nie posiadasz wykupionego...` | kod (klient: deposits.ts) + korpus |
| Wpłata / wypłata | triggery kontekstowe na lokacji z bindem `depozyt` (lista referencyjna, §5); linie `Wkladasz/Bierzesz <coś> do/z otwartej skrzynki depozytowej.` (w tym monety: `Wkladasz dwie mithrylowe monety do otwartej skrzynki depozytowej.`) | kod + mapa + korpus |
| Cudze operacje | `<Gracz> bierze ... ze swojej otwartej skrzynki depozytowej.` — marker `swojej` = odfiltrować (nie nasz depozyt) | korpus |
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
| Realizacja | **Sekwencja (konteksty łowiska v2):** echo komendy `→ daj <towaru> <NPC>` (dopełniacz partitywny, np. `→ daj miesiwa mezczyznie`) → **para linii na każdą sztukę**: `<NPC> mowi do ciebie: Dziekuje, potrzebuje jeszcze <pozostała ilość>.` (tracker postępu, warianty: „dwoch kilogramow", „ponad kilogram") + `<NPC> odbiera od ciebie <towar> i wrecza ci <kwota>.` **Brak linii `Dajesz/Oddajesz` po stronie gracza** — dowód to echo + linie NPC. Jedna komenda `daj` może dać N dostaw. Dowód: oferta „czterech kilogramow miesa z zajaca" (00:32:07) → 41 s później dwie pary postęp+zapłata od `Wysoki zwinny mezczyzna` (4 zł 8 sr 4 mdz + 4 zł 13 sr; zlecenie nieukończone w logach — linia finalizacji nieznana, tryb capture). Samo `odbiera od ciebie` jest **trójznaczne** (blok niżej) — kotwica na pełnej formie z `i wrecza ci`; reguła §4 (kolizja ze sprzedażą) bez zmian. Lokalizacja dowodu: Parravon | korpus III + łowisko v2 (2026-09-17) |
| Realizacja give-based (bounty) | `Dajesz/Oddajesz <NPC> <przedmiot>.` + okno: `<NPC> mowi do ciebie: ... daje <kwote> ... .` i/lub `<NPC> wrecza ci monety.` (kwota w komentarzu NPC, linia wręczenia bez kwoty — łączyć w oknie). Potwierdzone: Adler, ciała szczurów | korpus (grepy 2026-09-17) |
| Odmowa dawania | `<NPC> mowi do ciebie: A po co mi to dajesz?` — nieudana próba `daj` (NPC nie chce towaru), osobne zdarzenie | korpus (6× Adler) |
| Reklama kontraktu myśliwskiego | `Mam zlecenie na swieze (skory/ryby/mieso), chetnie za nie zaplace.`, `Mam zlecenie na kilka sztuk broni, chetnie za nie zaplace.` | korpus (Lucciano, Benito, Aubert, Naula/Rudolf) |
| Oferta (warianty korpusowe) | towar na wagę: `Potrzebuje trzydziestu jeden kilogramow miesa z sarny. Dobrze zaplace!`; z jakością: `Potrzebuje osmiu tarcz, przynajmniej sredniej jakosci. Dobrze zaplace za kazda sztuke.`; z rozmiarem/typem: `dziesieciu srednich ryb slodkowodnych`; sufiksy `Dobrze zaplace!` / `Dobrze zaplace za kazda sztuke.` | korpus (Anatol, Ferdinand, Mortimer, Ghaadrav) |
| Cudze oferty | oferty kierowane do innych graczy (`mowi do <ktoś>:` zamiast `mowi do ciebie:`) — ignorowane | korpus |
| Tablica bounty | `| Zleceniodawca | Scigana osoba | Data` | korpus |
| Ogłoszenie bounty (przekrzyk) | `<NPC> krzyczy po bretonsku, ale udaje ci sie zrozumiec tylko czesc: ...` — ogłoszenie bounty na potwory czytane na głos w niektórych miastach (ekwiwalent listu gończego); tekst **urwany**, kwota niewiarygodna → kategoria informacyjna, nie zdarzenie finansowe | korpus III |

**Trójznaczność `odbiera od ciebie` (korpus III):**
1. `<NPC> odbiera od ciebie <towar> i wrecza ci <kwota>.` — **realizacja zlecenia**
   (towar+kasa w jednej linii).
2. `<NPC> odbiera od ciebie <przedmiot>.` — **sprzedaż sklepowa** (NPC przejmuje
   sprzedany przedmiot, bez kasy w linii; sklepikarze: Antonietta, Olof, Ernest).
3. `<NPC> odbiera od ciebie <kwota> w zamian za zakupiony towar.` — **zakup** (NPC
   pobiera zapłatę od gracza; §2.2).

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
   zamowienia!`) — odrębna mechanika, nie mieszać ze zleceniami.

**Bounty za ciała szczurów (korpus III 2026-09-17):** mechanika młodego expa —
zabijasz szczury (zabójstwo liczone **normalnie** w statystykach zabitych, jak każde
inne), a ciała oddajesz za kasę szczurolapom. Pełna sekwencja: `daj <ciała> <NPC>`
→ `<NPC> oglada uwaznie cialo.` (lub `... sterte szczatkow szczura.`) → `<NPC> mowi
do ciebie: Dorodny okaz! Za takiego slicznego szczurka daje trzy srebrne monety.`
→ `<NPC> wrecza ci monety.` → `<NPC> usmiecha sie z zadowoleniem.` Cennik: szczur
3 sr, **mysz 8 pensów** (`Adler mowi: ... mysz - osiem pensow`). Klasyfikacja kasy:
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
| Zabójstwo własne | forma surowa (live, z kodu): `^[ >]*Zabil(?<v>es|as) (?<name>...)\.$`; **forma w logach HTML (backfill): klient przepisuje linię — `[ ZABILES ] Zabiles <name>. (<n> / <m>)`** | kod (kill.ts) + korpus (241 wyst., 103 unikalne) |
| Zabójstwo drużyny | forma surowa: `^[ >]*(?<player>...) zabil(?<v>a?) (?<name>...)\.$`; w logach: `[ ZABIL ] <ktoś> zabil <name>. (n / m)`, `[ ZABILA ] <ktoś> zabila <name>. (n / m)` | kod (kill.ts) + korpus (174+150 wyst., 81+77 unikalnych) |
| Suffix licznika | ` (n / m)` na końcu przepisanej linii — liczniki klienta; metadane gratis, parser toleruje i wykorzystuje | korpus |

**Uwaga korpusowa (zabici):** surowa forma `Zabiles X.` w logach HTML **nie występuje**
— klient przepisuje linię przed zapisem (prefix `[ ZABILES ]` / `[ ZABIL ]` /
`[ ZABILA ]`, suffix licznika). Parser backfillu kotwiczy na przepisanej formie.
**Znacznik ma dopełnienie spacjami do stałej szerokości** (łowisko v2: 1796 linii
`[ ZABIL* ]` w korpusie): `[  ZABILES  ]` (2 spacje), `[   ZABIL   ]` /
`[   ZABILA   ]` (3 spacje) — regex kotwiczy na `^\[\s*ZABIL(?:ES|A|)\s*\]`
(sztywna pojedyncza spacja gubi 100% trafień). Forma trzecioosobowa drużyny też
dotyczy szczurów (`[   ZABILA   ]  Gwenn zabila brudnego smierdzacego szczura.`).
Szum do odfiltrowania: plotki NPC (`mowi: A wczoraj to... zabil`), opisy lokacji
(`kosciotrup`, trupy), nazwy własne (Trupa Trupi Trup). Linie walki otoczenia noszą
tagi `[1/6]`, `[par]`, `[unk]`. Zabójstwa szczurów (exp + bounty u szczurolapów) bez
specjalnego traktowania — normalne wpisy statystyk; kasa za ciała to osobne zdarzenie
bounty (§2.4).
| Premium live | eventy API `kill` {killer: ME/TEAM/OTHER} i `enemyKilled` {objNum, killer, hasBody} | kod (plugin-types) |
| Premium historia | IndexedDB `ArkadiaKillsDB` (indeks `character`) | kod |

### 2.6 Postępy

| Zdarzenie | Detekcja | Weryfikacja |
|---|---|---|
| Wbicie postępu (live) | GMCP `char.state.improve` (0–15; 16 stanów) | kod (klient improveCounter, tjurczyk gmcp_handler_improvement, Towarzysz) |
| Linia tekstowa (sesyjna) | `Poczynil(?:es|as) (.*) postepy, od momentu kiedy .* gry\.$` + wariant bez „momentu": `..., od kiedy wszedles do gry.` + zerowy `Nie poczynil(?:es|as) zadnych postepow...` | kod + korpus (197 wyst. przy 9 echach — **linia jest spontanicznym pushem gry**, nie tylko odpowiedzią na komendę) |
| Linia eksploracji | `Masz wrazenie, iz ostatnimi czasy poczynil(?:es|as) (.*) postepy w poznawaniu swiata.` (+ wariant zerowy) | korpus (155 wyst.) |
| Linia nauki | `Wydaje ci sie, ze poczynil(?:es|as) (.*) postepy w nauce.` | korpus (18 wyst.) |
| Skala | 16 poziomów: minimalne, nieznaczne, bardzo małe, małe, nieduże, zadowalające, spore, znaczne, dość duże, duże, bardzo duże, ogromne, wspaniałe, imponujące, niebotyczne, gigantyczne = 1:1 `IMPROVE_STATES` klienta | kod + wiki + korpus (wszystkie 16 gradacji obecne) |

Linia tekstowa pojawia się samoistnie (push) — **pełny backfill historii postępów
z samych logów jest możliwy**, bez zależności od ech `→ postepy`. Na żywo służy jako
koroboracja GMCP. Licznik postępów resetuje się przy wylogowaniu — naturalnie sesyjny.

### 2.7 Cechy

Klient przechwytuje komendę `cechy` i parsuje odczyt; Kronikarz używa tych samych
zweryfikowanych patternów (klient: lvlCalc.ts):

- `Jestes <opis> i <ile> ci brakuje, zebys mogla? wyzej ocenic sw(a|oj) <cecze>.` z opcjonalnym suffiksem modyfikatora `( +cos )`,
- `Twoja/Twoj <cecha> osiagnela/al nadludzki poziom.`,
- linia zamykająca `Obecnie do waznych cech zaliczasz...` z **opcjonalnym sufiksem** ` Mozesz to zmienic podczas medytacji w gildii podrozniczej.` (oba warianty w korpusie),
- `Twoje cechy sa oslabione po ostatniej smierci.` (snapshot oznaczany jako osłabiony).

Odczyt z modyfikatorem (sprzęt/zioła) jest odrzucany — nie zapisuje się fałszywej
wartości. Detekcja odczytu: event `command` = `cechy` + własny parsing linii (istnieje
też subkomenda `cechy um`).
Premium: storage klienta klucz `cechy_history` (historia zmian i koszt w postępach).
Backfill: echo `→ cechy` + odczyt w logach. **Uwaga korpusowa:** logi HTML zawierają
linie cech w wersji **wzbogaconej przez klienta** — `[18] Jestes krzepki [4/10] i
niewiele [3/5] ci brakuje, zebys mogl wyzej ocenic swa sile.` — parser backfillu musi
tolerować prefiks `[N]` i wstawki `[x/y]` (bonus: wartości liczbowe dostępne wprost).

### 2.8 Śmierć

| Zdarzenie | Detekcja | Weryfikacja |
|---|---|---|
| Śmierć własna | `^Umierasz\.$` (następna linia `Oddalasz sie.` to odejście duszy — ignorowana); przyczyna bywa środowiskowa, nie tylko walka (korpus: upadek — `Odpadasz od sciany i lecisz w dol...`) | kod (Towarzysz DEATH_PATTERNS) + korpus |
| Osłabienie po śmierci | `Twoje cechy sa oslabione po ostatniej smierci\.` + **6 gradacji** wymaganych postępów: minimalne / bardzo małe / nieduże / nieznaczne / małe / zadowalające | kod (klient: afterDeathProgress, lvlCalc) + korpus (143 odczyty przy 10 śmierciach) |
| Śmierć członka drużyny | możliwa wyłącznie live: GMCP `objects.data` flaga `living` przy `team: true` | kod (klient: TeamManager) — **odłożone** |

### 2.9 Poczta (listy)

| Zdarzenie | Detekcja | Weryfikacja |
|---|---|---|
| Nowy list | `^Masz nowa poczte od [A-Za-z]+\.$`; w logach HTML linia występuje z **prefiksem klienta** `[ POCZTA ] ` — parser backfillu akceptuje opcjonalny prefiks | kod (Towarzysz MAIL_PATTERN) + korpus (21 nadawców) |

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

**Prefixy klienta w logach (korpus):** zalogowana linia bywa wersją **przepisaną
przez klienta**, nie surowym tekstem gry — parser backfillu toleruje: `[ POCZTA ] `
(poczta), `[N] ` (licznik przy liniach cech i przybyciach), `[unk] ` (linie walki bez
rozpoznanego typu), wstawki `[x/y]` w liniach cech. W szeptach pomocy poczty komendy
są osadzone jako klikalne elementy i w spłaszczonym tekście znikają — nie traktować
takich linii jako dowodu braku komendy.

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

Nadal otwarte:
1. Linia finalizacji zlecenia (po ostatniej dostawie — w korpusie zlecenie nie
   zostało ukończone: wciąż „potrzebuje jeszcze ponad kilogram") — tryb capture.
2. Rola NPC-ów Szczuroslaw (`Niski korpulentny mezczyzna`) i Fuats (`Lysiejacy
   szczurkowaty mezczyzna`) — niezidentyfikowani, bez kotwic finansowych.
3. Wzrost wiedzy (`twoja wiedza o <kategorii> wzrosla ...`) jako osobne zdarzenie
   kroniki — decyzja odłożona.
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

Odrzucone / poza zakresem (z uzasadnieniem):
- Kradzież — nie istnieje na Arkadii.
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

---

## 12. Zasady repozytorium

- Treści wyłącznie profesjonalne: bez wulgaryzmów, bez danych wrażliwych, bez tokenów.
- Commity małe, tematyczne; checkbox w dokumentacji domykany w commicie swojej zmiany.
- Nigdy force-push; błędne commity zostają w historii i są dokumentowane.
- Każda zmiana kodu: red→green (§9). Commity dokumentacji nie wymagają testów.
