# Automatyczne reklamy wideo (n8n + OpenAI + Pollo.ai)

Workflow n8n, który z krótkiego opisu produktu robi pionową reklamę wideo 9:16 pod TikTok, Instagram Reels i YouTube Shorts.

Etap 1 (ten workflow) obejmuje kroki 1 do 3 planu:

1. **Wejście:** formularz n8n albo webhook `POST` z danymi produktu.
2. **Scenariusz:** OpenAI (GPT-4o) pisze hook, tekst lektora, napisy na ekran, CTA i angielski prompt do wideo.
3. **Generowanie:** Pollo.ai tworzy wideo 9:16, a workflow co 20 s sprawdza status, aż wideo będzie gotowe.

Wersje pod poszczególne platformy, akceptacja i publikacja to kolejne etapy.

## Przebieg workflow

```
Formularz reklamy ─┐
                   ├─> Normalizuj dane i ustawienia ─> Zbuduj zapytanie do AI ─> OpenAI: scenariusz i prompt
Webhook API ───────┘
  ─> Parsuj scenariusz ─> Pollo.ai: utwórz wideo ─> Zapamiętaj taskId
  ─> Czekaj 20 s ─> Pollo.ai: status zadania ─> Sprawdź status ─> Wideo gotowe? ─ tak ─> Wynik
                ^                                                              │
                └──────────────────────────── nie ─────────────────────────────┘
```

Węzeł **Wynik** zwraca link do wideo (`videoUrl`), okładkę (`coverUrl`), zużyte kredyty i cały scenariusz. Pollo.ai przechowuje pliki tylko do 14 dni, więc pobierz wideo albo zapisz je u siebie.

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
| `Pollo.ai API` (typ **Header Auth**) | `x-api-key` | klucz Pollo.ai | panel Pollo.ai, sekcja API Keys ([instrukcja](https://docs.pollo.ai/quick-start)) | Pollo.ai: utwórz wideo, Pollo.ai: status zadania |

Po imporcie n8n może pokazać ostrzeżenie przy tych węzłach. Otwórz każdy z nich i wybierz właściwe poświadczenie z listy.

## Jak uruchomić

### Formularz

Otwórz węzeł **Formularz reklamy** i skopiuj **Production URL** (albo **Test URL**, gdy testujesz w edytorze). Pola:

- **Produkt** (wymagane)
- **Opis produktu** (wymagane)
- **Styl reklamy**, np. dynamiczny, zabawny, premium, UGC
- **Wezwanie do działania (CTA)**
- **URL zdjęcia produktu** (opcjonalnie, musi być `https://`, JPG lub PNG). Gdy je podasz, Pollo.ai użyje zdjęcia jako pierwszej klatki.

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
| `polloModel` | `pollo/pollo-v1-6` | Model Pollo.ai. Alternatywa: `google/veo3-1-fast` (droższy, generuje też dźwięk). |
| `resolution` | `720p` | Pollo 1.6: `480p`, `720p`, `1080p`. Veo 3.1 Fast: `720p`, `1080p`, `4k`. |
| `length` | `5` | Długość w sekundach. Pollo 1.6: `5` lub `10`. Veo 3.1 Fast: `4`, `6` lub `8`. |
| `aspectRatio` | `9:16` | Pionowe wideo. |
| `llmModel` | `gpt-4o` | Model OpenAI, który pisze scenariusz. |
| `language` | `polski` | Język tekstów reklamy. Prompt do wideo jest zawsze po angielsku. |

Pollo 1.6 w trybie zdjęcie-na-wideo bierze proporcje ze zdjęcia, więc do pionowej reklamy podawaj pionowe zdjęcie. Veo 3.1 Fast wymusza 9:16 w obu trybach.

Węzeł **Sprawdź status** czeka maksymalnie 60 sprawdzeń po 20 s (20 minut), a potem kończy wykonanie błędem. Błąd pojawia się też, gdy Pollo.ai zwróci status `failed` albo OpenAI odmówi odpowiedzi.

Uwaga na limit czasu wykonania w n8n. Jeśli Twoja instancja ma ustawione `EXECUTIONS_TIMEOUT` (np. 120 s), workflow zostanie przerwany, zanim wideo się wygeneruje. Ustaw limit na co najmniej 1800 s (zmienne `EXECUTIONS_TIMEOUT` i `EXECUTIONS_TIMEOUT_MAX` w kontenerze, potem restart n8n) albo podnieś go w ustawieniach workflow (Settings > Timeout Workflow).

Testowy adres formularza (`form-test/...`) działa tylko chwilę po kliknięciu **Execute workflow** i tylko na jedno wysłanie.

## Dokumentacja API

- Pollo.ai: [Pollo 1.6](https://docs.pollo.ai/m/pollo/pollo-v1-6), [Veo 3.1 Fast](https://docs.pollo.ai/m/google/veo3-1-fast), [status zadania](https://docs.pollo.ai/task/get-task-status), [webhooki](https://docs.pollo.ai/webhooks)
- OpenAI: [Structured Outputs](https://platform.openai.com/docs/guides/structured-outputs)
