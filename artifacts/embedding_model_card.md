# Model Card: reprezentacja embeddingowa PolEmo 2.0 (multilingual-E5-small)

## Model

| parametr | wartość |
|---|---|
| checkpoint | `intfloat/multilingual-e5-small` |
| wymiar reprezentacji | 384 (`float32`) |
| normalizacja | L2 (`normalize_embeddings=True`), cosinus = iloczyn skalarny |
| okno modelu | `max_seq_length = 512` tokenów |

Wymagane prefiksy wejścia:

| zastosowanie | zapytanie | dokument |
|---|---|---|
| porównanie tekstów, sąsiedzi, PCA (symetryczne) | `query: ...` | `query: ...` |
| wyszukiwanie (asymetryczne) | `query: ...` | `passage: ...` |

## Dane
- Zbiór: PolEmo 2.0-IN (KLEJ, Allegro), recenzje z medycyny i hotelarstwa oraz teksty neutralne (`zero`, głównie fragmenty artykułów).
- Próbka: 1000 dokumentów, wyłącznie ze splitu `train` (bez validation i test).
- Dobór próbki: `train_test_split(..., train_size=1000, stratify=target, random_state=42)`, czyli losowanie deterministyczne z zachowaniem proporcji klas (`minus` 0.379, `plus` 0.271, `amb` 0.182, `zero` 0.168, tak samo jak w `train`).
- Teksty obcięte przez okno modelu: 3.7% (dla porównania MiniLM z oknem 128 tokenów obcina 76.7%).

## Koszt

| miara | wartość |
|---|---:|
| czas utworzenia reprezentacji (1000 dok.) | 1.058 s |
| ms/dokument | 1.058 |
| rozmiar reprezentacji w pamięci | 1.465 MB (gęsta macierz 1000 × 384) |

Pomiar na GPU, bez ładowania modelu, po rozgrzewce na 10 dokumentach. To pojedynczy pomiar, więc wartości są orientacyjne. Ponowne kodowanie dokumentów z prefiksem `passage:` do wyszukiwania nie jest wliczone.

## Jakość sąsiedztwa

| metryka | E5 | punkt odniesienia (losowi sąsiedzi) |
|---|---:|---:|
| `purity@1` | 0.687 | 0.279 |
| `purity@5` | 0.665 | 0.279 |
| `purity@10` | 0.642 | 0.279 |

`purity@5` według klas:

| klasa | `purity@5` |
|---|---:|
| zero | 0.825 |
| minus | 0.758 |
| plus | 0.658 |
| amb | 0.334 |

E5 ma najwyższą purity spośród testowanych reprezentacji (`purity@5`: TF-IDF 0.559, MiniLM 0.580).

## Wyszukiwanie

| metryka | wartość |
|---|---:|
| `Precision@5` | 0.500 |
| `Precision@10` | 0.475 |

Benchmark: 8 zapytań (po 2 na klasę), relewantność przybliżona etykietą sentymentu. MiniLM uzyskał wyższe wyniki (0.625 / 0.562), ale przy 8 zapytaniach różnica ok. 0.1 nie jest rozstrzygająca.

## Analiza jakościowa
- PCA: dwie pierwsze składowe wyjaśniają tylko 17.5% wariancji (12.2% + 5.3%). PC1 rozdziela domeny (hotele i medycyna), PC2 oddziela teksty `zero` (artykuły) od recenzji. Klasy `plus`, `minus` i `amb` całkowicie się mieszają, więc sentyment nie jest widoczny w głównych kierunkach przestrzeni.
- Dominujące typy błędów (12 przypadków, zad. 8):
  - wspólny temat przy przeciwnym sentymencie: pary `plus`–`minus` z danych mają podobieństwo ok. 0.94,
  - antonimy i negacja: „miła” / „niemiła” = 0.98, „Hotel jest dobry.” / „Hotel nie jest dobry.” = 0.92,
  - zamiana ról tych samych słów jest niewidoczna (0.98),
  - podobieństwa są ściśnięte wysoko: para niezwiązana tematycznie ma już 0.83, więc wartości bezwzględne słabo różnicują wyniki.

## Zastosowanie
Rekomendowany do:
- wyszukiwania dokumentów i przyszłego systemu RAG (osobne prefiksy `query:` / `passage:`, długie okno, mało obciętych tekstów),
- grupowania i wyszukiwania sąsiadów według tematu i domeny.

Nie należy go wybierać do:
- klasyfikacji sentymentu na podstawie samego podobieństwa (nie odróżnia negacji ani antonimów, `amb` ma `purity@5` = 0.334),
- zadań z progiem podobieństwa ustalanym na wartościach bezwzględnych cosinusa,
- wyszukiwania po dokładnych nazwach i terminach (tu wystarcza tańszy TF-IDF).

## Ograniczenia
- **Dane:** tylko 1000 dokumentów z `train` i dwie domeny. Klasa `zero` to głównie inny gatunek tekstu (artykuły), co zawyża jej purity. Teksty są wstępnie stokenizowane (np. „byli śmy”).
- **Benchmark:** etykieta sentymentu zastępuje ocenę relewantności, wyszukiwanie oceniono na 8 zapytaniach, a analizę sąsiadów na 3 dokumentach. Δ w analizie przypadków liczona względem jednej pary niezwiązanej. Czasy to pojedyncze pomiary.
- **Reprezentacja:** podobieństwa w wąskim zakresie (ok. 0.83–0.98), wrażliwość na brak prefiksów, słabe rozróżnianie negacji, antonimów i ironii. 3.7% tekstów jest obcinanych do 512 tokenów.
