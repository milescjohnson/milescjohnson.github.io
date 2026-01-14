
# Bias & Sentiment Analysis API

## Base URL

```
https://api.knowhere.ai/v1
```

All responses are JSON. All requests must be made over HTTPS.

---

## Authentication

The API uses API keys via the `Authorization` header.

```
Authorization: Bearer YOUR_API_KEY
```

---

## Core Concepts

### Sentiment

Represents emotional tone toward a subject.

* `positive`
* `neutral`
* `negative`

Returned as both a **label** and a **confidence score**.

### Bias

Bias is represented along multiple axes:

* **Political**: left ↔ center ↔ right
* **Establishment**: establishment ↔ anti-establishment
* **Emotional framing**: factual ↔ inflammatory

Bias scores are normalized to `[-1.0, 1.0]` where `0` is neutral.

### Framing & Language Signals

Identifies rhetorical techniques such as:

* Loaded language
* Omission
* Fear-based framing
* Moral appeals

---

## Endpoints

---

## 1. Analyze a Single Article

Analyze a full article or long-form text.

### `POST /analyze/article`

#### Request Body

```json
{
  "url": "https://examplenews.com/politics/climate-policy",
  "options": {
    "extract_paywalled": false
    "include_sentence_analysis": true,
    "include_key_phrases": true
  }
}
```

#### Response

```json
{
  "article_id": "art_92kaf8",
  "story_id": "story_32fez9",
  "title": "Government Announces New Climate Policy",
  "text": "The administration unveiled sweeping reforms today...",
  "source": "Example News",
  "published_at": "2026-01-10T14:30:00Z",
  "sentiment": {
    "label": "negative",
    "score": -0.42,
    "confidence": 0.87
  },
  "bias": {
    "political": -0.35,
    "establishment": 0.12,
    "emotional_framing": 0.58
  },
  "framing_signals": [
    "loaded_language",
    "fear_appeal"
  ],
  "key_phrases": [
    "sweeping reforms",
    "economic uncertainty",
    "critics warn"
  ],
  "summary": "Article frames the policy as risky and economically disruptive.",
  "model_version": "bias-sentiment-v3.2"
}
```

---

## 2. Analyze a URL

Fetch and analyze a publicly available news article.

### `POST /analyze/url`

#### Request Body

```json
{
  "url": "https://examplenews.com/politics/climate-policy",
  "options": {
    "extract_paywalled": false
  }
}
```

#### Response

```json
{
  "article_id": "art_77fd12",
  "source": "Example News",
  "headline": "Government Announces New Climate Policy",
  "sentiment": {
    "label": "neutral",
    "score": 0.05
  },
  "bias": {
    "political": -0.28,
    "emotional_framing": 0.21
  }
}
```

---

## 3. Sentence-Level Bias Breakdown

Useful for highlighting bias directly in the UI.

### `POST /analyze/sentences`

#### Request Body

```json
{
  "text": "The proposal was met with outrage. Supporters argue it is necessary."
}
```

#### Response

```json
{
  "sentences": [
    {
      "text": "The proposal was met with outrage.",
      "sentiment": "negative",
      "bias": {
        "emotional_framing": 0.74
      }
    },
    {
      "text": "Supporters argue it is necessary.",
      "sentiment": "positive",
      "bias": {
        "emotional_framing": 0.12
      }
    }
  ]
}
```

---

## 4. Compare Coverage Across Sources

Compare how different outlets frame the same topic.

### `POST /compare`

#### Request Body

```json
{
  "query": "climate policy",
  "sources": [
    "Outlet A",
    "Outlet B",
    "Outlet C"
  ]
}
```

#### Response

```json
{
  "topic": "climate policy",
  "comparisons": [
    {
      "source": "Outlet A",
      "avg_sentiment": -0.31,
      "political_bias": -0.45
    },
    {
      "source": "Outlet B",
      "avg_sentiment": 0.02,
      "political_bias": 0.01
    }
  ]
}
```

---

## 5. Model Explainability

Explain *why* the model detected bias or sentiment.

### `POST /explain`

#### Request Body

```json
{
  "article_id": "art_92kaf8"
}
```

#### Response

```json
{
  "highlights": [
    {
      "text": "critics warn the policy could devastate jobs",
      "reason": "loaded language",
      "impact": "negative sentiment"
    }
  ],
  "confidence_notes": "Bias confidence is moderate due to mixed framing."
}
```

---

## Error Handling

### Error Response Format

```json
{
  "error": {
    "code": "INVALID_INPUT",
    "message": "Text field is required"
  }
}
```

### Common Error Codes

| Code               | Description                |
| ------------------ | -------------------------- |
| `UNAUTHORIZED`     | Invalid or missing API key |
| `INVALID_INPUT`    | Malformed request          |
| `CONTENT_TOO_LONG` | Input exceeds token limit  |
| `RATE_LIMITED`     | Too many requests          |

---

## Rate Limits

* **Free tier**: 100 requests/day
* **Pro tier**: 10,000 requests/day
* Burst limit: 5 requests/second

---

## Versioning

The API is versioned via the URL (`/v1`).
Breaking changes will be released under `/v2`.

---

## Notes

* Bias scores are probabilistic, not factual judgments.
* Results should be presented with uncertainty and context.
* Intended for **analysis and comparison**, not absolute truth.
