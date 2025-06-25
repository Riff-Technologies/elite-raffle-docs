
# Conventions

| Item           | Detail                                            |
|----------------|---------------------------------------------------|
| **Auth**       | Cognito JWT in `Authorization: Bearer …` header   |
| **IDs**        | UUID v4 unless noted                              |
| **Dates**      | ISO-8601, UTC                                     |
| **Pagination** | `page` (int, default 1), `limit` (int, default 50, max 200), `sort` (`field,[asc\|desc]`) |
| **Media-type** | All bodies are `application/json`                 |

---

## 1  System Management

| Method | Path | Query Params | Body (request) |
|--------|------|--------------|----------------|
| `GET`  | `/system/health` | — | — |
| `GET`  | `/system/verification` | `components` (CSV) | — |
| `GET`  | `/system/time` | — | — |
| `PATCH`| `/system/time/sync` | `strategy` (`ntp`\|`manual`) | ```json
{ "datetime": "2025-06-13T19:04:00Z" }
``` |
| `GET`  | `/system/config` | — | — |
| `PUT`  | `/system/config` | — | ```json
{ "settingKey": <value>, … }
``` |
| `GET`  | `/system/version` | — | — |
| `POST` | `/system/maintenance/enter` | `reason` | ```json
{ "until": "2025-06-14T02:00:00Z", "note": "DB patch window" }
``` |
| `POST` | `/system/maintenance/exit`  | — | ```json
{ "note": "Patch complete" }
``` |

---

## 2  Operator Management

| Method | Path | Query Params | Body |
|--------|------|--------------|------|
| `GET`  | `/operators` | `page`,`limit`,`sort`,`status`,`search` | — |
| `GET`  | `/operators/{operatorId}` | — | — |
| `POST` | `/operators` | — | ```json
{ "cognitoId": "uuid", "displayName": "Jane Doe", "roles": ["cashier"] }
``` |
| `PATCH`| `/operators/{operatorId}` | — | ```json
{ "displayName?": "...", "roles?": ["manager"] }
``` |
| `DELETE`| `/operators/{operatorId}` | — | — |
| `POST` | `/operators/{operatorId}/rsus/{rsuId}` | — | ```json
{ "action": "assign" }
``` |
| `DELETE`| `/operators/{operatorId}/rsus/{rsuId}` | — | — |
| `GET`  | `/operators/{operatorId}/activity` | `page`,`limit`,`sort`,`from`,`to` | — |
| `POST` | `/operators/auth/login` | — | ```json
{ "operatorId": "uuid", "ip": "203.0.113.5" }
``` |
| `POST` | `/operators/auth/logout` | — | ```json
{ "operatorId": "uuid" }
``` |
| `GET`  | `/operators/{operatorId}/permissions` | — | — |
| `PUT`  | `/operators/{operatorId}/permissions` | — | ```json
{ "permissions": ["ticket.sell","raffle.create"] }
``` |
| `GET`  | `/operators/search` | `query`,`page`,`limit` | — |

---

## 3  Raffle Management

| Method | Path | Query Params | Body |
|--------|------|--------------|------|
| `POST` | `/raffles` | — | ```json
{ "name": "50/50 Night", "start": "2025-07-01T23:00Z", "end": "2025-07-02T02:00Z", "ticketPrice": 10.0, "currency": "USD" }
``` |
| `GET`  | `/raffles` | `page`,`limit`,`sort`,`status`,`from`,`to` | — |
| `GET`  | `/raffles/{raffleId}` | — | — |
| `GET`  | `/raffles/{raffleId}/details` | — | — |
| `PATCH`| `/raffles/{raffleId}/configuration` | — | ```json
{ "ticketPrice?": 20.0, "maxTickets?": 50000 }
``` |
| `POST` | `/raffles/{raffleId}/open-sales` | — | ```json
{ "note": "Gate open" }
``` |
| `POST` | `/raffles/{raffleId}/close-sales`| — | ```json
{ "note": "7th inning cutoff" }
``` |
| `GET`  | `/raffles/{raffleId}/status` | — | — |
| `DELETE`| `/raffles/{raffleId}` | — | ```json
{ "reason": "Event cancelled" }
``` |
| `GET`  | `/raffles/{raffleId}/transactions` | `page`,`limit`,`type` | — |

---

## 4  Draw Management

| Method | Path | Query Params | Body |
|--------|------|--------------|------|
| `POST` | `/raffles/{raffleId}/draws` | — | ```json
{ "method": "rng", "rngParams": { "seedSource": "hardware" }, "approvedBy": "uuid" }
``` |
| `GET`  | `/raffles/{raffleId}/draws/latest` | — | — |
| `GET`  | `/raffles/{raffleId}/draws/{drawId}/results` | — | — |
| `POST` | `/raffles/{raffleId}/draws/{drawId}/verify`  | — | ```json
{ "verifierId": "uuid", "hash": "SHA256…" }
``` |
| `GET`  | `/raffles/{raffleId}/winners` | `page`,`limit` | — |

---

## 5  Ticket Management

| Method | Path | Query Params | Body |
|--------|------|--------------|------|
| `POST` | `/raffles/{raffleId}/tickets` | — | ```json
{ "operatorId": "uuid", "rsuId": "uuid", "quantity": 1, "payment": { "method": "cash", "amount": 10.0 } }
``` |
| `POST` | `/raffles/{raffleId}/tickets/batch` | — | ```json
{ "batch": [ { "operatorId": "...", "quantity": 5, "payment": { … } } ] }
``` |
| `GET`  | `/raffles/{raffleId}/tickets` | `page`,`limit`,`status` | — |
| `GET`  | `/tickets/{ticketId}` | — | — |
| `POST` | `/tickets/{ticketId}/void` | — | ```json
{ "operatorId": "uuid", "reason": "spoil" }
``` |
| `POST` | `/tickets/batch-void` | — | ```json
{ "ticketIds": ["id1","id2"], "operatorId": "uuid", "reason": "printer jam" }
``` |
| `GET`  | `/raffles/{raffleId}/tickets/voids` | `page`,`limit` | — |
| `POST` | `/tickets/validate` | — | ```json
{ "validationNumber": "ABC123" }
``` |
| `GET`  | `/tickets/search` | `query`,`raffleId`,`page`,`limit` | — |
| `POST` | `/tickets/{ticketId}/refund` | — | ```json
{ "operatorId": "uuid", "amount": 10.0, "reason": "customer request" }
``` |

---

## 6  Winner Management

| Method | Path | Body |
|--------|------|------|
| `POST` | `/winners/verify` | ```json
{ "ticketId": "uuid", "operatorId": "uuid" }
``` |
| `POST` | `/winners/{winnerId}/claim` | ```json
{ "operatorId": "uuid", "payoutMethod": "cash", "amount": 5000.0 }
``` |
| `GET`  | `/winners/{winnerId}/status` | — |
| `PATCH`| `/winners/{winnerId}/documentation` | ```json
{ "fields": { "photoId": "s3://…", "signature": "base64…" } }
``` |

---

## 7  RSU Management

| Method | Path | Query Params | Body |
|--------|------|--------------|------|
| `GET`  | `/rsus` | `page`,`limit`,`sort`,`status` | — |
| `GET`  | `/rsus/{rsuId}` | — | — |
| `POST` | `/rsus` | — | ```json
{ "serial": "ABC-123", "location": "Gate 4", "model": "X500" }
``` |
| `DELETE`| `/rsus/{rsuId}` | — | ```json
{ "reason": "retired" }
``` |
| `PATCH`| `/rsus/{rsuId}` | — | ```json
{ "location?": "Suite Level", "firmwareVersion?": "1.2.4" }
``` |
| `PUT`  | `/rsus/{rsuId}/enable` | — | ```json
{ "operatorId": "uuid" }
``` |
| `PUT`  | `/rsus/{rsuId}/disable`| — | ```json
{ "operatorId": "uuid", "reason": "maintenance" }
``` |
| `POST` | `/rsus/{rsuId}/heartbeat` | — | ```json
{ "battery": 87, "temperature": 35.2 }
``` |
| `GET`  | `/rsus/{rsuId}/heartbeat-history` | `page`,`limit`,`from`,`to` | — |
| `PUT`  | `/rsus/{rsuId}/configuration` | — | ```json
{ "printer": { "dpi": 300 }, "timezone": "America/New_York" }
``` |

---

## 8  RSU Ticket Operations

| Method | Path | Body |
|--------|------|------|
| `POST` | `/rsus/{rsuId}/tickets/allocate` | ```json
{ "raffleId": "uuid", "range": { "start": 10001, "end": 10100 } }
``` |
| `GET`  | `/rsus/{rsuId}/tickets` | `page`,`limit`,`status` | — |
| `POST` | `/rsus/{rsuId}/reconcile` | ```json
{ "raffleId": "uuid", "sales": 87, "voids": 3, "cashTotal": 870.0 }
``` |
| `GET`  | `/rsus/{rsuId}/reconcile/status` | `raffleId` | — |
| `POST` | `/rsus/{rsuId}/tickets/return` | ```json
{ "raffleId": "uuid", "range": { "start": 10101, "end": 10200 } }
``` |

---

## 9  Reporting

| Method | Path | Query Params / Body |
|--------|------|---------------------|
| `GET` | `/reports/raffle-draw/{raffleId}` | `format` (`pdf`\|`csv`) |
| `GET` | `/reports/exceptions` | `from`,`to`,`rsuId`,`operatorId` |
| `GET` | `/reports/bearer-tickets/{raffleId}` | `format` |
| `GET` | `/reports/sales-by-rsu` | `raffleId`,`format` |
| `GET` | `/reports/voided-tickets` | `raffleId`,`format` |
| `GET` | `/reports/rsu-events` | `rsuId`,`from`,`to` |
| `GET` | `/reports/rsu-integrity` | `rsuId`,`from`,`to` |
| `POST`| `/reports/custom` | ```json
{ "sqlId": "storedQuery42", "params": { "raffleId": "uuid" }, "format": "csv" }
``` |
| `GET` | `/reports/financial-summary/{raffleId}` | `format` |
| `GET` | `/reports/financial-summary` | `from`,`to`,`format` |

---

## 10  Event & Audit

| Method | Path | Query Params |
|--------|------|--------------|
| `GET` | `/events` | `page`,`limit`,`type`,`component`,`from`,`to` |
| `GET` | `/events/search` | `query`,`page`,`limit` |
| `GET` | `/events/{eventId}` | — |
| `GET` | `/audit/changes` | `page`,`limit`,`table`,`userId`,`from`,`to` |
| `GET` | `/audit/access` | `page`,`limit`,`userId`,`from`,`to` |
| `GET` | `/audit/authentication` | `page`,`limit`,`userId`,`success`,`from`,`to` |

---

## 11  RNG Management

| Method | Path | Body / Query |
|--------|------|--------------|
| `GET` | `/rng/status` | — |
| `GET` | `/rng/verification` | — |
| `POST`| `/rng/tests` | ```json
{ "battery": "dieharder", "cycles": 100000 }
``` |
| `GET` | `/rng/entropy` | — |
| `POST`| `/rng/reseed` | ```json
{ "source": "manual", "entropy": "hex-string" }
``` |

---

## 12  Security

| Method | Path | Query.Params | Body |
|--------|------|-------------|------|
| `GET` | `/firewall/status` | — | — |
| `GET` | `/firewall/logs` | `page`,`limit`,`from`,`to`,`action` | — |
| `PUT` | `/firewall/rules` | — | ```json
{ "add": [ { "cidr": "198.51.100.0/24", "action": "allow" } ], "remove": ["203.0.113.0/24"] }
``` |
| `GET` | `/remote-sessions` | `page`,`limit` | — |
| `GET` | `/remote-sessions/history` | `page`,`limit`,`from`,`to`,`userId` | — |

---

## 13  Backup & Recovery

| Method | Path | Body / Query |
|--------|------|--------------|
| `POST` | `/backups` | ```json
{ "type": "full", "note": "pre-upgrade" }
``` |
| `GET`  | `/backups` | `page`,`limit`,`type`,`from`,`to` |
| `POST` | `/backups/{backupId}/restore` | ```json
{ "target": "production" }
``` |
| `GET`  | `/backups/{backupId}/verify` | — |
| `GET`  | `/backups/{backupId}/download` | — |

---

### Next Steps

* Add **response schemas** & error codes via OpenAPI once fields are confirmed.  
* Validate roles/permissions per endpoint.  
* Align with GLI audit-logging requirements in implementation.
