# Ingestor MQTT

## Opis

Ingestor to serwis backendowy, ktory subskrybuje topici MQTT, waliduje przychodzace wiadomosci JSON i zapisuje poprawne dane pomiarowe do bazy PostgreSQL.

## Architektura

```
ESP32 / MQTT Explorer --> Broker MQTT --> Ingestor --> PostgreSQL
```

## Uruchomienie

Ingestor jest czescia Docker Compose i uruchamia sie automatycznie:

```bash
docker compose up -d --build
```

Podglad logow:

```bash
docker compose logs -f ingestor
```

## Konfiguracja

| Parametr     | Wartosc        | Opis                        |
|-------------|----------------|-----------------------------|
| MQTT_HOST   | `broker`       | Adres brokera MQTT          |
| MQTT_PORT   | `1883`         | Port brokera MQTT           |
| MQTT_TOPIC  | `lab/+/+/+`   | Subskrybowany wzorzec topicu|
| DB_HOST     | `database`     | Adres bazy PostgreSQL       |
| DB_NAME     | `abcd_db`      | Nazwa bazy danych           |
| DB_USER     | `admin`        | Uzytkownik bazy             |

## Walidacja wiadomosci

Ingestor sprawdza kazda wiadomosc przed zapisem:

1. Czy payload jest poprawnym JSON-em
2. Czy zawiera wymagane pola: `device_id`, `sensor`, `value`, `ts_ms`
3. Czy typy danych sa poprawne:
   - `device_id` — niepusty string
   - `sensor` — niepusty string
   - `value` — liczba (int lub float)
   - `ts_ms` — dodatnia liczba calkowita

Wiadomosci niespelniajace wymagan sa odrzucane i logowane.

## Tabela `measurements`

```sql
CREATE TABLE IF NOT EXISTS measurements (
    id SERIAL PRIMARY KEY,
    group_id TEXT,
    device_id TEXT NOT NULL,
    sensor TEXT NOT NULL,
    value DOUBLE PRECISION NOT NULL,
    unit TEXT,
    ts_ms BIGINT NOT NULL,
    seq INTEGER,
    topic TEXT,
    received_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## Weryfikacja zapisu

```bash
docker exec postgres psql -U admin -d abcd_db -c "SELECT * FROM measurements ORDER BY id DESC;"
```

## Testowanie

### Wyslanie poprawnej wiadomosci

```bash
docker exec broker mosquitto_pub -h localhost -p 1883 \
  -t "lab/g03/esp32-test/temperature" \
  -m '{"schema_version":1,"group_id":"g03","device_id":"esp32-test","sensor":"temperature","value":24.5,"unit":"C","ts_ms":1742030400000,"seq":1}'
```

Oczekiwany rezultat: rekord pojawia sie w tabeli `measurements`.

### Wyslanie blednej wiadomosci (brak ts_ms)

```bash
docker exec broker mosquitto_pub -h localhost -p 1883 \
  -t "lab/g03/esp32-test/temperature" \
  -m '{"device_id":"esp32-test","sensor":"temperature","value":24.5}'
```

Oczekiwany rezultat: wiadomosc odrzucona, w logach informacja `Missing required field: ts_ms`.

## Struktura plikow

```
ingestor/
  Dockerfile
  requirements.txt
  ingestor.py      -- logika MQTT + walidacja
  db.py            -- polaczenie z PostgreSQL
```
