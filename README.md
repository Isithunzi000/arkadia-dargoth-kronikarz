# arkadia-dargoth-kronikarz

Plugin do klienta Dargoth (Arkadia MUD) prowadzący audytowalny dziennik wypraw:
zdarzenia, finanse, paczki, zlecenia, zabici, postępy, cechy, śmierci i sesje —
z pełnymi wykresami, wyszukiwaniem, eksportem i importem oraz backfillem historii
z logów klienta.

## Status

**Planowanie.** Zakończono analizę wykonalności i zbieranie danych (patterny
zweryfikowane w kodzie klienta, skryptach tjurczyka, pluginie Towarzysz i repo
arkadia-python-toolkit). Implementacja nie rozpoczęła się.

## Dokumentacja

- [docs/SPEC.md](docs/SPEC.md) — pełna specyfikacja produktu: katalog zdarzeń
  z patternami, zasady księgowe, gate'y lokalizacji, źródła danych (live i
  backfill), architektura, interfejs, plan testów i rejestr decyzji.
- [docs/lokacje-poczta-banki.json](docs/lokacje-poczta-banki.json) — dane
  referencyjne: punkty pocztowe i bankowe z mapy Arkadii (sync 430) zweryfikowane
  z ArkadiaWiki, wraz z zasadami gate'u lokalizacji.

## Zasady współpracy

- Każda zmiana kodu wymaga dowodu red→green: najpierw test padający na starym
  kodzie (weryfikowany lokalnie), potem implementacja; test i poprawka lądują
  w jednym commicie, a wynik RED jest w nim udokumentowany.
- Testy integracyjne i e2e prowadzi harness w repo arkadia-dargoth-testy.
- W repozytorium nie wolno umieszczać danych wrażliwych ani tokenów.

## Licencja

AGPL-3.0 — patrz [LICENSE](LICENSE).
