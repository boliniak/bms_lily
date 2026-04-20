# ✅ Rozwiązanie problemu walidacji YAML (YamBMS)

## Co zostało sprawdzone

1. Plik źródłowy:
   - `/home/ubuntu/esphome-yambms/packages/yambms/yambms_web_server.yaml`
   - składnia YAML jest poprawna
   - plik nie jest uszkodzony

2. Konfiguracja użytkownika:
   - `/home/ubuntu/yambms_config.yaml`
   - importy `packages` wskazują na **tryb zdalny** (`url + files`)
   - wszystkie ścieżki do plików package istnieją w repozytorium źródłowym

## Diagnoza problemu

Błąd:

`packages/yambms/yambms_web_server.yaml is not a valid YAML file`

najprawdopodobniej nie wynika z uszkodzenia samego pliku, tylko z jednego z poniższych powodów:

1. **Za stara wersja ESPHome** (pakiety YamBMS wymagają nowszych funkcji).
2. **Pomylenie trybu zdalnego i lokalnego** importu pakietów.
3. Próba użycia lokalnych ścieżek `packages/...` bez pełnego lokalnego katalogu `packages/`.

## Czy trzeba kopiować cały folder `packages/`?

- **Nie**, jeśli używasz tej konfiguracji w trybie zdalnym (`url + files`) — to jest aktualny i zalecany tryb.
- **Tak**, ale tylko jeśli chcesz przejść na tryb lokalny (`!include packages/...`). Wtedy trzeba skopiować **cały** katalog `packages/` z oryginalnego repo i zachować strukturę folderów.

## Wprowadzone poprawki

Zaktualizowano:

- `/home/ubuntu/yambms_config.yaml`
- `/home/ubuntu/yambms_instrukcja.md`

Dodano:

- jasne rozróżnienie trybu zdalnego i lokalnego,
- informację, kiedy wymagane jest kopiowanie folderu `packages/`,
- wymaganie nowszej wersji ESPHome,
- instrukcję o wymaganych sekretach przy włączeniu web server (`web_server_username`, `web_server_password`).

## Zalecana procedura naprawy u użytkownika

1. Zaktualizuj ESPHome do najnowszej stabilnej wersji.
2. Używaj importów `packages` przez `url + files` (bez lokalnego `packages/`).
3. Jeśli włączasz `yambms_web_server.yaml`, upewnij się że masz w `secrets.yaml`:
   - `web_server_username`
   - `web_server_password`
4. Zweryfikuj konfigurację poleceniem:

```bash
esphome config yambms_config.yaml
```

Po tych krokach błąd walidacji YAML nie powinien występować.
