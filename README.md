# Domonaut XML-RPC importní API

Domonaut.cz je český realitní portál, který podporuje automatický import nabídek z realitních systémů. XML-RPC rozhraní slouží pro export realitních kanceláří, makléřů, inzerátů a fotografií.

Web
https://domonaut.cz

## Přehled

Domonaut XML-RPC API umožňuje externím realitním systémům publikovat a aktualizovat nabídky na Domonaut.cz.

Integrace probíhá jako aktivní export z externího systému do Domonautu. Externí systém odesílá makléře, inzeráty a fotografie pomocí XML-RPC volání.

Rozhraní je navrženo tak, aby bylo kompatibilní s běžně používanými exporty typu Sreality.

## Endpoint

Produkční endpoint

```text
https://domonaut.cz/xmlrpc/sreality.php
```

Transport

```text
HTTP POST
```

Content type

```text
text/xml; charset=utf-8
```

Endpoint vrací standardní XML-RPC `methodResponse`.

Aplikační chyby jsou vracené ve struktuře odpovědi. HTTP odpověď může být i při aplikační chybě `200 OK`.

## Formát odpovědi

Každá metoda vrací strukturu v tomto formátu.

```json
{
  "status": 200,
  "statusMessage": "OK",
  "output": []
}
```

Běžné stavové kódy

| Kód | Význam            | Popis                                              |
| --- | ----------------- | -------------------------------------------------- |
| 200 | OK                | Požadavek byl úspěšný                              |
| 402 | Client unknown    | Neznámé `client_id` nebo chybějící credentials     |
| 404 | Not found         | Požadovaný objekt nebyl nalezen                    |
| 405 | Software key      | Neplatný nebo chybějící software key               |
| 407 | Bad session       | Neplatná, expirovaná nebo špatně zrotovaná session |
| 409 | Conflict          | Inzerát už pro danou RK existuje                   |
| 452 | Invalid params    | Chybí povinná pole nebo mají neplatný formát       |
| 461 | Seller not found  | Makléř nebyl nalezen                               |
| 462 | Seller login used | Login makléře je už použitý                        |
| 500 | Internal error    | Chyba na straně serveru                            |

## Autentizace

Autentizace používá dvoufázový session flow.

Nejdřív se zavolá `getHash(client_id)`, které vrátí počáteční session token.

Poté se token zrotuje a zavolá se `login(sessionId)`.

Po každém úspěšném XML-RPC volání musí klient session token znovu zrotovat a až poté provést další volání.

Potřebné údaje

| Pole           | Popis                                             |
| -------------- | ------------------------------------------------- |
| `client_id`    | ID realitní kanceláře v Domonautu                 |
| `secret`       | Sdílený tajný klíč vygenerovaný při registraci RK |
| `password_md5` | `md5(secret)`                                     |
| `software_key` | Klíč konkrétní integrace poskytnutý Domonautem    |

Algoritmus rotace session

```php
$fixedPart = substr($currentSessionId, 0, 48);
$varPart = md5($currentSessionId . $password_md5 . $software_key);
$nextSessionId = $fixedPart . $varPart;
```

Typický průběh

```text
getHash(client_id)
rotace sessionId
login(sessionId)
rotace sessionId
addSeller(...)
rotace sessionId
addAdvert(...)
rotace sessionId
addPhoto(...)
```

Výchozí TTL session je 900 sekund. Hodnota se může lišit podle konkrétní integrace.

Pokud API vrátí `407 Bad session`, opakujte autentizaci od `getHash`.

## Metody

### getHash

Založí novou session pro zadaného klienta.

Parametry

| Název       | Typ | Popis                             |
| ----------- | --- | --------------------------------- |
| `client_id` | int | ID realitní kanceláře v Domonautu |

Výstup

| Název       | Typ    | Popis                   |
| ----------- | ------ | ----------------------- |
| `sessionId` | string | Počáteční session token |

### login

Autorizuje session.

Před voláním `login` je nutné zrotovat `sessionId` vrácené metodou `getHash`.

Parametry

| Název       | Typ    | Popis                   |
| ----------- | ------ | ----------------------- |
| `sessionId` | string | Zrotovaný session token |

### logout

Ukončí aktuální session.

Parametry

| Název       | Typ    | Popis                          |
| ----------- | ------ | ------------------------------ |
| `sessionId` | string | Aktuální validní session token |

### addSeller

Vytvoří nebo aktualizuje makléře.

Identifikace makléře probíhá podle `seller_id` nebo `seller_rkid`. Pro importy je doporučené používat `seller_rkid` jako stabilní identifikátor makléře z externího systému.

Parametry

| Název         | Typ    | Popis                                                                 |
| ------------- | ------ | --------------------------------------------------------------------- |
| `sessionId`   | string | Aktuální validní session token                                        |
| `seller_id`   | int    | Volitelné interní ID makléře. Při použití `seller_rkid` posílejte `0` |
| `seller_rkid` | string | Externí ID nebo login makléře                                         |
| `clientData`  | struct | Data makléře                                                          |

Podporovaná pole `clientData`

| Pole            | Typ    | Popis                          |
| --------------- | ------ | ------------------------------ |
| `client_name`   | string | Jméno makléře                  |
| `contact_email` | string | E-mail makléře                 |
| `client_login`  | string | Alternativní e-mail nebo login |
| `contact_gsm`   | string | Mobilní telefon                |
| `contact_phone` | string | Alternativní telefon           |
| `client_ic`     | string | IČO                            |
| `photo`         | base64 | Volitelná fotografie makléře   |

Výstup

| Pole          | Typ    | Popis                  |
| ------------- | ------ | ---------------------- |
| `seller_id`   | int    | ID makléře v Domonautu |
| `seller_rkid` | string | Externí ID makléře     |

### updateSeller

Aktualizuje existujícího makléře.

Parametry jsou stejné jako u metody `addSeller`.

### delSeller

Smaže makléře a archivuje jeho inzeráty.

Parametry

| Název         | Typ    | Popis                                   |
| ------------- | ------ | --------------------------------------- |
| `sessionId`   | string | Aktuální validní session token          |
| `seller_id`   | int    | ID makléře nebo `0`                     |
| `seller_rkid` | string | Externí ID makléře nebo prázdný řetězec |

### listSeller

Vrátí seznam makléřů patřících k aktuální RK.

Parametry

| Název       | Typ    | Popis                          |
| ----------- | ------ | ------------------------------ |
| `sessionId` | string | Aktuální validní session token |

Výstup obsahuje struktury makléřů s poli jako `seller_id`, `seller_rkid`, `client_name`, `client_login` a `photo`.

### addAdvert

Vytvoří nový inzerát.

Používejte stabilní `advert_rkid`, které je unikátní v rámci jedné realitní kanceláře.

Pokud inzerát už existuje, API vrátí `409 Conflict`. V takovém případě exportér nemá donekonečna opakovat stejný `addAdvert`. Pro existující inzerát použijte `updateAdvert`.

Parametry

| Název        | Typ    | Popis                          |
| ------------ | ------ | ------------------------------ |
| `sessionId`  | string | Aktuální validní session token |
| `advertData` | struct | Data inzerátu                  |

Výstup

| Pole        | Typ | Popis                   |
| ----------- | --- | ----------------------- |
| `advert_id` | int | ID inzerátu v Domonautu |

### updateAdvert

Aktualizuje existující inzerát podle `advert_rkid` nebo `advert_id`.

Pokud byl inzerát archivovaný, `updateAdvert` ho znovu aktivuje.

Parametry

| Název        | Typ    | Popis                          |
| ------------ | ------ | ------------------------------ |
| `sessionId`  | string | Aktuální validní session token |
| `advertData` | struct | Data inzerátu                  |

### delAdvert

Archivuje inzerát a odstraní uložené fotografie daného inzerátu.

Parametry

| Název         | Typ    | Popis                             |
| ------------- | ------ | --------------------------------- |
| `sessionId`   | string | Aktuální validní session token    |
| `advert_id`   | int    | Volitelné ID inzerátu v Domonautu |
| `advert_rkid` | string | Volitelné externí ID inzerátu     |

### listAdvert

Vrátí aktivní inzeráty aktuální RK.

Parametry

| Název       | Typ    | Popis                          |
| ----------- | ------ | ------------------------------ |
| `sessionId` | string | Aktuální validní session token |

Výstup obsahuje struktury inzerátů s poli jako `advert_id`, `advert_rkid`, `advert_type`, `advert_subtype`, `advert_url`, `hash_id`, `modified`, `published`, `published_status` a `top`.

### listProject

Kompatibilní alias pro `listAdvert`.

### addPhoto

Nahraje fotografii k inzerátu.

Data fotografie se posílají jako base64 řetězec s binárním obsahem obrázku.

Parametry

| Název         | Typ    | Popis                          |
| ------------- | ------ | ------------------------------ |
| `sessionId`   | string | Aktuální validní session token |
| `advert_id`   | int    | ID inzerátu v Domonautu        |
| `advert_rkid` | string | Externí ID inzerátu            |
| `data`        | struct | Data fotografie                |

Podporovaná pole `data`

| Pole         | Typ    | Popis                            |
| ------------ | ------ | -------------------------------- |
| `photo_rkid` | string | Volitelné externí ID fotografie  |
| `data`       | base64 | Binární data fotografie          |
| `binary`     | base64 | Alternativní pole pro fotografii |
| `file`       | base64 | Alternativní pole pro fotografii |
| `order`      | int    | Volitelné pořadí fotografie      |

Výstup

| Pole         | Typ    | Popis                              |
| ------------ | ------ | ---------------------------------- |
| `photo_id`   | string | ID fotografie v Domonautu          |
| `photo_rkid` | string | Kompatibilní externí ID fotografie |

### listPhoto

Vrátí fotografie přiřazené k inzerátu.

Parametry

| Název         | Typ    | Popis                                    |
| ------------- | ------ | ---------------------------------------- |
| `sessionId`   | string | Aktuální validní session token           |
| `advert_id`   | int    | ID inzerátu v Domonautu nebo `0`         |
| `advert_rkid` | string | Externí ID inzerátu nebo prázdný řetězec |

Výstup obsahuje struktury fotografií s poli `photo_id`, `photo_rkid`, `main` a `order`.

### delPhoto

Smaže fotografii u inzerátu.

Doporučená signatura

```text
delPhoto(sessionId, photo_rkid, advert_rkid)
```

Server je z důvodu kompatibility tolerantní k různému pořadí parametrů. Pokud `advert_rkid` chybí, server se ho může pokusit odvodit z `photo_rkid` ve formátu `<advert>_<photo>`.

### listFullInquiry

Vrátí poptávky pro aktuální RK za zvolený den.

Parametry

| Název       | Typ                | Popis                          |
| ----------- | ------------------ | ------------------------------ |
| `sessionId` | string             | Aktuální validní session token |
| `date`      | string nebo struct | Datum ve formátu `YYYY-MM-DD`  |

Výstup obsahuje struktury poptávek s poli jako `inquiry_id`, `date`, `email`, `name`, `phone`, `message`, `advert_id` a `rkid`.

## advertData

Minimální doporučená pole pro `addAdvert`

| Pole                 | Typ    | Popis                        |
| -------------------- | ------ | ---------------------------- |
| `advert_rkid`        | string | Stabilní externí ID inzerátu |
| `advert_function`    | int    | Typ nabídky                  |
| `advert_type`        | int    | Typ nemovitosti              |
| `description`        | string | Popis inzerátu               |
| `seller_id`          | int    | ID makléře                   |
| `seller_rkid`        | string | Externí ID makléře           |
| `locality_city`      | string | Město                        |
| `locality_latitude`  | double | Zeměpisná šířka              |
| `locality_longitude` | double | Zeměpisná délka              |

Musí být uvedeno alespoň `seller_id` nebo `seller_rkid`.

Lokalitu je vhodné posílat pomocí města, souřadnic nebo RÚIAN údajů, pokud jsou dostupné.

Podporovaná pole inzerátu

| Pole                        | Typ                | Popis                                   |
| --------------------------- | ------------------ | --------------------------------------- |
| `advert_id`                 | int                | Volitelné ID inzerátu v Domonautu       |
| `advert_rkid`               | string             | Externí ID inzerátu                     |
| `advert_function`           | int                | 1 prodej, 2 pronájem, 3 dražba, 4 podíl |
| `advert_type`               | int                | Typ nemovitosti                         |
| `advert_subtype`            | int                | Sreality kompatibilní podtyp            |
| `description`               | string             | Popis inzerátu                          |
| `advert_price`              | double             | Cena                                    |
| `advert_price_currency`     | int                | 3 znamená EUR, jinak CZK                |
| `advert_price_unit`         | int                | 3 nebo 4 znamená cenu za metr čtvereční |
| `advert_price_text_note`    | string             | Poznámka k ceně                         |
| `cost_of_living`            | string nebo double | Měsíční poplatky                        |
| `poplatky`                  | int                | Měsíční poplatky                        |
| `refundable_deposit`        | double             | Kauce                                   |
| `commission`                | string nebo double | Provize                                 |
| `tenant_not_pay_commission` | int                | 1 znamená, že nájemce neplatí provizi   |
| `building_condition`        | int                | Stav objektu                            |
| `building_type`             | int                | Typ konstrukce                          |
| `ownership`                 | int                | Typ vlastnictví                         |
| `furnished`                 | int                | Vybavenost                              |
| `energy_efficiency_rating`  | int                | Energetická třída                       |
| `usable_area`               | int                | Užitná plocha v metrech čtverečních     |
| `estate_area`               | int                | Plocha pozemku v metrech čtverečních    |
| `floor_number`              | int                | Podlaží                                 |
| `floors`                    | int                | Celkový počet podlaží                   |
| `advert_room_count`         | int                | Počet pokojů u domu                     |
| `ready_date`                | string             | Datum nastěhování                       |
| `locality_cp`               | string             | Číslo popisné nebo orientační           |
| `locality_street`           | string             | Ulice                                   |
| `locality_city`             | string             | Město                                   |
| `locality_citypart`         | string             | Část obce                               |
| `locality_psc`              | string             | PSČ                                     |
| `locality_latitude`         | double             | Zeměpisná šířka                         |
| `locality_longitude`        | double             | Zeměpisná délka                         |
| `locality_ruian`            | int                | Volitelný RÚIAN kód                     |
| `locality_ruian_level`      | int                | Volitelná RÚIAN úroveň                  |

Podporované formáty `ready_date`

```text
YYYYMMDDTHHMMSS
YYYY-MM-DD
```

## Mapování hodnot

### advert_function

| Kód | Význam   |
| --- | -------- |
| 1   | Prodej   |
| 2   | Pronájem |
| 3   | Dražba   |
| 4   | Podíl    |

### advert_type

| Kód | Typ nemovitosti             |
| --- | --------------------------- |
| 1   | Byt                         |
| 2   | Dům                         |
| 3   | Pozemek                     |
| 4   | Komerční prostor            |
| 5   | Garáž nebo nebytový prostor |

### Běžné hodnoty advert_subtype

| Typ nemovitosti  | Kód | Význam           |
| ---------------- | --- | ---------------- |
| Byt              | 2   | 1+kk             |
| Byt              | 4   | 2+kk             |
| Byt              | 6   | 3+kk             |
| Byt              | 8   | 4+kk             |
| Dům              | 37  | Rodinný dům      |
| Dům              | 33  | Chata            |
| Pozemek          | 19  | Bydlení          |
| Pozemek          | 18  | Komerční         |
| Komerční prostor | 25  | Kanceláře        |
| Komerční prostor | 28  | Obchodní prostor |
| Komerční prostor | 26  | Sklady           |
| Garáž            | 34  | Garáž            |
| Nebytový prostor | 36  | Ostatní          |

### building_condition

| Kód | Význam                 |
| --- | ---------------------- |
| 1   | Velmi dobrý            |
| 2   | Dobrý                  |
| 3   | Špatný                 |
| 4   | Probíhá výstavba       |
| 5   | Projekt                |
| 6   | Novostavba             |
| 9   | Dokončená rekonstrukce |
| 10  | Probíhá rekonstrukce   |

### building_type

| Kód | Význam    |
| --- | --------- |
| 1   | Dřevo     |
| 2   | Cihla     |
| 3   | Kámen     |
| 4   | Montovaná |
| 5   | Panel     |
| 6   | Skeletová |
| 7   | Smíšená   |
| 8   | Modulární |

### ownership

| Kód | Význam     |
| --- | ---------- |
| 1   | Osobní     |
| 2   | Družstevní |
| 3   | Státní     |

### furnished

| Kód | Význam            |
| --- | ----------------- |
| 1   | Vybavený          |
| 2   | Nevybavený        |
| 3   | Částečně vybavený |

### energy_efficiency_rating

| Kód | Třída |
| --- | ----- |
| 1   | A     |
| 2   | B     |
| 3   | C     |
| 4   | D     |
| 5   | E     |
| 6   | F     |
| 7   | G     |

## Registrace realitní kanceláře

Pro vytvoření `client_id` a `secret` poskytuje Domonaut JSON registrační endpoint.

Endpoint

```text
https://domonaut.cz/xmlrpc/register.php
```

Metoda

```text
HTTP POST
```

Autorizace

```text
Authorization: Bearer <REGISTER_TOKEN>
```

Content type

```text
application/json
```

Příklad požadavku

```json
{
  "rpcRkid": "agency-123",
  "name": "Ukázková realitní kancelář",
  "ico": "12345678",
  "email": "info@example.com",
  "phone": "+420700000000",
  "website": "https://example.com",
  "aboutMe": "Krátký popis realitní kanceláře"
}
```

Příklad odpovědi

```json
{
  "status": "ok",
  "data": {
    "id": 123,
    "secret": "generated-secret"
  }
}
```

`secret` je vrácen pouze v odpovědi při registraci. Uložte ho bezpečně.

Pro rotaci XML-RPC session použijte

```text
password_md5 = md5(secret)
```

Opakovaná registrace se stejným `rpcRkid` v rámci stejného integračního zdroje aktualizuje údaje RK a vygeneruje nový secret.

Integrační zdroj se odvozuje z bearer tokenu. V JSON požadavku se neposílá přímo.

## Doporučený synchronizační postup

Prvotní synchronizace

```text
getHash
login
addSeller nebo updateSeller pro všechny makléře
addAdvert nebo updateAdvert pro všechny inzeráty
addPhoto pro fotografie inzerátů
logout
```

Průběžná synchronizace

```text
getHash
login
updateSeller při změně údajů makléře
updateAdvert při změně inzerátu
addPhoto nebo delPhoto při změně fotografií
delAdvert při archivaci inzerátu
logout
```

## Doporučení pro exportéry

Používejte stabilní hodnoty `seller_rkid`.

Používejte stabilní hodnoty `advert_rkid`.

`advert_rkid` musí být unikátní v rámci jedné realitní kanceláře.

Pro existující inzeráty používejte `updateAdvert`.

Po odpovědi `409 Conflict` neopakujte donekonečna stejný `addAdvert`.

Fotografie posílejte jako skutečná binární obrazová data zakódovaná v base64.

Pokud máte dostupné souřadnice, posílejte `locality_latitude` a `locality_longitude`.

Používejte HTTPS.

Při odpovědi `407 Bad session` spusťte autentizaci znovu od `getHash`.

Identifikátory udržujte jednoduché. Doporučený formát jsou alfanumerické ASCII hodnoty bez diakritiky a bez speciálních znaků.

## Kompatibilita

API je kompatibilní se stylem exportu Sreality, ale není nutné posílat všechna pole používaná Sreality.

Nepodporovaná nebo neznámá pole může Domonaut ignorovat.

Některá pole jsou před uložením normalizována.

Domonaut umí z textu inzerátu odvodit některé atributy pronájmu, například bez provize nebo bez kauce. Pokud jsou dostupná strukturovaná pole, je lepší posílat přímo ta.

## Podpora

Pro přístup k integraci, credentials nebo technickou podporu kontaktujte Domonaut.cz.

Web
https://domonaut.cz

## Copyright

© Domonaut.cz

Tato dokumentace je poskytována pouze pro účely integrace s Domonaut.cz. Bez předchozího písemného souhlasu není udělena licence ke kopírování, úpravě, dalšímu šíření nebo opětovnému použití této dokumentace mimo integrace s Domonaut.cz.
