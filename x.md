# Sprawdzian Grupa I: IIS + NAT, rozwiązanie

**Oznaczenia:** zamień `imie` i `nazwisko` na swoje dane (bez polskich znaków), a `nr` na numer z dziennika.
- `IP_MOST` to adres karty mostkowanej serwera (sprawdzisz przez `ipconfig`).
- `IP_HOST` to adres Twojego **komputera fizycznego**, czyli „systemu rzeczywistego” (`ipconfig` na hoście, adres z tej samej sieci co `IP_MOST`).

**Kolejność wykonania:** przygotowanie → role → użytkownicy → NAT → DNS → 1a → 1b → 1c → 1d → 1e.

**Zasada do zapamiętania:** w przeglądarce wpisuj zawsze pełny adres z `http://` i ukośnikiem na końcu, np. `http://www.nazwisko/`. Bez tego Edge potraktuje `www.nazwisko` jak hasło do wyszukania, bo nie zna takiej końcówki domeny.

---

## 0. Przygotowanie VM

**[VirtualBox]** maszyny wyłączone:
1. **W2k25**:
   - Karta 1: **Mostkowana karta sieciowa**
   - Karta 2: **Sieć wewnętrzna**, nazwa `intnet`
2. **W11** → Karta 1: **Sieć wewnętrzna**, ta sama nazwa `intnet`

**Dlaczego:** serwer stoi między siecią wewnętrzną a mostkowaną i robi za router z NAT. Host (system rzeczywisty) widzi serwer przez kartę mostkowaną, co jest potrzebne w 1d.

**[W2k25]** `ncpa.cpl`:
1. Zmień nazwy kart na **MOST** i **WEWN**.
2. **WEWN** → IPv4:
   - IP `10.90.0.1`, maska `255.255.0.0`
   - **bez bramy**
3. **MOST**: zostaw DHCP albo ustaw adres od nauczyciela. Preferowany DNS: `127.0.0.1`.

**[W11]** IPv4:
- IP `10.90.0.10`, maska `255.255.0.0`
- brama `10.90.0.1`
- DNS `10.90.0.1`

**Dlaczego maska 255.255.0.0:** sieć z zadania to `/16`, a `/16` oznacza właśnie maskę `255.255.0.0`.

**Test:** `ipconfig` na obu maszynach i na hoście. Zanotuj `IP_MOST` i `IP_HOST`.

---

## 1. Instalacja ról

**[W2k25]** Menedżer serwera → **Dodaj role i funkcje** → Role serwera:
1. Zaznacz **Serwer DNS**.
2. Zaznacz **Dostęp zdalny** → usługa roli **Routing**.
3. Zaznacz **Serwer sieci Web (IIS)**, a w usługach ról dodatkowo:
   - Zabezpieczenia → **Uwierzytelnianie podstawowe** (1b)
   - Zabezpieczenia → **Ograniczenia adresów IP i domen** (1d)
   - Narzędzia do zarządzania → **Usługa zarządzania** (1d)
   - **Serwer FTP** → **Usługa FTP** (1e)
4. Kliknij **Zainstaluj**.

**Dlaczego:** IIS jest modułowy. Bez tych usług ról odpowiednie ikony w ogóle nie pojawią się w Menedżerze IIS.

---

## 2. Konta użytkowników

**[W2k25]** Narzędzia → **Zarządzanie komputerem** → Użytkownicy i grupy lokalne → Użytkownicy → PPM → **Nowy użytkownik…**
1. Utwórz `imie` i `nazwisko`, oba z hasłem.
2. Odznacz „Użytkownik musi zmienić hasło przy następnym logowaniu”.

**Dlaczego:** tych kont użyjesz do logowania do strony (1b) i do FTP (1e). Uwierzytelnianie podstawowe i FTP sprawdzają zwykłe konta Windows.

---

## 3. NAT (zadanie NAT)

**[W2k25]** Narzędzia → **Routing i dostęp zdalny**:
1. PPM na serwerze → **Konfiguruj i włącz routing i dostęp zdalny** → Dalej.
2. Wybierz **Translator adresów sieciowych (NAT)** → Dalej.
3. **Użyj tego interfejsu publicznego** → **MOST** → Dalej.
4. Usługi nazw i adresów: skonfigurujesz później → Zakończ → **Uruchom usługę**.

**Dlaczego:** każdy adres `10.90.x.x` wychodzący przez serwer zostaje podmieniony na `IP_MOST`. Na zewnątrz widać tylko serwer.

**Test [W11]:**
- `ping 10.90.0.1`
- `ping 8.8.8.8`
- `ping wp.pl`
- `tracert 8.8.8.8`: pierwszy skok to `10.90.0.1`

**Test „dowolny adres”:** zmień w W11 adres na np. `10.90.5.20` (maska, brama i DNS bez zmian) i powtórz `ping 8.8.8.8`. To pokazuje, że NAT działa dla całej sieci `/16`, a nie tylko jednego adresu. Potem możesz wrócić do `10.90.0.10`.

**Test [W2k25]:** IPv4 → **NAT** → PPM na **MOST** → **Pokaż mapowania**. Widać tłumaczenia adresów `10.90.x.x`.

**Typowy błąd:** ping do `10.90.0.1` nie odpowiada. Włącz regułę zapory **Udostępnianie plików i drukarek (żądanie echa — ICMPv4 — przychodzące)**.

---

## 4. DNS + dokumentacja

**[W2k25]** Narzędzia → **DNS**:
1. **Strefy wyszukiwania do przodu** → PPM → **Nowa strefa…** → Strefa podstawowa.
2. Nazwa strefy: `nazwisko` → Dalej → Zakończ.
3. PPM na strefie → **Nowy host (A lub AAAA)…**:
   - `www` → `IP_MOST`
   - `imie` → `IP_MOST`

**Dlaczego:** strefa `nazwisko` z rekordami `www` i `imie` daje pełne nazwy `www.nazwisko` i `imie.nazwisko`. Obie wskazują ten sam serwer, a IIS rozróżni witryny po nazwie hosta.

**Do dokumentacji (tego wymaga uwaga w treści zadania):**
- zrzut okna Menedżera DNS ze strefą `nazwisko` i oboma rekordami A
- zrzut `cmd` z wynikami `nslookup www.nazwisko` i `nslookup imie.nazwisko` (Server: localhost, Address: `IP_MOST`)
- ten sam `nslookup` na W11 (Server: `10.90.0.1`)

---

## Zadanie 1a: www.nazwisko

**[W2k25]**
1. Utwórz `C:\pRo` i skopiuj do niego stronę z serwera S (w Eksploratorze wpisz `\\adres_serwera_S`).
2. Menedżer IIS → PPM **Witryny** → **Dodaj witrynę sieci Web…**:
   - nazwa: `www`, ścieżka: `C:\pRo`
   - typ: `http`, IP: `IP_MOST`, port: `80`
   - **Nazwa hosta:** `www.nazwisko`
3. Jeśli plik strony ma niestandardową nazwę: witryna → **Dokument domyślny** → **Dodaj…**

**Test:** `http://www.nazwisko/` na W2k25 i na W11 wyświetla stronę z serwera S.

**Typowy błąd:** 403.14 oznacza, że brakuje pliku na liście dokumentów domyślnych.

---

## Zadanie 1b: logowanie do witryny

**[W2k25]** witryna **www** → **Uwierzytelnianie**:
1. **Uwierzytelnianie anonimowe** → **Wyłącz**.
2. **Uwierzytelnianie podstawowe** → **Włącz**.

**Dlaczego:** dopóki działa uwierzytelnianie anonimowe, IIS wpuszcza każdego bez pytania. Po jego wyłączeniu jedyną drogą wejścia jest login i hasło. Żółte ostrzeżenie o SSL jest normalne: hasło idzie zakodowane Base64, a nie zaszyfrowane, dlatego w praktyce stosuje się to tylko z HTTPS.

**Test (W11, okno InPrivate, żeby przeglądarka nie pamiętała logowania):**
1. `http://www.nazwisko/` → pojawia się okno logowania.
2. Wpisz złe hasło → okno wraca, a po Anuluj widać **401 Unauthorized**.
3. Wpisz `imie` + poprawne hasło → strona się wyświetla.

---

## Zadanie 1c: druga witryna imie.nazwisko

**[W2k25]**
1. Utwórz `C:\imie` i w Notatniku plik `mojedane.html`, np.:
```html
<!DOCTYPE html>
<html><head><meta charset="utf-8"/></head>
<body><h2>Imię: ... | Nazwisko: ... | Nr w dzienniku: ...</h2></body>
</html>
```
   Przy zapisie wybierz **Wszystkie pliki**, a kodowanie **UTF-8**. Inaczej Notatnik zapisze `mojedane.html.txt`.
2. **Dodaj witrynę sieci Web…**:
   - nazwa: `imie`, ścieżka: `C:\imie`
   - IP: `IP_MOST`, port: `80`
   - **Nazwa hosta:** `imie.nazwisko`
3. Witryna `imie` → **Dokument domyślny** → **Dodaj…** → `mojedane.html`.

**Dlaczego:** obie witryny mają ten sam IP i port 80, ale różne nazwy hosta, więc działają jednocześnie. `mojedane.html` nie jest na domyślnej liście, dlatego trzeba go dodać ręcznie.

**Test:** w dwóch kartach otwórz `http://www.nazwisko/` (pyta o login) i `http://imie.nazwisko/` (od razu pokazuje Twoje dane). Zrób jeden zrzut z oboma kartami i listą witryn, na której obie mają stan **Uruchomione**.

---

## Zadanie 1d: zdalne połączenie i blokada systemu rzeczywistego

**[W2k25] Przygotowanie:**
1. **Default Web Site** → **Rozpocznij**, jeśli jest zatrzymana.
2. **Test przed blokadą:** na hoście otwórz `http://IP_MOST/`. Widać stronę powitalną IIS. Zapisz zrzut, bo to „przed”.
3. Serwer → **Usługa zarządzania**:
   - zaznacz **Włącz połączenia zdalne**
   - wybierz **Poświadczenia systemu Windows lub poświadczenia Menedżera usług IIS**
   - IP `IP_MOST`, port `8172`
   - **Zastosuj** → **Uruchom**
4. Serwer → **Użytkownicy Menedżera usług IIS** → **Dodaj użytkownika…** → `webadmin` + hasło.
5. **Default Web Site** → **Uprawnienia Menedżera usług IIS** → **Zezwalaj użytkownikowi…** → `webadmin`.
6. Serwer → **Delegowanie funkcji** → **Ograniczenia adresów IP i domen** → zmień z „Tylko do odczytu” na **Odczyt/zapis**.

**Dlaczego punkt 6:** domyślnie ta funkcja jest zablokowana dla zdalnego administratora. Bez zmiany zobaczysz alert „Ta funkcja została zablokowana i jest tylko do odczytu”.

**Połączenie zdalne:**
1. Strona początkowa → **Połącz z witryną…**
2. Serwer: `IP_MOST`, witryna: `Default Web Site` → `webadmin` + hasło.
3. Przy alercie certyfikatu kliknij **Połącz** → nadaj nazwę połączenia.

**Blokada (w węźle połączenia zdalnego):**
1. **Ograniczenia adresów IP i domen** → **Dodaj wpis Odmów…**
2. **Określony adres IP:** `IP_HOST` → OK.

**Dlaczego:** IIS sprawdza adres nadawcy żądania. Gdy pasuje do wpisu „Odmów”, zamiast strony odsyła błąd 403.

**Test:**
- pasek stanu Menedżera IIS: „IP_MOST:8172 jako webadmin”
- **host:** `http://IP_MOST/` (odśwież przez Ctrl+F5) → **403 Forbidden**
- **W11:** `http://IP_MOST/` → strona IIS działa, bo blokada dotyczy tylko hosta

---

## Zadanie 1e: FTP z izolacją użytkowników

**[W2k25] Katalogi:**
1. Utwórz `C:\FTP\LocalUser\imie` i `C:\FTP\LocalUser\nazwisko`. Nazwa `LocalUser` musi być dokładnie taka.
2. Do każdego folderu wrzuć inny plik testowy, np. `plik_imie.txt` i `plik_nazwisko.txt`.
3. Folder `imie` → Właściwości → **Zabezpieczenia** → dodaj użytkownika `imie` → **Modyfikacja**. To samo dla `nazwisko` w jego folderze.

**Dlaczego:** przy izolacji IIS sam szuka katalogu `LocalUser\<nazwa_konta>` dla kont lokalnych. Użytkownik po zalogowaniu „ląduje” w swoim folderze i nie może wyjść wyżej.

**[W2k25] Witryna FTP:** PPM **Witryny** → **Dodaj witrynę FTP…**
1. Nazwa `FTP`, ścieżka `C:\FTP` → Dalej.
2. IP: `IP_MOST`, port: `21`, SSL: **Bez protokołu SSL** → Dalej.
3. Uwierzytelnianie: **Podstawowe**.
4. Autoryzacja: **Określeni użytkownicy** → `imie, nazwisko`.
5. Uprawnienia: **Odczyt** + **Zapis** → Zakończ.
6. Witryna FTP → **Izolowanie użytkowników FTP** → **Katalog nazwy użytkownika (wyłącz katalogi globalne)** → **Zastosuj**.

**Test** (`cmd` → `ftp` → `open IP_MOST`):
1. Zaloguj się jako `imie`:
   - `dir` pokazuje tylko `plik_imie.txt`
   - `put C:\Windows\win.ini` działa (zapis)
   - `cd ..` → dalej ten sam katalog (nie da się wyjść)
2. Zaloguj się jako `nazwisko`: `dir` pokazuje tylko jego plik, bez `win.ini` od `imie`.
3. Zaloguj się jako `Administrator` → **530** (brak dostępu).

Testuj najpierw na serwerze, potem z W11.

---

## Tabela błędów

| Błąd | Przyczyna | Naprawa |
|---|---|---|
| Edge wyszukuje zamiast otworzyć | brak `http://` w adresie | wpisz `http://www.nazwisko/` |
| ERR_NAME_NOT_RESOLVED | brak rekordu A lub zły DNS w karcie | dodaj host, ustaw DNS |
| 401 Unauthorized | złe hasło lub konto | sprawdź konto w Zarządzaniu komputerem |
| Strona nie pyta o hasło | anonimowe nadal włączone albo przeglądarka pamięta login | wyłącz anonimowe, testuj w InPrivate |
| 403.14 | pliku nie ma w Dokumencie domyślnym | **Dodaj…** nazwę pliku |
| 403 na hoście bez blokady | Default Web Site zatrzymana albo zły wpis | sprawdź stan i adres w ograniczeniach |
| Blokada nie działa | zły `IP_HOST` (np. adres karty VirtualBox) | `ipconfig` na hoście, adres z sieci mostkowanej |
| Szara funkcja w połączeniu zdalnym | delegowanie „Tylko do odczytu” | zmień na **Odczyt/zapis** |
| Brak połączenia z :8172 | WMSVC zatrzymana | Usługa zarządzania → **Uruchom** |
| „Inna witryna używa portu” | brak nazwy hosta w powiązaniu | uzupełnij nazwę hosta |
| FTP 530 „home directory inaccessible” | zła nazwa folderu (`LocalUser`, nazwa konta) | popraw ścieżkę |
| FTP 550 przy `put` | brak Zapisu w FTP lub Modyfikacji w NTFS | popraw oba miejsca |
| W11 bez internetu | zły interfejs publiczny w NAT lub brak bramy | NAT na **MOST**, brama `10.90.0.1` |

Mogę połączyć obie grupy w jeden dokument do wydrukowania przed sprawdzianem.

Wysłano z programu Outlook dla systemu Android
