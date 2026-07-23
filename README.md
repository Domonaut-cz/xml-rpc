# Domonaut XML-RPC importní API

Domonaut.cz poskytuje XML-RPC rozhraní pro aktivní export realitních kanceláří, makléřů, nabídek a fotografií z externích realitních systémů.

- Web: [https://domonaut.cz](https://domonaut.cz)
- XML-RPC endpoint: [https://domonaut.cz/xmlrpc/sreality.php](https://domonaut.cz/xmlrpc/sreality.php)
- Registrace RK: [https://domonaut.cz/xmlrpc/register.php](https://domonaut.cz/xmlrpc/register.php)
- Referenční dokumentace Sreality: [https://admin.sreality.cz/doc/import.pdf](https://admin.sreality.cz/doc/import.pdf)

## Přehled integračního toku

Externí systém aktivně odesílá data do Domonautu:

1. zaregistruje realitní kancelář přes JSON endpoint,
2. získá `client_id`, `secret` a samostatně přidělený `software_key`,
3. zahájí XML-RPC session metodami `getHash` a `login`,
4. odešle nebo aktualizuje makléře metodou `addSeller`,
5. odešle nebo aktualizuje nabídky metodou `addAdvert`,
6. synchronizuje fotografie přes `addPhoto`, `listPhoto` a `delPhoto`,
7. odstraněné nabídky archivuje přes `delAdvert`,
8. session ukončí metodou `logout`.

Každé autorizované XML-RPC volání spotřebuje aktuální `sessionId`. Před dalším voláním je nutné vypočítat nový token.

## Endpointy a transport

| Účel | Endpoint | Metoda | Content-Type |
| --- | --- | --- | --- |
| XML-RPC import | `https://domonaut.cz/xmlrpc/sreality.php` | `POST` | `text/xml; charset=utf-8` |
| Registrace RK | `https://domonaut.cz/xmlrpc/register.php` | `POST` | `application/json` |

Veškerý text se posílá v UTF-8.

XML-RPC endpoint vrací standardní `methodResponse`. Běžné aplikační chyby jsou vracené uvnitř struktury odpovědi a HTTP odpověď proto může být `200 OK` i při neúspěšné operaci.

Neplatné XML-RPC XML nebo neznámá metoda vrací XML-RPC `fault`, nikoli běžnou strukturu `status/statusMessage/output`.

## Identifikátory

Domonaut používá interní identifikátory s příponou `_id` a identifikátory externího systému s příponou `_rkid`.

| Pole | Rozsah | Význam | Doporučení |
| --- | --- | --- | --- |
| `client_id` | globální | Interní ID realitní kanceláře v Domonautu | Uložit z registrační odpovědi |
| `rpcRkid` | v rámci integračního zdroje | ID realitní kanceláře v externím systému | Stabilní, max. 60 znaků |
| `seller_id` | globální | Interní ID makléře v Domonautu | Lze uložit z odpovědi `addSeller` |
| `seller_rkid` | v rámci jedné RK | ID makléře v externím systému | Preferovaný stabilní identifikátor, max. 20 znaků |
| `advert_id` | globální | Interní ID nabídky v Domonautu | Lze uložit z odpovědi `addAdvert` |
| `advert_rkid` | v rámci jedné RK | ID nabídky v externím systému | Povinné pro vytvoření nové nabídky; doporučeno max. 60 znaků |
| `photo_id` | v rámci nabídky | Identifikátor fotografie vrácený Domonautem | Může být číslo nebo řetězec |
| `photo_rkid` | v rámci nabídky | ID fotografie v externím systému | Stabilní; používat hodnotu vrácenou API |

Pro maximální předvídatelnost posílejte v jedné operaci pouze jeden z dvojice `_id` / `_rkid`. Jednotlivé metody mají při současném vyplnění různou prioritu identifikátorů.

### Pravidla pro `advert_rkid`

- Pro novou nabídku je v Domonautu `advert_rkid` prakticky povinné.
- Musí být stabilní po celou dobu života nabídky.
- Opakovaný `addAdvert` se stejným `advert_rkid` provede aktualizaci.
- `advert_rkid` existující nabídky se při aktualizaci nemění.
- Identifikátor musí být unikátní v rámci jedné realitní kanceláře.
- Doporučený formát je ASCII bez diakritiky, například `offer-123456`.

## Formát XML-RPC odpovědi

Po dekódování XML-RPC má běžná odpověď tuto strukturu:

```json
{
  "status": 200,
  "statusMessage": "OK",
  "output": []
}
```

`output` je vždy pole. Pokud metoda vrací jeden objekt, je struktura vložena jako první prvek pole.

Příklad odpovědi `getHash`:

```json
{
  "status": 200,
  "statusMessage": "OK",
  "output": [
    {
      "sessionId": "..."
    }
  ]
}
```

### Stavové kódy používané aktuální implementací

| Kód | Význam | Typické situace |
| --- | --- | --- |
| `200` | OK | Operace proběhla úspěšně |
| `402` | Client unknown | Neplatné `client_id` nebo RK nemá aktivní XML-RPC secret |
| `404` | Not found | Nabídka nebo fotografie nebyla nalezena; některé mazací metody jsou místo toho idempotentní a vrací `200` |
| `407` | Bad session | Session je neplatná, expirovaná, neautorizovaná nebo nebyla správně zrotována |
| `452` | Invalid parameters | Chybí požadovaný parametr, datum nelze zpracovat nebo je překročen podporovaný limit |
| `461` | Seller not found | Makléř nebyl nalezen v aktuální RK |
| `462` | Seller login already exists | Kolize unikátního údaje makléře |
| `500` | Internal server error | Databázová, souborová nebo jiná interní chyba |

Kód `405` je součástí kompatibilní terminologie pro neplatný software key, ale klient má v praxi považovat neúspěšné přihlášení za chybu session a zahájit novou relaci od `getHash`.

## Registrace realitní kanceláře

Registrační endpoint vytvoří nebo aktualizuje realitní kancelář a vrátí údaje potřebné pro XML-RPC autentizaci.

### Požadavek

```http
POST /xmlrpc/register.php HTTP/1.1
Host: domonaut.cz
Authorization: Bearer <REGISTER_TOKEN>
Content-Type: application/json
```

Bearer token určuje integrační zdroj, například konkrétní realitní software. Hodnota zdroje se v JSON neposílá.

### Pole JSON požadavku

| Pole | Povinné | Max. délka | Chování |
| --- | --- | ---: | --- |
| `rpcRkid` | ano | 60 | Stabilní ID RK v externím systému; při delší hodnotě endpoint vrátí chybu |
| `name` | ano | 60 | Název RK; delší hodnota je zkrácena |
| `ico` | ne | 8 | Prázdná hodnota se uloží jako `null`; delší hodnota je zkrácena |
| `email` | ne | 100 | Prázdná hodnota se uloží jako `null`; delší hodnota je zkrácena |
| `phone` | ne | 13 | Prázdná hodnota se uloží jako `null`; delší hodnota je zkrácena |
| `website` | ne | 100 | Prázdná hodnota se uloží jako `null`; delší hodnota je zkrácena |
| `aboutMe` | ne | 200 | Krátký popis RK; delší hodnota je zkrácena |

Endpoint u volitelných polí neprovádí úplnou významovou validaci. Neověřuje například syntaxi e-mailu, URL, telefonu ani skutečnou platnost IČO. Odesílatel má proto posílat již normalizovaná data.

### Příklad registrace

```bash
curl --request POST \
  --url 'https://domonaut.cz/xmlrpc/register.php' \
  --header 'Authorization: Bearer <REGISTER_TOKEN>' \
  --header 'Content-Type: application/json' \
  --data '{
    "rpcRkid": "agency-123",
    "name": "Ukázková realitní kancelář",
    "ico": "12345678",
    "email": "info@example.com",
    "phone": "+420700000000",
    "website": "https://example.com",
    "aboutMe": "Krátký popis realitní kanceláře"
  }'
```

### Úspěšná odpověď

```json
{
  "status": "ok",
  "data": {
    "id": 123,
    "secret": "generated-secret"
  }
}
```

- `data.id` je XML-RPC `client_id`.
- `data.secret` je nové tajemství pro tuto RK.
- Pro výpočet session se používá `password_md5 = md5(secret)`.
- Secret má délku 12 až 16 znaků, obsahuje nejméně jedno písmeno a jednu číslici a nepoužívá některé snadno zaměnitelné znaky.
- Server ukládá pouze MD5 otisk secretu.

### Opakovaná registrace

Klíčem kanceláře je kombinace integračního zdroje a `rpcRkid`.

Opakovaný požadavek se stejným `rpcRkid` a stejným zdrojem:

1. aktualizuje údaje existující RK,
2. vygeneruje nový secret,
3. okamžitě nahradí předchozí `xmlrpc_password_md5`,
4. vrátí stejné `client_id` a nový secret.

Opakovaná registrace tedy není pouze bezpečné opakování čtení. Vždy rotuje přihlašovací tajemství a může zneplatnit právě používané session.

Pro výchozí integrační zdroj umí endpoint převzít starší záznam RK, který dosud nemá vyplněné `src`, a při aktualizaci mu zdroj doplnit.

### Unikátnost IČO

Neprázdné IČO je unikátní napříč tabulkou realitních kanceláří. Stejné IČO nelze zaregistrovat pro jiný záznam, ani pod jiným integračním zdrojem.

### Registrační chyby

Registrační endpoint používá skutečné HTTP stavové kódy a JSON chybovou odpověď:

```json
{
  "status": "error",
  "message": "invalid_json"
}
```

| HTTP | `message` | Význam |
| ---: | --- | --- |
| `400` | `invalid_src` | Zdroj odvozený z tokenu je prázdný nebo delší než 16 znaků |
| `400` | `invalid_json` | Tělo není platný JSON objekt |
| `400` | `rpcRkid is required` | Chybí `rpcRkid` |
| `400` | `rpcRkid too long` | `rpcRkid` je delší než 60 znaků |
| `400` | `name is required` | Chybí název RK |
| `401` | `unauthorized` | Chybí nebo nesouhlasí Bearer token |
| `405` | `method_not_allowed` | Byla použita jiná HTTP metoda než `POST` |
| `409` | `ico_already_exists` | IČO patří jiné RK |
| `409` | `duplicate_key` | Jiná databázová kolize unikátního klíče |
| `500` | `database_error` | Databázová nebo interní chyba |

## Autentizace a rotace session

XML-RPC autentizace používá stejný princip rotujícího session tokenu jako importní rozhraní Sreality.

### Potřebné údaje

| Údaj | Zdroj |
| --- | --- |
| `client_id` | `data.id` z registrace RK |
| `secret` | `data.secret` z registrace RK |
| `password_md5` | Lokálně vypočtené `md5(secret)` |
| `software_key` | Klíč přidělený Domonautem pro konkrétní integrační zdroj |

`software_key` není vracen registračním endpointem a neposílá se jako samostatný parametr XML-RPC metod. Je součástí lokálního výpočtu dalšího `sessionId`.

### Výpočet dalšího session tokenu

```php
function nextSessionId(
    string $currentSessionId,
    string $passwordMd5,
    string $softwareKey
): string {
    $fixedPart = substr($currentSessionId, 0, 48);
    $variablePart = md5($currentSessionId . $passwordMd5 . $softwareKey);

    return $fixedPart . $variablePart;
}
```

Výsledný token má 80 znaků:

- prvních 48 znaků je fixní část vytvořená metodou `getHash`,
- posledních 32 znaků je MD5 vypočtené z předchozího celého tokenu, `password_md5` a `software_key`.

### Přihlášení

```text
1. getHash(client_id)
2. sessionId = nextSessionId(sessionId, password_md5, software_key)
3. login(sessionId)
```

### Každé další volání

Před každou další autorizovanou metodou, včetně `logout`, vypočítejte nový token:

```text
sessionId = nextSessionId(sessionId, password_md5, software_key)
addSeller(sessionId, ...)

sessionId = nextSessionId(sessionId, password_md5, software_key)
addAdvert(sessionId, ...)

sessionId = nextSessionId(sessionId, password_md5, software_key)
logout(sessionId)
```

Token použitý v požadavku, který prošel ověřením session, už znovu nepoužívejte. To platí i tehdy, když metoda následně vrátí aplikační chybu, například `404`, `452`, `461` nebo `500`: ověření a rotace session probíhají před vlastní validací dat metody.

Výjimkou je `407 Bad session`. V takovém případě token přijat nebyl a klient má vždy zahájit novou relaci od `getHash`; nemá se pokoušet odhadovat stav původní session.

### TTL a souběh

Výchozí neaktivní TTL session je 900 sekund. Konkrétní integrační zdroj může mít v serverové konfiguraci jinou hodnotu.

Po každém úspěšném ověření se aktualizuje čas poslední aktivity. Při překročení TTL server vrátí `407`.

Požadavky v jedné session posílejte sériově. Paralelní volání se stejným stavem session vede k závodu v rotaci tokenu. Pro paralelní import použijte samostatnou session pro každý nezávislý worker.

Při `407`:

1. přestaňte používat dosavadní token,
2. zavolejte nové `getHash(client_id)`,
3. znovu proveďte `login`,
4. bezpečně opakujte poslední idempotentní operaci.

## Přehled metod

### Veřejné a doporučené metody

| Oblast | Metoda | Signatura |
| --- | --- | --- |
| Session | `getHash` | `getHash(client_id)` |
| Session | `login` | `login(session_id)` |
| Session | `logout` | `logout(session_id)` |
| Makléři | `addSeller` | `addSeller(session_id, seller_id, seller_rkid, client_data)` |
| Makléři | `delSeller` | `delSeller(session_id, seller_id, seller_rkid)` |
| Makléři | `listSeller` | `listSeller(session_id)` |
| Nabídky | `addAdvert` | `addAdvert(session_id, advert_data)` |
| Nabídky | `delAdvert` | `delAdvert(session_id, advert_id, advert_rkid)` |
| Nabídky | `listAdvert` | `listAdvert(session_id)` |
| Kompatibilita | `listProject` | `listProject(session_id)` |
| Statistiky | `listStat` | `listStat(session_id, advert_id[], advert_rkid[])` |
| Statistiky | `listAllDailyStat` | `listAllDailyStat(session_id, date)` |
| Statistiky | `listSellerStat` | `listSellerStat(session_id, seller_id, seller_rkid, from, till)` |
| Fotografie | `addPhoto` | `addPhoto(session_id, advert_id, advert_rkid, data)` |
| Fotografie | `listPhoto` | `listPhoto(session_id, advert_id, advert_rkid)` |
| Fotografie | `delPhoto` | Viz samostatná kapitola; doporučená kompatibilní varianta se liší od původní signatury Sreality |
| Poptávky | `listFullInquiry` | `listFullInquiry(session_id, date)` |

### Legacy kompatibilita

Endpoint stále přijímá také:

- `updateAdvert(session_id, advert_data)` – vyžaduje existující nabídku; pouze pro starší klienty, nepoužívat v nové integraci,
- `updateSeller(session_id, seller_id, seller_rkid, client_data)` – vyžaduje existujícího makléře; Domonaut rozšíření mimo běžný veřejný Sreality flow.

Nové exportéry mají pro aktualizaci nabídky vždy opakovat `addAdvert` a pro vytvoření nebo aktualizaci makléře používat `addSeller`.

## Session metody

### `getHash`

Založí novou neautorizovanou session.

```text
getHash(client_id)
```

| Parametr | Typ | Popis |
| --- | --- | --- |
| `client_id` | `int` | Interní ID RK vrácené registrací |

Úspěšný výstup:

```json
{
  "status": 200,
  "statusMessage": "OK",
  "output": [
    {
      "sessionId": "80-character-session-token"
    }
  ]
}
```

Možné chyby:

- `402`, pokud `client_id` neexistuje, je neplatné nebo RK nemá nastavené XML-RPC heslo,
- `500`, pokud nelze session uložit.

### `login`

Autorizuje session vytvořenou metodou `getHash`.

```text
login(rotated_session_id)
```

Před voláním je nutné zrotovat token vrácený z `getHash`.

Úspěšný výstup má prázdné `output`.

Při neplatném výpočtu, expirované session nebo chybné konfiguraci vrací endpoint `407`.

### `logout`

Ukončí a odstraní session.

```text
logout(next_rotated_session_id)
```

Také před `logout` je nutné provést další rotaci. Po úspěšném odhlášení už fixní část session neexistuje a token nelze znovu použít.

## Makléři

### `addSeller`

Vytvoří nového makléře nebo aktualizuje existujícího. Jde o upsert metodu.

```text
addSeller(session_id, seller_id, seller_rkid, client_data)
```

Pro běžnou integraci:

- posílejte `seller_id = 0`,
- vyplňte stabilní `seller_rkid`,
- při každé změně údajů zopakujte `addSeller` se stejným `seller_rkid`.

Identifikace probíhá v rámci aktuální RK:

1. pokud je `seller_id > 0`, hledá se interní ID,
2. jinak se hledá `seller_rkid` v externím kódu makléře,
3. jako kompatibilní fallback se může použít starší mapování na `username`.

Musí být vyplněno alespoň jedno z polí `seller_id` nebo `seller_rkid`.

#### `client_data`

| Pole | Typ | Chování |
| --- | --- | --- |
| `client_name` | `string` | Celé jméno; první slovo se uloží jako jméno, zbytek jako příjmení |
| `contact_email` | `string` | Preferovaný kontaktní e-mail |
| `client_login` | `string` | Fallback e-mail a při zvláštním legacy scénáři také fallback externí kód |
| `contact_gsm` | `string` | Preferovaný telefon |
| `contact_phone` | `string` | Fallback telefon |
| `client_ic` | `string` nebo `int` | IČO makléře |
| `photo` | `base64` | Volitelná profilová fotografie |

#### Normalizace

- `seller_rkid` může mít nejvýše 20 znaků; delší hodnota vrací `452`.
- Telefonu se odstraní mezery, pomlčky a jiné nepodporované znaky.
- Telefon může obsahovat číslice a volitelný úvodní znak `+`.
- Normalizovaný telefon může mít nejvýše 13 znaků; delší hodnota vrací `452`.
- Jméno se rozdělí na první slovo a zbytek.
- Jméno se zkrátí na 12 znaků a příjmení na 25 znaků podle interního datového modelu Domonautu.
- Databázový sloupec pro e-mail má limit 100 znaků a pro IČO 8 znaků.
- `addSeller` e-mail ani IČO před zápisem sám nezkracuje a významově je nevaliduje. Hodnoty delší než databázový limit proto neposílejte; podle režimu MySQL mohou skončit databázovou chybou místo řízeného statusu `452`.

Při aktualizaci přes `addSeller` se zcela chybějící e-mail, telefon, jméno nebo IČO typicky zachová z existujícího záznamu. Prázdný řetězec však není ve všech případech totéž co chybějící pole, proto pro zachování hodnoty pole raději vůbec neposílejte.

Pokud je přítomné pole `photo`, server:

1. dekóduje base64,
2. odstraní dosavadní profilové soubory makléře,
3. uloží nový obsah jako `profile.jpg`.

Profilová fotografie v tomto flow není součástí samostatné foto metody.

#### Výstup

```json
{
  "status": 200,
  "statusMessage": "OK",
  "output": [
    {
      "seller_id": 456,
      "seller_rkid": "broker-42"
    }
  ]
}
```

Možné chyby:

- `407` neplatná session,
- `452` chybí identifikace, `seller_rkid` nebo telefon překračuje limit,
- `462` kolize unikátního údaje,
- `500` interní chyba.

### `updateSeller` – legacy rozšíření

```text
updateSeller(session_id, seller_id, seller_rkid, client_data)
```

Metoda pouze aktualizuje existujícího makléře. Pokud makléř neexistuje, vrátí `461`.

Na rozdíl od `addSeller` tato implementace nezpracovává pole `photo`. Pro nové integrace používejte opakovaný `addSeller`.

### `delSeller`

```text
delSeller(session_id, seller_id, seller_rkid)
```

Odstraní makléře z aktuální RK. Před odstraněním uživatele projdou jeho nabídky standardním interním procesem odstranění nabídky.

Pokud makléř neexistuje, vrátí `461`.

### `listSeller`

```text
listSeller(session_id)
```

Vrátí makléře přiřazené aktuální RK, seřazené podle interního ID vzestupně.

Každý prvek `output` obsahuje:

| Pole | Typ | Popis |
| --- | --- | --- |
| `seller_id` | `int` | Interní ID makléře |
| `seller_rkid` | `string` | Externí kód; při legacy záznamu fallback na `username` |
| `client_name` | `string` | Jméno a příjmení |
| `client_login` | `string` | E-mail nebo prázdný řetězec |
| `photo` | `int` | Aktuální implementace vrací vždy `0` |

## Nabídky

### `addAdvert`

`addAdvert` je jediná veřejná metoda pro vytvoření i aktualizaci nabídky.

```text
addAdvert(session_id, advert_data)
```

Aktuální endpoint toleruje také třetí parametr s `advert_rkid`, ale nové integrace ho mají posílat uvnitř `advert_data` podle standardní signatury.

#### Vytvoření nové nabídky

Pro nový záznam jsou v Domonautu vyžadována tato data:

| Pole | Podmínka |
| --- | --- |
| `advert_rkid` | Povinné stabilní ID nabídky |
| `advert_function` | Povinné |
| `advert_type` | Povinné |
| `description` | Povinný neprázdný popis |
| `seller_id` nebo `seller_rkid` | Povinná vazba na existujícího makléře |

Lokalita, cena a podtyp nejsou technicky povinné pro uložení do Domonautu. Pro kvalitní zveřejnění je však doporučeno posílat alespoň město nebo RÚIAN a platné souřadnice.

#### Aktualizace existující nabídky

Pokud `advert_rkid` již v aktuální RK existuje, stejné volání `addAdvert` provede aktualizaci:

- pole přítomná v `advert_data` se znovu zpracují,
- většina nepřítomných polí zůstane beze změny,
- není nutné opakovat `seller_id` ani `seller_rkid`; vazba na původního makléře se zachová,
- archivovaná nabídka se znovu aktivuje,
- `advert_rkid` zůstává původní a nelze ho tímto způsobem přejmenovat.

Domonaut nevrací `409` pouze proto, že nabídka existuje. Existující `advert_rkid` je očekávaný způsob aktualizace.

#### Částečné aktualizace a mazání hodnot

Chování prázdných hodnot není obecně jednotné pro všechna pole. Klient nemá předpokládat, že každá prázdná hodnota automaticky smaže údaj podle celé dokumentace Sreality.

Příklady skutečného chování:

- prázdné `advert_code` nastaví interní hodnotu na `null`,
- `advert_price <= 0` odstraní konkrétní cenu,
- prázdná `advert_price_text_note` odstraní poznámku k ceně,
- prázdné nebo neplatné `ready_date` nastaví datum na `null`,
- prázdný `description` může uložit prázdný popis při aktualizaci; neposílejte ho,
- prázdné lokalitní texty mohou odstranit příslušnou část adresy,
- neznámá pole jsou ignorována.

#### Souběžné volání

Při práci podle `advert_rkid` endpoint používá MySQL advisory lock vázaný na kombinaci RK a `advert_rkid`. Lock čeká nejvýše 5 sekund. Pokud ho nelze získat, metoda vrátí `500`.

Exportér přesto nemá aktualizovat stejnou nabídku paralelně.

#### Výstup

```json
{
  "status": 200,
  "statusMessage": "OK",
  "output": [
    {
      "advert_id": 789
    }
  ]
}
```

Stejné `advert_id` se vrací při dalších aktualizacích stejné nabídky.

#### Možné chyby

| Kód | Situace |
| ---: | --- |
| `407` | Neplatná session |
| `404` | Legacy `updateAdvert` nebo aktualizace pouze přes neexistující `advert_id` |
| `452` | Chybí identifikátor, typ nabídky, typ nemovitosti nebo jiné nutné údaje |
| `461` | Makléř neexistuje nebo nepatří aktuální RK |
| `500` | Databázová chyba, nedostupný lock nebo jiná interní chyba |

### Příklad vytvoření nabídky

Struktura je níže zapsána jako JSON pro čitelnost; přes XML-RPC se posílá jako `struct`.

```json
{
  "advert_rkid": "offer-987654",
  "advert_code": "ZAK-2026-001",
  "advert_function": 2,
  "advert_type": 1,
  "advert_subtype": 4,
  "description": "Pronájem bytu 2+kk po rekonstrukci.",
  "advert_price": 24500,
  "advert_price_currency": 1,
  "advert_price_unit": 2,
  "cost_of_living": "3500 Kč měsíčně",
  "refundable_deposit": 49000,
  "tenant_not_pay_commission": 1,
  "building_condition": 9,
  "building_type": 2,
  "ownership": 1,
  "furnished": 3,
  "energy_efficiency_rating": 3,
  "usable_area": 54,
  "floor_number": 3,
  "floors": 6,
  "ready_date": "2026-08-01",
  "seller_rkid": "broker-42",
  "locality_street": "Radlická",
  "locality_cp": "123",
  "locality_co": "45",
  "locality_city": "Praha",
  "locality_citypart": "Smíchov",
  "locality_psc": "15000",
  "locality_latitude": 50.07123,
  "locality_longitude": 14.40123
}
```

### Příklad částečné aktualizace

```json
{
  "advert_rkid": "offer-987654",
  "advert_price": 23900,
  "advert_price_text_note": "Včetně poplatku za internet",
  "ready_date": "2026-08-15"
}
```

Při tomto volání zůstane původní popis, typ, makléř a lokalita nezměněna.

### `updateAdvert` – pouze legacy kompatibilita

Endpoint metodu stále přijímá kvůli staršímu exportéru. Vyžaduje, aby nabídka existovala, a při chybějícím záznamu vrací `404`.

Metoda není součástí veřejné dokumentace Sreality a nesmí se používat v nové integraci. Stejného výsledku dosáhnete opakovaným `addAdvert`.

### `delAdvert`

```text
delAdvert(session_id, advert_id, advert_rkid)
```

Identifikujte nabídku jedním z parametrů:

```text
delAdvert(session_id, 0, "offer-987654")
```

nebo:

```text
delAdvert(session_id, 789, "")
```

Pokud je nabídka nalezena, projde standardním interním procesem odstranění. Pokud neexistuje, metoda přesto vrátí `200`, takže je idempotentní.

### `listAdvert`

```text
listAdvert(session_id)
```

Vrací pouze aktivní nabídky aktuální RK, tedy záznamy s `archived IS NULL`, seřazené podle data vytvoření vzestupně.

Každý prvek `output` obsahuje:

| Pole | Typ | Popis |
| --- | --- | --- |
| `advert_id` | `int` | Interní ID nabídky |
| `advert_rkid` | `string` | Externí ID nabídky |
| `advert_type` | `int` | Kód typu nemovitosti |
| `advert_subtype` | `int` nebo `null` | Kód podtypu odvozený z uložených kategorií |
| `advert_url` | `string` | URL detailu ve formátu `/detail/<advert_id>` |
| `hash_id` | `string` | Interní ID převedené na řetězec |
| `modified` | `string` | Datum poslední změny nebo vytvoření ve formátu `YYYY-MM-DD` |
| `published` | `int` | U vrácených aktivních nabídek `1` |
| `published_status` | `int` | U vrácených aktivních nabídek `1` |
| `top` | `int` | `1`, pokud je nabídka právě zvýhodněna, jinak `0` |

Host část `advert_url` se sestavuje z aktuálního HTTP požadavku na endpoint.

### `listProject`

```text
listProject(session_id)
```

Jde o kompatibilní alias `listAdvert`. Domonaut touto metodou nevrací samostatné developerské projekty; vrací stejná data jako `listAdvert`.

## Podporovaná pole `advert_data`

Endpoint zpracovává následující pole. Ostatní položky kompatibilního Sreality exportu jsou přijaty v XML struktuře, ale tato implementace je ignoruje.

| Pole | Typ | Skutečné chování |
| --- | --- | --- |
| `advert_id` | `int` | Interní identifikátor existující nabídky; pro nové integrace preferujte `advert_rkid` |
| `advert_rkid` | `string` | Externí ID; povinné pro nový záznam, doporučený limit 60 znaků |
| `advert_code` | `string` | Interní číslo zakázky RK; ořízne se na 60 znaků, prázdná hodnota se smaže |
| `advert_function` | `int` | Typ nabídky `1–4`; povinný při vytvoření |
| `advert_type` | `int` | Typ nemovitosti `1–5`; povinný při vytvoření |
| `advert_subtype` | `int` | Podtyp podle podporovaného číselníku |
| `description` | `string` | Povinný při vytvoření; HTML značky se odstraní a text se ořízne na okrajích |
| `advert_price` | `double` | Hodnota větší než nula se uloží; nula nebo záporná hodnota odstraní konkrétní cenu |
| `advert_price_currency` | `int` | Pouze `3` znamená EUR; všechny ostatní hodnoty se mapují na Kč |
| `advert_price_unit` | `int` | `3` nebo `4` se uloží jako `za m²`; ostatní hodnoty jako celková cena |
| `advert_price_text_note` | `string` | Poznámka k ceně, max. 200 znaků; prázdná hodnota se smaže |
| `poplatky` | `int` | Domonaut rozšíření pro přímé měsíční poplatky |
| `cost_of_living` | `string` nebo `double` | Z hodnoty se vezme číslo, zaokrouhlí se a uloží jako poplatky |
| `refundable_deposit` | `double` | Kauce, zaokrouhlená na celé Kč |
| `commission` | `string` nebo `double` | Provize, zaokrouhlená na celé Kč |
| `tenant_not_pay_commission` | `int` nebo `bool` | Hodnota `1` znamená, že nájemce neplatí provizi |
| `building_condition` | `int` | Stav objektu podle podporovaného číselníku |
| `building_type` | `int` | Konstrukce podle podporovaného číselníku |
| `ownership` | `int` | Vlastnictví podle číselníku |
| `furnished` | `int` | Vybavenost podle číselníku |
| `energy_efficiency_rating` | `int` | Energetická třída A–G |
| `usable_area` | `int` | Užitná plocha; ukládá se pro všechny typy kromě pozemku |
| `estate_area` | `int` | Plocha pozemku; ukládá se jen pro dům nebo pozemek |
| `floor_number` | `int` | Podlaží; ukládá se jen pro byt |
| `floors` | `int` | Celkový počet podlaží; ukládá se jen pro byt |
| `advert_room_count` | `int` | Počet pokojů domu; hodnoty se omezí na rozsah `1–5` |
| `ready_date` | `string` | Datum nastěhování; podporované formáty jsou uvedené níže |
| `seller_id` | `int` | Interní ID makléře |
| `seller_rkid` | `string` | Externí ID makléře |
| `locality_cp` | `string` | Číslo popisné, max. 8 znaků |
| `locality_co` | `string` | Číslo orientační; připojí se k `locality_cp` za `/` |
| `locality_street` | `string` | Ulice, max. 41 znaků |
| `locality_city` | `string` | Město, max. 33 znaků |
| `locality_citypart` | `string` | Část obce; vstup až 80 znaků, výsledná uložená část max. 33 znaků |
| `locality_psc` | `string` | PSČ, max. 5 znaků |
| `locality_latitude` | `double` | Zeměpisná šířka |
| `locality_longitude` | `double` | Zeměpisná délka |
| `locality_ruian` | `int` nebo číselný `string` | RÚIAN kód |
| `locality_ruian_level` | `int` nebo číselný `string` | Úroveň RÚIAN; bez hodnoty se předpokládá úroveň `11` |

### Zástupné textové hodnoty

U vybraných textových polí se následující hodnoty po odstranění mezer, teček, lomítek a diakritiky považují za chybějící údaj:

```text
neuvedeno
neuvedena
neuvedene
nezadano
nezadana
nezadane
undefined
null
n/a
```

### Datum nastěhování

Podporované vstupy:

```text
YYYYMMDDTHHMMSS
YYYY-MM-DD
```

Časová část se neukládá. Rok musí být nejméně 1900 a datum musí být kalendářně platné.

## Finanční údaje

### Cena

- `advert_price > 1` se zobrazí jako konkrétní cena.
- `advert_price <= 1` se zobrazí jako "Cena na vyžádání".

### Měna

| `advert_price_currency` | Uložená měna |
| ---: | --- |
| `3` | EUR |
| jiná hodnota nebo chybí | Kč |

USD se samostatně neukládá. Kód `2`, který v původním číselníku Sreality označuje USD, se v Domonautu mapuje na Kč.

### Jednotka ceny

| `advert_price_unit` | Uložení |
| ---: | --- |
| `3` | `za m²` |
| `4` | `za m²` |
| ostatní | celková cena |

Domonaut ve svém datovém modelu nerozlišuje `za m²` a `za m²/měsíc`.

### Poplatky

`cost_of_living` může být číslo nebo text. Server:

1. zkusí parsovat celý obsah jako číslo,
2. pokud to nejde, vezme první číselnou hodnotu v textu,
3. desetinnou čárku převede na tečku,
4. hodnotu zaokrouhlí na celé číslo.

Příklad:

```text
"3 500 Kč + elektřina"
```

V aktuálním jednoduchém parseru nemusí text s oddělenými tisíci vytvořit očekávanou hodnotu. Pro spolehlivý výsledek posílejte čisté číslo, například `3500`.

### Provize a kauce

- `tenant_not_pay_commission = 1` nastaví nabídku jako bez provize pro nájemce.
- Neprázdné `commission` se převede na číslo, zaokrouhlí a uloží jako konkrétní částka. Posílejte čistou číselnou hodnotu; textové částky s oddělovači nebo měnou se zde neparsují stejně tolerantně jako `cost_of_living`.
- `refundable_deposit` se zaokrouhlí na celé číslo.
- Pokud není explicitně určena provize, endpoint umí jako pomocný fallback rozpoznat některé české formulace typu „bez provize“ v `description` a `advert_price_text_note`.
- U kauce může fráze typu „bez kauce“ u běžných zdrojů přepsat i současně poslanou hodnotu `refundable_deposit` na `0`. Payload proto nesmí obsahovat vzájemně rozporný strukturovaný údaj a text.
- Nové nabídky navíc procházejí interním finančním extraktorem. Jeho přesná pravidla nejsou veřejným API kontraktem; integrace se má spoléhat na konzistentní strukturovaná pole.

## Lokalita a geokódování

Domonaut kombinuje explicitní adresní údaje, RÚIAN a externí geokódování.

### Doporučené pořadí kvality vstupu

1. platné `locality_ruian` a `locality_ruian_level`,
2. město, ulice, číslo popisné/orientační a PSČ,
3. platné GPS souřadnice,
4. alespoň město.

Ideální export posílá RÚIAN i textové údaje. Text může pomoci zpřesnit obecnější RÚIAN kód na ulici nebo adresní místo.

### Podporované RÚIAN úrovně

| Kód | Úroveň |
| ---: | --- |
| `1` | Okres |
| `3` | Obec |
| `5` | Část obce |
| `7` | Ulice |
| `9` | Stavební objekt |
| `11` | Adresní místo |
| `17` | Městská část / městský obvod |

Pokud je `locality_ruian` vyplněné a `locality_ruian_level` chybí, server předpokládá `11` – adresní místo.

### Zpracování RÚIAN

Server dotazuje veřejný ArcGIS MapServer ČÚZK a podle úrovně doplňuje:

- číslo domu,
- ulici,
- část obce,
- obec,
- okres nebo pražský obvod,
- kraj,
- PSČ,
- souřadnice.

U obecnějších úrovní se pokouší vstup zpřesnit podle `locality_city`, `locality_street`, `locality_cp` a `locality_co`.

Výsledky RÚIAN se interně cachují. Při výpadku služby může server použít starší cache nebo další geokódovací fallback.

### Nominatim fallback

Pokud RÚIAN:

- není zadán,
- není dohledán,
- dočasně selže,
- nebo neposkytne dostatečně úplnou lokalitu,

server se pokusí doplnit adresu přes interní Nominatim flow.

Neúspěch geokódování sám o sobě nemusí způsobit chybu `addAdvert`. Pokud se nepodaří určit město, Domonaut použije v pořadí:

1. část obce,
2. okres,
3. hodnotu `Neuvedeno`.

Chybějící souřadnice se uloží jako `0.0`.

### Změna lokality při aktualizaci

Pokud `addAdvert` u existující nabídky obsahuje některé lokalitní pole, server porovná novou lokalitu s uloženou. Při skutečné změně může odstranit staré odvozené části adresy nebo staré souřadnice, aby se k nové adrese nepřenesly neplatné údaje.

Pokud aktualizační payload žádné lokalitní pole neobsahuje, původní lokalita zůstává zachována.

## Číselníky

Následující tabulky popisují hodnoty skutečně mapované aktuálním kódem Domonautu.

### `advert_function`

| Kód | Hodnota |
| ---: | --- |
| `1` | Prodej |
| `2` | Pronájem |
| `3` | Dražba |
| `4` | Podíl |

### `advert_type`

| Kód | Hodnota |
| ---: | --- |
| `1` | Byt |
| `2` | Dům |
| `3` | Pozemek |
| `4` | Komerční prostor |
| `5` | Garáž nebo nebytový prostor |

U kódu `5` rozhoduje `advert_subtype`:

- `36` → Nebytový prostor,
- jiná nebo chybějící hodnota → Garáž.

### `advert_subtype` – byty

| Kód | Hodnota |
| ---: | --- |
| `2` | 1+kk |
| `3` | 1+1 |
| `4` | 2+kk |
| `5` | 2+1 |
| `6` | 3+kk |
| `7` | 3+1 |
| `8` | 4+kk |
| `9` | 4+1 |
| `10` | 5+kk |
| `11` | 5+1 |
| `12` | 6+ |
| `16` | Atypický |
| `47` | Pokoj |

### `advert_subtype` – domy

| Kód | Hodnota |
| ---: | --- |
| `33` | Chata |
| `35` | Památka |
| `37` | Rodinný dům |
| `39` | Vila |
| `40` | Na klíč |
| `43` | Chalupa |
| `44` | Zemědělská usedlost |
| `54` | Vícegenerační dům |

### `advert_subtype` – pozemky

| Kód | Hodnota |
| ---: | --- |
| `18` | Komerční |
| `19` | Bydlení |
| `20` | Pole |
| `21` | Les |
| `22` | Louka |
| `23` | Zahrada |
| `24` | Ostatní |
| `46` | Rybník |
| `48` | Sady a vinice |

### `advert_subtype` – komerční prostory

| Kód | Hodnota |
| ---: | --- |
| `25` | Kanceláře |
| `26` | Sklady |
| `27` | Výroba |
| `28` | Obchodní prostor |
| `29` | Ubytování |
| `30` | Restaurace |
| `31` | Zemědělský |
| `32` | Ostatní |
| `38` | Činžovní dům |
| `56` | Ordinace |
| `57` | Apartmány |

### `advert_subtype` – ostatní

| Kód | Hodnota |
| ---: | --- |
| `34` | Garáž |
| `36` | Nebytový prostor / ostatní |

### `building_condition`

| Kód | Hodnota |
| ---: | --- |
| `1` | Velmi dobrý |
| `2` | Dobrý |
| `3` | Špatný |
| `4` | Probíhá výstavba |
| `5` | Projekt |
| `6` | Novostavba |
| `9` | Dokončená rekonstrukce |
| `10` | Probíhá rekonstrukce |

Nepodporovaný kód se při aktualizaci nepromítne do uložené hodnoty.

### `building_type`

| Kód | Hodnota v Domonautu |
| ---: | --- |
| `1` | Dřevo |
| `2` | Cihla |
| `3` | Kámen |
| `4` | Montovaná |
| `5` | Panel |
| `6` | Skeletová |
| `7` | Smíšená |
| `8` | Modulární |

### `ownership`

| Kód | Hodnota |
| ---: | --- |
| `1` | Osobní |
| `2` | Družstevní |
| `3` | Státní |

### `furnished`

| Kód | Hodnota |
| ---: | --- |
| `1` | Vybavený |
| `2` | Nevybavený |
| `3` | Částečně vybavený |

### `energy_efficiency_rating`

| Kód | Třída |
| ---: | --- |
| `1` | A |
| `2` | B |
| `3` | C |
| `4` | D |
| `5` | E |
| `6` | F |
| `7` | G |

## Fotografie nabídek

### `addPhoto`

```text
addPhoto(session_id, advert_id, advert_rkid, data)
```

Pro běžné volání podle externího ID:

```text
addPhoto(session_id, 0, "offer-987654", data)
```

Pokud je vyplněné `advert_rkid`, má při hledání nabídky přednost před `advert_id`.

#### Struktura `data`

| Pole | Typ | Chování |
| --- | --- | --- |
| `photo_rkid` | `string` | Volitelný stabilní identifikátor fotografie |
| `data` | `base64` | Preferované pole s obrazovými daty |
| `binary` | `base64` | Kompatibilní alternativní název |
| `file` | `base64` | Kompatibilní alternativní název |
| `order` | `int` nebo číselný `string` | Volitelné pořadí |

Server nevyužívá vstupní pole `main`, `alt`, `photo_kind` ani `room_type`.

#### Normalizace `photo_rkid`

- případná přípona souboru se odstraní,
- zachovají se pouze znaky `A-Z`, `a-z`, `0-9`, `_` a `-`,
- pokud identifikátor chybí, server vytvoří další číselnou hodnotu podle existujících fotografií,
- stejné `photo_rkid` nahraje nový obsah na místo původního souboru.

#### Uložení

1. Base64 se dekóduje; mezery a nové řádky jsou tolerované.
2. Obraz se zpracuje interním `resizeImage` flow.
3. Hlavní soubor se uloží interně jako WebP.
4. Odstraní se případný starý JPG stejného jména.
5. Znovu se vytvoří náhledy a pořadí galerie.

Přesto API kvůli kompatibilitě vrací `photo_rkid` s příponou `.jpg`.

#### Výstup

```json
{
  "status": 200,
  "statusMessage": "OK",
  "output": [
    {
      "photo_id": 1,
      "photo_rkid": "1.jpg"
    }
  ]
}
```

`photo_id` je číslo, pokud je normalizované `photo_rkid` číselné; jinak může být řetězec.

Možné chyby:

- `404` nabídka nebyla nalezena,
- `407` neplatná session,
- `452` chybí fotografická data nebo base64 nelze dekódovat,
- `500` nelze vytvořit adresář, zpracovat obraz nebo zapsat soubor.

### `listPhoto`

```text
listPhoto(session_id, advert_id, advert_rkid)
```

Vrací fotografie v pořadí galerie.

| Pole | Typ | Popis |
| --- | --- | --- |
| `photo_id` | `int` nebo `string` | Název fotografie bez přípony |
| `photo_rkid` | `string` | Kompatibilní hodnota s příponou `.jpg` |
| `main` | `int` | První fotografie má `1`, ostatní `0` |
| `order` | `int` | Pořadí od `1` |

Pokud nabídka neexistuje, metoda vrátí `404`.

### `delPhoto`

Původní dokumentace Sreality definuje signaturu:

```text
delPhoto(session_id, photo_id, photo_rkid)
```

Domonaut však ukládá fotografie v adresáři konkrétní nabídky, a proto musí být nabídka známá nebo odvoditelná.

Doporučené volání pro Domonaut a používané kompatibilní exportéry je:

```text
delPhoto(session_id, photo_rkid, advert_rkid)
```

Použijte `photo_rkid` vrácené metodou `addPhoto` nebo `listPhoto`, tedy obvykle hodnotu s příponou `.jpg`:

```text
delPhoto(session_id, "photo-001.jpg", "offer-987654")
```

Endpoint je tolerantní také k rozšířeným variantám s `advert_id` nebo `advert_rkid` na dalších pozicích parametrů. Pokud identifikace nabídky chybí, pokusí se `advert_rkid` odvodit z `photo_rkid` ve formátu:

```text
<advert_rkid>_<photo>
<advert_rkid>-<photo>
```

Tento fallback bere pouze jednoduchý alfanumerický prefix před prvním `_` nebo `-`. Není proto spolehlivý, pokud samotné `advert_rkid` obsahuje oddělovače. V produkční integraci vždy posílejte nabídku explicitně.

Při odstranění:

- se smaže WebP, JPG nebo JPEG odpovídajícího jména,
- odstraní se záznam pořadí,
- znovu se vytvoří náhledy,
- neexistující soubor u existující nabídky nevyvolá chybu.

Pokud nelze určit nabídku, metoda vrátí `404`.

## Statistiky

Domonaut poskytuje návštěvnost z tabulky `offer_views`. Finanční hodnoty kompatibilní se Sreality nejsou v tomto endpointu účtovány a vracejí se jako `0.0`; `with_vat` se vrací jako `1`.

### `listStat`

```text
listStat(session_id, advert_id_array, advert_rkid_array)
```

Parametry mohou být XML-RPC pole nebo jednotlivá skalární hodnota; skalární hodnota se interně převede na jednoprvkové pole.

Chování filtrů:

- pokud obsahuje platné hodnoty `advert_id`, filtruje se podle nich a `advert_rkid` se ignoruje,
- jinak se filtruje podle neprázdných `advert_rkid`,
- pokud jsou obě pole prázdná, vrátí se statistiky všech nabídek RK, včetně archivovaných.

Výstup:

| Pole | Typ | Popis |
| --- | --- | --- |
| `advert_id` | `int` | Interní ID nabídky |
| `rkid` | `string` | `advert_rkid` |
| `total_views` | `int` | Celkový počet uložených zhlédnutí |
| `total_price` | `double` | Vždy `0.0` |
| `advert_code` | `string` | Interní číslo zakázky RK nebo prázdný řetězec |
| `topped_price` | `double` | Vždy `0.0` |
| `advert_price` | `double` | Vždy `0.0` |
| `top` | `int` | `1`, pokud je u nabídky uložený údaj o topování, jinak `0` |
| `with_vat` | `int` | Vždy `1` |

### `listAllDailyStat`

```text
listAllDailyStat(session_id, date)
```

Vrátí statistiku všech nabídek RK, které byly aktivní alespoň po část zadaného dne. Nabídky bez zhlédnutí se vracejí s `views = 0`.

Podporované vstupy data:

- XML-RPC `datetime`,
- `YYYY-MM-DD`,
- `YYYYMMDD` nebo delší hodnota začínající tímto datem.

Výstup:

| Pole | Typ | Popis |
| --- | --- | --- |
| `advert_id` | `int` | Interní ID nabídky |
| `rkid` | `string` | `advert_rkid` |
| `views` | `int` | Počet zhlédnutí vytvořených v daném dni |
| `advert_price` | `double` | `0.0` |
| `topped_price` | `double` | `0.0` |
| `total_price` | `double` | `0.0` |
| `with_vat` | `int` | `1` |

### `listSellerStat`

```text
listSellerStat(session_id, seller_id, seller_rkid, from, till)
```

Vrátí denní statistiku jednoho makléře za uzavřený interval včetně obou krajních dnů.

- Makléře identifikujte jedním z `seller_id` / `seller_rkid`.
- `from` a `till` používají stejné datumové formáty jako `listAllDailyStat`.
- Pokud `till < from`, metoda vrátí `452`.
- Pokud makléř neexistuje, vrátí `461`.

Každý den obsahuje:

| Pole | Typ | Popis |
| --- | --- | --- |
| `date` | `string` | `YYYY-MM-DD` |
| `advert_count` | `int` | Počet nabídek makléře aktivních v daném dni |
| `views` | `int` | Zhlédnutí nabídek makléře v daném dni |
| `advert_price` | `double` | `0.0` |
| `topped_price` | `double` | `0.0` |
| `total_price` | `double` | `0.0` |
| `with_vat` | `int` | `1` |

## Poptávky

### `listFullInquiry`

```text
listFullInquiry(session_id, date)
```

Na rozdíl od signatury uvedené v některých verzích dokumentace Sreality vyžaduje Domonaut jako první parametr platné rotované `session_id`.

Datum lze poslat jako:

- řetězec začínající `YYYY-MM-DD`,
- řetězec začínající `YYYYMMDD`,
- XML-RPC datetime reprezentovaný parserem jako struktura s `xmlrpc_datetime`.

Metoda vrátí zprávy z chatu odeslané k nabídkám aktuální RK v daném kalendářním dni.

Každý prvek obsahuje:

| Pole | Typ | Aktuální chování |
| --- | --- | --- |
| `inquiry_id` | `int` | Interní ID zprávy |
| `date` | `string` | Datum a čas zprávy |
| `email` | `string` | E-mail odesílatele nebo prázdný řetězec |
| `name` | `string` | Jméno a příjmení odesílatele |
| `phone` | `string` | Telefon odesílatele nebo prázdný řetězec |
| `message` | `string` | Text zprávy |
| `advert_id` | `int` | Interní ID nabídky |
| `rkid` | `string` | V aktuální implementaci interní ID nabídky převedené na řetězec, tedy stejná číselná hodnota jako `advert_id` |

Pokud odesílatel už neexistuje nebo nejsou kontaktní údaje dostupné, kontaktní pole mohou být prázdná.

## Doporučená synchronizace

### Prvotní synchronizace

```text
register.php
getHash
login

pro každého makléře:
    rotace session
    addSeller

pro každou nabídku:
    rotace session
    addAdvert

pro každou fotografii:
    rotace session
    addPhoto

rotace session
logout
```

### Průběžná synchronizace

```text
getHash
login

změněný nebo nový makléř:
    addSeller se stejným seller_rkid

změněná nebo nová nabídka:
    addAdvert se stejným advert_rkid

změny fotografií:
    listPhoto podle potřeby
    addPhoto pro nové nebo změněné fotografie
    delPhoto pro odstraněné fotografie

nabídka odstraněná ve zdrojovém systému:
    delAdvert

logout
```

### Doporučení pro exportér

- Udržujte stabilní `rpcRkid`, `seller_rkid`, `advert_rkid` a `photo_rkid`.
- Neodvozujte externí ID z interního `advert_id` Domonautu.
- Aktualizace nabídky provádějte výhradně přes opakovaný `addAdvert`.
- Ve stejné session neposílejte paralelní požadavky.
- Před každým chráněným voláním, včetně `logout`, zrotujte session.
- Při `407` zahajte novou session od `getHash`.
- Po aplikační chybě jiné než `407` počítejte s tím, že session token už byl spotřebován, a pro další požadavek proveďte další rotaci.
- Pokud se ztratí HTTP odpověď a nelze zjistit, zda server požadavek přijal, zahajte novou session; nesnažte se pokračovat od nejistého tokenu.
- Pro bezpečné opakování po síťové chybě používejte stejný `advert_rkid` nebo `photo_rkid`.
- Registrační požadavek po síťové chybě neopakujte bez rozmyslu; mohl již zrotovat secret.
- Pro finanční údaje posílejte strukturovaná pole místo spoléhání na textový popis.
- Pro lokalitu posílejte RÚIAN a současně co nejúplnější textovou adresu.
- Fotografie posílejte s explicitním stabilním `photo_rkid` a pořadím.
- Používejte HTTPS a credentials neukládejte do veřejného repozitáře.

## Retry a idempotence

| Operace | Bezpečné opakování se stejným ID |
| --- | --- |
| `addSeller` | Ano; existující makléř se aktualizuje |
| `addAdvert` | Ano; existující nabídka se aktualizuje |
| `addPhoto` | Ano, pokud použijete stejné `photo_rkid`; obsah se nahradí |
| `delAdvert` | Ano; nenalezená nabídka vrací `200` |
| `delPhoto` | Ano pro existující nabídku; chybějící soubor se toleruje |
| registrace RK | Ne z hlediska credentials; každé úspěšné opakování rotuje secret |

Po `500` není vždy možné z odpovědi určit, zda byla část operace dokončena. U upsert metod je proto správný postup znovu navázat session a operaci zopakovat se stejným externím identifikátorem.

## Kompatibilita se Sreality

Domonaut zachovává zejména:

- XML-RPC transport a strukturu odpovědi,
- rotující `session_id`,
- dvojice interních a externích identifikátorů,
- metody `getHash`, `login`, `logout`, `addSeller`, `delSeller`, `listSeller`, `addAdvert`, `delAdvert`, `listAdvert`, foto metody a statistiky,
- číselníky používané u podporovaných polí.

Důležité odlišnosti aktuální implementace:

1. `addAdvert` je create/update upsert a je jedinou doporučenou metodou pro změny nabídky.
2. `updateAdvert` je pouze legacy kompatibilita mimo veřejný kontrakt.
3. `addSeller` v Domonautu také funguje jako upsert.
4. `updateSeller` je Domonaut rozšíření a není nutné pro běžnou integraci.
5. `listProject` je alias `listAdvert`, nikoli samostatná správa developerských projektů.
6. `listAdvert` vrací jen aktivní nabídky a navíc obsahuje `advert_subtype`.
7. Finanční statistické položky se vracejí jako nuly.
8. `delPhoto` potřebuje určit nabídku nebo ji odvodit z `photo_rkid`; doporučená tříparametrová varianta je `delPhoto(session_id, photo_rkid, advert_rkid)`.
9. `listFullInquiry` vyžaduje session jako první parametr.
10. Domonaut ignoruje kompatibilní vstupní pole, která nejsou uvedena v této dokumentaci.
11. Domonaut nepoužívá obecné Sreality statusy pro nejednoznačnou lokalitu; snaží se lokalitu doplnit a nabídku uložit i s fallbackem.

## Bezpečnost

- Registrační Bearer token, `secret`, `password_md5` a `software_key` jsou citlivé údaje.
- Neukládejte je do veřejného Git repozitáře, logů ani klientského JavaScriptu.
- Secret ukládejte v secret manageru nebo serverové konfiguraci s omezenými právy.
- Při podezření na únik registračního tokenu ho okamžitě nahraďte.
- Při podezření na únik secretu proveďte kontrolovanou opakovanou registraci RK a distribuujte nový secret exportéru.
- Nepoužívejte registrační endpoint jako běžný health check, protože úspěšné volání rotuje credentials.

## Podpora

Pro přístup k integraci, přidělení registračního tokenu nebo software key a technickou podporu použijte kontaktní formulář:

[https://domonaut.cz/kontakt](https://domonaut.cz/kontakt)

## Copyright

© Roman Rusianowski

Tato dokumentace je poskytována pouze pro účely integrace s Domonaut.cz. Bez předchozího písemného souhlasu není udělena licence ke kopírování, úpravě, dalšímu šíření nebo opětovnému použití této dokumentace mimo integrace s Domonaut.cz.
