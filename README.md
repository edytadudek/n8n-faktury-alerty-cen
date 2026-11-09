# Faktury z Dysku Google do arkusza z alertem o podwyżkach cen (n8n)

**EN:** An n8n workflow that watches a Google Drive folder for invoice PDFs, extracts line items with an OpenAI model into a fixed JSON schema, validates the result, saves it to Google Sheets without duplicates and sends a Telegram alert when a product's net price rises by more than 10% compared with an earlier invoice.

**Stos:** n8n, OpenAI (gpt-4o-mini), Google Drive, Google Sheets, Telegram, JavaScript

![Workflow](screens/workflow.png)

Workflow w n8n, który po wrzuceniu faktury PDF do folderu na Dysku Google automatycznie wyciąga z niej pozycje, zapisuje je w Google Sheets (bez duplikatów) i porównuje ceny z wcześniejszymi fakturami. Gdy cena netto produktu wzrosła o więcej niż 10%, wysyła alert na Telegram. Dokumenty, które nie są fakturami albo nie mają numeru lub daty, nie trafiają do arkusza, a Telegram informuje o nich osobnym alertem.

Projekt był testowany na przykładowych, zanonimizowanych fakturach.

## Problem, który rozwiązuje

Faktury od dostawców trzeba przepisywać do arkusza ręcznie, a podwyżki cen łatwo przeoczyć. Workflow robi to samo automatycznie: od pliku PDF do powiadomienia o zmianie ceny.

## Co pokazuje ten projekt

- Integracja kilku usług przez API: Google Drive, Google Sheets, OpenAI i Telegram.
- Ekstrakcja danych z dokumentu przez model językowy do stałego schematu JSON.
- Walidacja wyniku modelu i ochrona przed halucynacjami (cztery warstwy, opis niżej).
- Zapis odporny na duplikaty: ta sama faktura nie tworzy nowych wierszy.
- Obsługa przypadków brzegowych: plik, który nie jest fakturą, brak numeru lub daty, kolejność wrzucania faktur.
- Obsługa błędów: ponowienia w węzłach korzystających z zewnętrznych usług i alerty o plikach wymagających kontroli.


## Węzły krok po kroku

| Węzeł | Co robi |
|-------|---------|
| PobierzPDF | Co minutę sprawdza wskazany folder na Dysku Google i uruchamia workflow, gdy pojawi się nowy plik. |
| PobierzFV | Pobiera plik z Dysku po ID przekazanym przez trigger. |
| PDFnaTekst | Zamienia PDF na tekst. Wymaga PDF z warstwą tekstową, skany nie zadziałają bez OCR. |
| CzyFaktura? | Sprawdza, czy tekst zawiera słowo „faktura" (lub „invoice") i „NIP". Jeśli nie, workflow kończy się alertem, a plik nie trafia do modelu, więc nie generuje kosztu wywołania. |
| AlertNieFaktura | Wiadomość na Telegram z nazwą pliku, który nie wygląda na fakturę. |
| WyciągnijPozycjeAI | Model `gpt-4o-mini` (temperatura 0) zwraca pozycje w stałym schemacie JSON: `nazwa`, `ilosc`, `jednostka`, `cena_netto`, `vat_proc`, `nr.fv`, `data_wystawienia`. Prompt wymaga liczb z kropką, ceny jednostkowej netto, daty w formacie RRRR-MM-DD i zabrania zmyślania danych. |
| RozdzielPozycje | Rozdziela tablicę pozycji na osobne rekordy, po jednym na pozycję. |
| SprawdzenieZmyśleniaModelu | Przepuszcza tylko pozycje, których nazwa faktycznie występuje w tekście PDF-a. Odsiewa pozycje wymyślone przez model. |
| CzyDataNumer | Sprawdza, czy pozycja ma numer faktury i datę wystawienia. Brak któregokolwiek kieruje ją do alertu. |
| AlertBrakNumeruDaty | Wiadomość na Telegram z nazwą pliku wymagającego ręcznej kontroli (jedna wiadomość na plik). |
| DodanieKlucza | Dodaje pole `klucz` w postaci `numer faktury\|nazwa produktu`, zachowując pozostałe pola. |
| ZapiszDoArkusza | Zapisuje pozycje w Google Sheets. Dopasowanie po kolumnie `klucz`: nowa pozycja dodaje wiersz, istniejąca nadpisuje ten sam wiersz. |
| PobierzHistorieCen | Pobiera całą historię z arkusza (jednorazowo, niezależnie od liczby pozycji). |
| PorownajCeny | Dla każdej nowej pozycji szuka tego samego produktu na fakturze o najbliższej wcześniejszej dacie i liczy zmianę ceny w procentach. Przepuszcza tylko podwyżki powyżej 10%. |
| AlertPodwyzka | Wysyła alert na Telegram z nazwą produktu, zmianą w procentach, cenami i numerami obu faktur. Gdy nic nie podrożało, wiadomość nie jest wysyłana. |

## Zabezpieczenia przed błędami modelu

Test na pliku, który nie był fakturą (zwykły dokument PDF), pokazał, że model potrafi zmyślić pozycje, numery i ceny tak, żeby wypełnić schemat. Dlatego workflow ma kilka warstw ochrony:

1. **CzyFaktura?** odrzuca dokumenty bez słów „faktura" i „NIP" jeszcze przed wywołaniem modelu.
2. **Prompt** wymaga pustej tablicy pozycji dla dokumentów, które nie są fakturami, i zakazuje zmyślania.
3. **SprawdzenieZmyśleniaModelu** odrzuca pozycje, których nazwy nie ma w tekście PDF-a.
4. **CzyDataNumer** nie dopuszcza do zapisu pozycji bez numeru faktury lub daty.

Żadna z tych warstw nie weryfikuje liczb (ilości, cen), więc przy prawdziwych danych warto wyrywkowo porównywać arkusz z PDF-em.

## Struktura arkusza

Pierwszy wiersz z nagłówkami, dokładnie takimi nazwami:

`nazwa | ilosc | jednostka | cena_netto | vat_proc | nr.fv | data_wystawienia | klucz`

Kolumnę `data_wystawienia` należy ustawić jako zwykły tekst, żeby Google nie zamieniał daty na własny format. Kolumnę `klucz` uzupełnia workflow, nic nie wpisuje się w nią ręcznie.

## Ochrona przed duplikatami

Każda pozycja dostaje klucz `nr.fv|nazwa`. Ponowne wrzucenie tej samej faktury nie dodaje nowych wierszy, tylko nadpisuje istniejące.

## Logika alertu

- Porównanie dotyczy ceny netto, więc zmiana stawki VAT nie wygląda jak podwyżka.
- Za cenę poprzednią uznawana jest cena z faktury o najbliższej wcześniejszej dacie, więc kolejność wrzucania faktur nie ma znaczenia.
- Produkty są dopasowywane po nazwie (bez względu na wielkość liter i spacje na brzegach).
- Pozycje z tej samej faktury ani faktury z tego samego dnia nie są ze sobą porównywane.
- Próg 10% zmienia się w jednym miejscu, w warunku `zmiana > 10`.

## Szybki start

1. Utwórz arkusz Google z nagłówkami z sekcji „Struktura arkusza" (kolumna `data_wystawienia` jako zwykły tekst) oraz folder na Dysku Google na faktury.
2. Utwórz bota Telegram (BotFather) i sprawdź ID swojego czatu.
3. Zaimportuj `workflow.json` do n8n (*Import from File*).
4. Wskaż własne poświadczenia: Google Drive OAuth2, Google Sheets OAuth2, klucz API OpenAI i token bota Telegram.
5. W węzłach ustaw własny folder (PobierzPDF), arkusz (ZapiszDoArkusza i PobierzHistorieCen) oraz ID czatu (trzy węzły Telegram).
6. Aktywuj workflow i wrzuć do folderu przykładową fakturę w PDF. Po chwili pozycje pojawią się w arkuszu.
7. Żeby sprawdzić alert, wrzuć drugą fakturę z późniejszą datą, na której ten sam produkt jest droższy o ponad 10%.

Węzły Telegram używają trybu HTML, dzięki czemu nazwy plików ze znakami takimi jak `_` nie psują wysyłki.

## Ograniczenia

- Działa tylko dla PDF z warstwą tekstową (skany wymagają OCR).
- Dwie pozycje o tej samej nazwie na jednej fakturze mają ten sam klucz, więc druga nadpisze pierwszą.
- Ta sama faktura wrzucona ponownie nadpisze wiersze i może wywołać powtórny alert o tej samej podwyżce.
- SprawdzenieZmyśleniaModelu odrzuca pozycje po cichu: jeśli model przepisze nazwę inaczej niż w PDF-ie (np. skrót), prawidłowa pozycja zniknie bez komunikatu.
- Wzorzec w CzyFaktura? jest prosty, więc dokument bez słowa „NIP" (np. paragon) zostanie uznany za nie-fakturę. Nazwy produktów muszą też być zapisane tak samo u dostawcy.
- Brak osobnego przepływu do obsługi błędów (Error Trigger). Ponowienia (Retry On Fail) są ustawione w węzłach OpenAI, Google Sheets i Telegram.
- Tekst faktur trafia do OpenAI, więc przy prawdziwych danych firmy trzeba to uwzględnić.

## Pomysły na rozwój

- Osobny przepływ obsługujący błędy (Error Trigger).
- Walidacja danych (poprawność NIP-u, zgodność sumy pozycji z kwotą faktury) i dane z nagłówka faktury: sprzedawca, nabywca, termin płatności.

