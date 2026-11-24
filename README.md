# KoloStestiGroub
# Kolo štěstí & Simulovaná platební brána (REST API + Webhook)

Tento projekt demonstruje architekturu dvou nezávislých služeb, které spolu komunikují pomocí **REST API** (synchronní požadavky) a **Webhooků** (asynchronní události na pozadí).

## Architektura a rozdělení týmů

Projekt je rozdělen na **2 samostatné aplikace**, které běží na rozdílných portech (nebo serverech) a vyvíjejí je dvě nezávislé skupinky.

### 1. Skupinka A: Kolo štěstí (Hlavní aplikace – REST API + Frontend)

* **Výchozí adresa:** `http://localhost:3000` (nebo port `5000` dle stacku)
* **Odpovědnost:**
  * Správa uživatelů (registrace, přihlášení) a zůstatku kreditů (coinů).
  * Herní logika kola štěstí – **výpočet výhry probíhá výhradně na backendu** (ochrana proti podvodům v prohlížeči), ukládání historie zatočení.
  * Obchod s kredity (např. balíček *100 coinů za 50 Kč*).
  * Vytvoření objednávky dobití ve stavu `PENDING` a její odeslání na platební bránu.
  * **Webhook endpoint (`POST /api/webhooks/payment`)**, který přijímá potvrzení o platbě, ověřuje bezpečnostní podpis a připisuje kredity uživateli.

### 2. Skupinka B: Simulovaná platební brána (Externí služba – Webhook Provider)

* **Výchozí adresa:** `http://localhost:4000`
* **Odpovědnost:**
  * Příjem požadavku na založení platby od Kola štěstí (`POST /api/payments/init`).
  * Jednoduché webové rozhraní platební brány (zobrazení částky, výběr metody a tlačítka: **Zaplatit**, **Zrušit platbu**, **Simulovat chybu**).
  * **Odeslání Webhooku** na `callback_url` Kola štěstí po dokončení akce uživatelem.
  * **Živý log webhooků (Dashboard):** Tabulka/konzole přímo na webu brány, kde je vidět historie odeslaných webhooků, jejich JSON payload a HTTP status kód odpovědi (`200 OK`, `401 Unauthorized` apod.).

## Jak probíhá komunikace (Flow)

```
[Uživatel] ──(1. Chce 100 coinů)──> [Kolo štěstí (Skupina A)]
                                           │
                      (2. POST /api/payments/init + callback_url)
                                           ▼
[Uživatel] <──(3. Přesměrování)──── [Platební brána (Skupina B)]
    │
(4. Klikne na "Zaplatit")
    │
    ▼
[Platební brána] ──(5. WEBHOOK: POST /api/webhooks/payment)──> [Kolo štěstí]
    │                                                                │
    │ <──────────────────(6. HTTP 200 OK)────────────────────────────┘
    │                                                      (Připsání 100 coinů)
    └──(7. Redirect zpět na web kola)──> [Uživatel vidí kredity a točí]
```

## API Kontrakt (Dohodnuté rozhraní mezi týmy)

Aby mohly oba týmy pracovat nezávisle, je nutné dodržet následující formát dat.

### A) Založení platby (Kolo štěstí -> Platební brána)

Kolo štěstí zavolá endpoint brány, když uživatel klikne na „Dobít kredity“.

* **Endpoint:** `POST http://localhost:4000/api/payments/init`
* **Request Body (JSON):**

```json
{
  "order_id": "ord_1054",
  "user_id": 12,
  "amount": 50,
  "currency": "CZK",
  "credits_to_add": 100,
  "callback_url": "http://localhost:3000/api/webhooks/payment",
  "return_url": "http://localhost:3000/shop?order=ord_1054"
}
```

* **Response (HTTP `201 Created`):**

```json
{
  "transaction_id": "tx_998877",
  "payment_url": "http://localhost:4000/pay/tx_998877"
}
```

*(Kolo štěstí následně přesměruje prohlížeč uživatele na `payment_url`.)*

### B) Potvrzení platby – WEBHOOK (Platební brána -> Kolo štěstí)

Jakmile uživatel na bráně klikne na **Zaplatit** (nebo **Zrušit**), server brány odešle na pozadí HTTP `POST` požadavek na `callback_url`.

* **Endpoint:** `POST http://localhost:3000/api/webhooks/payment`
* **Hlavičky (Headers):**
  * `Content-Type: application/json`
  * `X-Webhook-Signature: <HMAC_SHA256_HASH>`
* **Request Body (JSON):**

```json
{
  "event": "payment.completed",
  "transaction_id": "tx_998877",
  "order_id": "ord_1054",
  "status": "PAID",
  "paid_at": "2026-11-24T15:10:00Z"
}
```

*(Možné hodnoty `status`: `"PAID"` nebo `"FAILED"`)*

* **Očekávaná odpověď od Kola štěstí:** HTTP `200 OK` (např. `{ "received": true }`).

## Klíčové bezpečnostní a technické prvky

1. **Ověření podpisu (HMAC SHA-256):**
   * Oba týmy mají v `.env` souboru uložený stejný tajný klíč: `WEBHOOK_SECRET="tajny_klic_123"`.
   * Platební brána vytvoří HMAC SHA-256 podpis z JSON těla požadavku pomocí tohoto klíče a pošle ho v hlavičce `X-Webhook-Signature`.
   * Kolo štěstí podpis přepočítá. Pokud nesedí, vrátí `401 Unauthorized` a kredity nepřipíše (ochrana proti falešným požadavkům z Postmana).

2. **Idempotence (Ochrana proti dvojímu připsání):**
   * Pokud brána pošle stejný webhook dvakrát (např. při opakovaném pokusu), Kolo štěstí zkontroluje stav `order_id` v databázi. Pokud už je ve stavu `PAID`, kredity znovu nepřičte a pouze vrátí `200 OK`.

3. **Retry mechanismus (Bonus pro Skupinku B):**
   * Pokud server Kola štěstí neodpoví statusem `200` (nebo je dočasně vypnutý), platební brána zkusí webhook odeslat znovu (např. po 5 sekundách, max. 3 pokusy).

## Kontrolní seznam (Checklist) před integrací

* [ ] **Společně:** Odsouhlasit názvy polí v JSONu a hodnotu `WEBHOOK_SECRET`.
* [ ] **Skupina A (Kolo):** Mít připravenou DB tabulku `orders` (`order_id`, `user_id`, `credits`, `status`).
* [ ] **Skupina A (Kolo):** Mít funkční endpoint `POST /api/webhooks/payment` otestovaný lokálně přes Postman/cURL.
* [ ] **Skupina B (Brána):** Mít funkční endpoint `POST /api/payments/init` a stránku `/pay/:transaction_id`.
* [ ] **Skupina B (Brána):** Implementovat odeslání `POST` webhooku s hlavičkou `X-Webhook-Signature` a výpis do logu na frontendu.
* [ ] **Integrace:** Propojit obě aplikace a otestovat celý průchod od nákupu po zatočení kolem.
