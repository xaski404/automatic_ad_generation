# Automatyczne reklamy wideo (n8n + OpenAI + Higgsfield)

Workflow n8n, który z krótkiego opisu produktu robi pionową reklamę wideo 9:16 pod TikTok, Instagram Reels i YouTube Shorts.

Etap 1 (ten workflow) obejmuje kroki 1 do 3 planu:

1. **Wejście:** formularz n8n albo webhook `POST` z danymi produktu.
2. **Scenariusz:** OpenAI (GPT-4o) pisze hook, tekst lektora, napisy na ekran, CTA i angielski prompt do wideo.
3. **Generowanie:** Higgsfield (model Kling 2.6) tworzy wideo 9:16, a workflow co 20 s sprawdza status, aż wideo będzie gotowe.

Wersje pod poszczególne platformy, akceptacja i publikacja to kolejne etapy.

## Przebieg workflow

```
Formularz reklamy ─┐
                   ├─> Normalizuj dane i ustawienia ─> Zbuduj zapytanie do AI ─> OpenAI: scenariusz i prompt
Webhook API ───────┘
  ─> Parsuj scenariusz ─> Higgsfield: utwórz wideo ─> Zapamiętaj ID zadania
  ─> Czekaj 20 s ─> Higgsfield: status zadania ─> Sprawdź status ─> Wideo gotowe? ─ tak ─> Wynik
                ^                                                              │
                └──────────────────────────── nie ─────────────────────────────┘
```

Węzeł **Wynik** zwraca link do wideo (`videoUrl`) i cały scenariusz. Pobierz wideo albo zapisz je u siebie, bo link od dostawcy nie musi działać zawsze.

## Import do n8n

1. W n8n kliknij **Create workflow**, potem menu `...` i **Import from File**.
2. Wybierz plik [`workflows/video-ad-generator.json`](workflows/video-ad-generator.json).
3. Dodaj dwa poświadczenia (poniżej) i wybierz je w węzłach HTTP Request.
4. Zapisz workflow i włącz go przełącznikiem **Active**, żeby działał formularz produkcyjny i webhook.

## Poświadczenia (credentials)

Klucze API trzymasz wyłącznie w n8n. W repozytorium nie ma żadnych kluczy, a plik JSON odwołuje się tylko do nazw poświadczeń.

Potrzebne są dwa poświadczenia (Credentials > Add credential):

| Nazwa poświadczenia | Name (nagłówek) | Value | Gdzie wziąć klucz | Węzły |
|---|---|---|---|---|
| `OpenAI account` (typ **OpenAI**) | nie dotyczy | klucz OpenAI w polu API Key | [platform.openai.com/api-keys](https://platform.openai.com/api-keys) | OpenAI: scenariusz i prompt |
| `Higgsfield API` (typ **Header Auth**) | `Authorization` | `Key ` + ID klucza + `:` + sekret, np. `Key abc123:xyz789` | panel Higgsfield API, sekcja **API keys** ([instrukcja](https://docs.higgsfield.ai/docs/authentication)) | Higgsfield: utwórz wideo, Higgsfield: status zadania |

Po imporcie n8n może pokazać ostrzeżenie przy tych węzłach. Otwórz każdy z nich i wybierz właściwe poświadczenie z listy.

## Jak uruchomić

### Formularz

Otwórz węzeł **Formularz reklamy** i skopiuj **Production URL** (albo **Test URL**, gdy testujesz w edytorze). Pola:

- **Produkt** (wymagane)
- **Opis produktu** (wymagane)
- **Styl reklamy**, np. dynamiczny, zabawny, premium, UGC
- **Wezwanie do działania (CTA)**
- **URL zdjęcia produktu** (opcjonalnie, musi być `https://`, JPG lub PNG). Gdy je podasz, Higgsfield użyje zdjęcia jako pierwszej klatki.

Formularz od razu potwierdza przyjęcie, a wynik zobaczysz w zakładce **Executions**.

### Webhook

```bash
curl -X POST "https://TWOJ-N8N/webhook/video-ad" \
  -H "Content-Type: application/json" \
  -d @examples/request.json
```

Przykładowe dane są w [`examples/request.json`](examples/request.json). Pola: `product`, `description`, `style`, `cta`, `imageUrl`.

## Ustawienia

Na górze węzła **Normalizuj dane i ustawienia** jest obiekt `SETTINGS`:

| Pole | Domyślnie | Opis |
|---|---|---|
| `videoModel` | `kling-video/v2.6/pro` | Model w API Higgsfield. Workflow dokleja `/text-to-video` albo `/image-to-video` (gdy podasz zdjęcie). |
| `length` | `5` | Długość w sekundach: `5` lub `10`. |
| `aspectRatio` | `9:16` | Pionowe wideo (`16:9`, `9:16` albo `1:1`). |
| `sound` | `off` | `on` dogeneruje dźwięk do wideo, ale kosztuje więcej. |
| `llmModel` | `gpt-4o` | Model OpenAI, który pisze scenariusz. |
| `language` | `polski` | Język tekstów reklamy. Prompt do wideo jest zawsze po angielsku. |

Węzeł **Sprawdź status** czeka maksymalnie 60 sprawdzeń po 20 s (20 minut), a potem kończy wykonanie błędem. Błąd pojawia się też, gdy Higgsfield zwróci status `failed`, `nsfw` albo `canceled` albo OpenAI odmówi odpowiedzi.

Uwaga na limit czasu wykonania w n8n. Jeśli Twoja instancja ma ustawione `EXECUTIONS_TIMEOUT` (np. 120 s), workflow zostanie przerwany, zanim wideo się wygeneruje. Ustaw limit na co najmniej 1800 s (zmienne `EXECUTIONS_TIMEOUT` i `EXECUTIONS_TIMEOUT_MAX` w kontenerze, potem restart n8n) albo podnieś go w ustawieniach workflow (Settings > Timeout Workflow).

Testowy adres formularza (`form-test/...`) działa tylko chwilę po kliknięciu **Execute workflow** i tylko na jedno wysłanie.

## Dokumentacja API

- Higgsfield: [Kling 2.6 tekst na wideo](https://docs.higgsfield.ai/docs/models/kling-2-6/pro-text-to-video), [Kling 2.6 zdjęcie na wideo](https://docs.higgsfield.ai/docs/models/kling-2-6/pro-image-to-video), [status zadania](https://docs.higgsfield.ai/docs/api-reference/requests/get-request-status), [uwierzytelnianie](https://docs.higgsfield.ai/docs/authentication)
- OpenAI: [Structured Outputs](https://platform.openai.com/docs/guides/structured-outputs)
