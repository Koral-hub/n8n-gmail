# n8n – ćwiczenie przed rozmową (STX Next, AI Automation Engineer)

## Cel
Zbudować w n8n (klikając w UI, nie programowo) jeden działający workflow obsługi maili logistycznych,
żeby na rozmowie mówić z doświadczenia: konkretne nazwy węzłów i ustawień.
Rekruter zwrócił uwagę na brak konkretów: Retry On Fail, pinned data, queue mode, MCP Client Tool.

Rola Claude: pomoc przy setupie (Docker, credentials), debugowanie, wyjaśnianie ustawień węzłów,
przygotowanie odpowiedzi na rozmowę. Workflow buduję sam w UI, nie generuj gotowego JSON-a workflow,
chyba że poproszę.

## Środowisko (gotowe pliki w tym folderze)
- `docker-compose.yml`: n8n (main) + n8n-worker + Postgres 16 + Redis 7, **queue mode** włączony
  (`EXECUTIONS_MODE=queue`, `OFFLOAD_MANUAL_EXECUTIONS_TO_WORKERS=true`).
- `.env` (z `.env.example`): POSTGRES_USER / PASSWORD / DB, `N8N_ENCRYPTION_KEY`
  (wygenerowany raz przez `openssl rand -hex 32`, nie zmieniać po pierwszym starcie;
  przy zmianie: `docker compose down -v`).
- Klucze do LLM-ów (OpenAI / Anthropic) i Gmail **nie** idą do `.env`, tylko do n8n → Credentials.
- Credential Postgres w n8n: host `postgres` (nazwa usługi w Dockerze), nie `localhost`.
- Gmail OAuth na self-hosted: OAuth Client w Google Cloud,
  redirect `http://localhost:5678/rest/oauth2-credential/callback`, własny mail jako test user.
- UI: http://localhost:5678

## Docelowa architektura

### Workflow A: Mock TMS
Webhook (GET `tms/shipment`, respond via Respond to Webhook) → Edit Fields
(`status` zależny od `{{ $json.query.nr }}`, np. kończy się na 7 → "opóźniona") → Respond to Webhook.

### Workflow B: MCP Server – TMS
MCP Server Trigger + HTTP Request Tool do mocka TMS (dobry opis narzędzia, bo agent wybiera po opisie).
Aktywny; Production URL używany w MCP Client Tool.

### Workflow C: główny
1. **Gmail Trigger**, Message Received.
2. **AI Agent**, sub-node'y:
   - Chat Model (OpenAI / Anthropic),
   - **Postgres Chat Memory**, Session Key = `{{ $('Gmail Trigger').item.json.threadId }}` (pamięć per wątek),
   - **MCP Client Tool** → endpoint z workflow B,
   - Require Specific Output Format + **Structured Output Parser**:
     `{"kategoria": "...", "pewnosc": 0.0, "numer_przesylki": "...", "szkic_odpowiedzi": "..."}`
   - Settings: **Retry On Fail** (3 próby, 2000 ms).
3. **IF** `pewnosc < 0.7` → do człowieka (label / powiadomienie).
4. **Switch** po `kategoria`: status_przesylki / reklamacja / phishing_spam / fallback.
5. Gałąź status: **Gmail → Send and Wait for Response** (Approval, szkic w treści) → po zatwierdzeniu **Gmail → Reply**.
   Odrzucone → label „Do ręcznej obsługi”. Phishing → Add Label, bez odpowiedzi.

### Workflow D: Error Workflow
Error Trigger → Gmail/Slack z `workflow.name`, `execution.url`, `execution.error.message`.
Podpięty w Workflow C: Settings → Error Workflow. Działa tylko dla aktywnych (produkcyjnych) wykonań.
Test: aktywować C i zepsuć URL mocka.

## Testowanie
- **Pinned data**: przypiąć wyjście Gmail Trigger i edytować JSON (phishing, brak numeru przesyłki,
  reklamacja, niejednoznaczny mail). Pinned data działa tylko w ręcznych wykonaniach.
- Queue mode: sprawdzić w Executions, że wykonuje worker; `docker compose up -d --scale n8n-worker=2`.
- Na koniec: Download workflow jako JSON → repo w Git.

## Status
- [x] docker-compose.yml + .env.example
- [ ] Kontenery działają, konto założone
- [ ] Credentials: LLM, Postgres, Gmail
- [ ] Workflow A (mock TMS)
- [ ] Workflow B (MCP Server)
- [ ] Workflow C (główny)
- [ ] Workflow D (Error Workflow) + test
- [ ] Testy na pinned data, skalowanie workerów, eksport do Git

## Punkty na rozmowę (do przećwiczenia po zbudowaniu)
- Pamięć per `threadId`: dlaczego przy mailach w ogóle potrzebna jest pamięć.
- Próg `pewnosc`: samoocena LLM-a jest słabo skalibrowana, więc to heurystyka, a niska pewność trafia do człowieka.
- Obsługa błędów: Retry On Fail + error output + osobny Error Workflow z alertem.
- Queue mode: main (UI, triggery, webhooki) → Redis → workery; wymaga Postgresa i wspólnego `N8N_ENCRYPTION_KEY`.
- Kiedy Python zamiast n8n: testy i CI/CD, złożona logika agentowa ze stanem (LangGraph),
  systematyczna ewaluacja, duża skala. Często hybryda: n8n orkiestruje, a trudną część robi usługa w Pythonie przez HTTP.
