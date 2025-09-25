# Reboot-Palo-Alto-on-script-schedule

### Architecture:
* PANORAMA
* SERVER LINUX (for BACKUP)

### Part 1. Create Account on PANORAMA

<img width="1607" height="267" alt="image" src="https://github.com/user-attachments/assets/7982842e-0520-49cd-9110-e8c9c6955e40" />

### Part 2. Create API KEY for this Account The key is valid as long as the account exists and you do not change its password.

Run CMD and paste this command:

```bash
curl -sk "https://192.168.10.44/api/?type=keygen&user=super_admin&password=PASSWORD_FOR_THIS_ACCOUNT"
```

### Part 3. Create directory on Linux server

```bash
mkdir -p /home/super_admin/reboot_firewall/
```

### Part 4. Create file to login and API-KEY

```bash
touch /home/super_admin/reboot_firewall/palo_restart.sh
chmod 600 /home/super_admin/reboot_firewall/palo_restart.sh
```

### Part 5. Create script. Modyfing file (change API-KEY, IP-address, Serial Number)
```bash
#!/usr/bin/env bash
# Restart urządzenia (firewalla) przez Panorama za pomocą API key wbudowanego w skrypt.

set -euo pipefail

############# UZUPEŁNIJ TUTAJ (wstaw swoje wartości) #############
PAN_HOST="192.168.10.44"      # adres Panorama (hostname lub IP)
API_KEY="API-KEY # API key wygenerowany w Panorama
DEVICE_SERIAL="0011223344556677"             # numer seryjny urządzenia docelowego
INSECURE=true                         # true -> curl -k (pomija weryfikację certyfikatu)
TIMEOUT=30                             # timeout curl w sekundach
#################################################################

# opcjonalne: automatyczne potwierdzenie (ustaw true gdy chcesz brak promptu)
AUTO_CONFIRM=true

# timeout i dodatkowe opcje curl
CURL_BASE_OPTS=(--silent --show-error --max-time "$TIMEOUT")

if [ "$INSECURE" = true ]; then
  CURL_BASE_OPTS+=(-k)
fi

API_URL="https://${PAN_HOST}/api/"

# Komenda restartu (PAN-OS XML API)
CMD_XML='<request><restart><system></system></restart></request>'

# confirm
if [ "$AUTO_CONFIRM" != true ]; then
  echo "Przygotowano restart urządzenia o SN: ${DEVICE_SERIAL} przez Panorama: ${PAN_HOST}"
  read -rp "Na pewno wykonać restart? [y/N]: " CONF
  case "$CONF" in
    [yY]|[yY][eE][sS]) ;;
    *) echo "Anulowano."; exit 0;;
  esac
fi

# przygotuj tymczasowy plik na odpowiedź
TMP_RESP="$(mktemp /tmp/pa_resp.XXXXXX)"
trap 'rm -f "$TMP_RESP"' EXIT

# Wyślij zapytanie - używamy GET z parametrami
echo "Wysyłam polecenie restartu do Panorama -> target=${DEVICE_SERIAL} ..."

# curl zapisuje body do pliku, a http code do zmiennej
HTTP_CODE=$(curl "${CURL_BASE_OPTS[@]}" --get \
  --data-urlencode "cmd=${CMD_XML}" \
  --data "type=op" \
  --data-urlencode "key=${API_KEY}" \
  --data-urlencode "target=${DEVICE_SERIAL}" \
  --write-out "%{http_code}" \
  --output "$TMP_RESP" \
  "$API_URL" ) || {
    echo "Błąd: curl zakończył się niepowodzeniem."; exit 2;
}

# wczytaj body
BODY="$(cat "$TMP_RESP")"

if [ "$HTTP_CODE" != "200" ]; then
  echo "Błąd HTTP: ${HTTP_CODE}"
  echo "Odpowiedź serwera:"
  echo "$BODY"
  exit 3
fi

# sprawdź status w XML
if echo "$BODY" | grep -q 'status="success"'; then
  echo "OK: polecenie restartu wysłane pomyślnie. Urządzenie powinno się restartować."
  # pokaż ewentualny komunikat <msg>
  if echo "$BODY" | grep -q "<msg>"; then
    MSG="$(echo "$BODY" | sed -n 's/.<msg>\(.\)<\/msg>.*/\1/p' || true)"
    if [ -n "$MSG" ]; then
      echo "Komunikat Panorama: $MSG"
    fi
  fi
  exit 0
else
  echo "API zwróciło status inny niż success lub wystąpił błąd."
  echo "Odpowiedź Panorama:"
  echo "$BODY"
  exit 4
fi
```

### Part 6. Add modyfing permission to script

```bash
chmod +x /home/super_admin/reboot_firewall/palo_restart.sh
```

### Part 7. Testing manual

```bash
./home/super_admin/reboot_firewall/palo_restart.sh
```

### Part 8. Add to crontab

```bash
(crontab -l 2>/dev/null; echo "15 1 * * * /home/super_admin/reboot_firewall/palo_restart.sh >/dev/null 2>&1") | crontab -
```
