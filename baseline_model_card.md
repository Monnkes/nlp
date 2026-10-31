# Model Card: baseline sentymentu PolEmo 2.0 (TF-IDF + Logistic Regression)

## Model
Pipeline `scikit-learn`:

| krok | konfiguracja |
|---|---|
| `TfidfVectorizer` | `analyzer="word"`, `ngram_range=(1, 2)`, `min_df=2`, `max_features=50_000`, `sublinear_tf=True` |
| `LogisticRegression` | `C=2.0`, `max_iter=1000`, `random_state=42` |

- Wejście: kolumna `text_norm` (NFKC + lowercase + redukcja białych znaków).
- Plik: `artifacts/FINAL_word_lr.joblib`.

## Task
Klasyfikacja sentymentu całych recenzji do 4 klas: `plus`, `minus`, `zero`, `amb` (pozytywny, negatywny, neutralny, niejednoznaczny). Model zwraca klasę oraz prawdopodobieństwa (`predict_proba`).

## Training data
PolEmo 2.0-IN (KLEJ, Allegro): recenzje konsumenckie z medycyny i hotelarstwa. Szczegóły, czyszczenie i audyt wycieków: `dataset_card.md`. Klasy są niezbalansowane: `minus` ~38–42%, `plus` ~27–29%, `zero` i `amb` po ~15–18%.

## Splits and metric

| split | n | użycie |
|---|---|---|
| train | 5783 | uczenie |
| validation | 723 | wybór hiperparametrów |
| test | 722 | jedna ocena końcowa |

- Główna metryka: `macro-F1` (słaba jakość na małej klasie nie chowa się za dużą). Metryki pomocnicze: accuracy, macro-precision, macro-recall.
- Przedziały ufności: bootstrap 95% (1000 próbek).
- Hiperparametry (`C`) wybrane tylko na validation. Test: po wyborze konfiguracji trening na `train + validation` (6506), jedna ocena na teście.

## Results

| zbiór | accuracy | macro-F1 | macro-F1 CI95 |
|---|---:|---:|---|
| validation (trening: train) | 0.830 | 0.800 | [0.771, 0.827] |
| test (trening: train + validation) | 0.837 | 0.802 | [0.772, 0.833] |

Test: macro-precision 0.830, macro-recall 0.789.

F1 na validation według klas:

| klasa | F1 |
|---|---:|
| plus | 0.850 |
| minus | 0.867 |
| zero | 0.976 |
| amb | 0.505 |

## **Cost**

| miara | wartość |
|---|---:|
| czas treningu (`train`) | 4.1 s |
| czas treningu (`train + validation`) | 4.6 s |
| inference | 0.152 ms/dokument (`predict` na całej validation, średnia z 3 powtórzeń) |
| rozmiar pliku `.joblib` | 3.47 MB |

## Failure analysis
- 125 błędów na 723 przykłady validation.
- Największa pomyłka: `amb` → `minus` (49 przykładów); `plus` jest mylone z `amb` (20) i z `minus` (19).
- Najtrudniejsza klasa to `amb` (F1 ok. 0.5): recenzje mieszane zawierają słownictwo obu biegunów, a model liniowy sumuje wagi słów.
- Ręczna taksonomia 15 błędów (10 losowych + 5 o najwyższej pewności): `MIXED_SENTIMENT` 6, `AMBIGUOUS_GOLD` 5, `LONG_CONTEXT`, `DOMAIN_TERM`, `NEGATION`, `SPELLING_NOISE` po 1. Wszystkie 5 błędów o najwyższej pewności to klasa `amb`; przy części z nich sama etykieta wydaje się dyskusyjna.
- Cechy o dużych wagach zawierają artefakty (`em`, `proc`, `pap`): pozostałości pretokenizacji i skróty z artykułów prasowych. Waga cechy nie oznacza związku przyczynowego.
- Odporność: po usunięciu polskich znaków macro-F1 spada o 0.094 (z 0.800 do 0.706), bo model zna słowa tylko w formie z diakrytykami. Literówki (ok. 2% znaków) prawie nie zmieniają wyniku (-0.007).
- Długość tekstu nie wpływa wyraźnie na jakość (różnice między koszykami wynikają ze składu klas).

## Confidence threshold
Confidence treshold ma sens. Pewność modelu odróżnia trafienia od pomyłek (średnie maksimum `predict_proba`: 0.664 dla poprawnych, 0.483 dla błędnych).

| próg | coverage | accuracy | macro-F1 |
|---:|---:|---:|---:|
| 0.40 | 0.923 | 0.855 | 0.810 |
| 0.50 | 0.722 | 0.929 | 0.852 |
| 0.60 | 0.556 | 0.950 | 0.806 |
| 0.70 | 0.385 | 0.971 | 0.740 |
| 0.80 | 0.199 | 1.000 | 0.750 |

Rozsądny kompromis to próg ok. 0.5: model odpowiada na ok. 72% przykładów z accuracy 0.93, a resztę kieruje do człowieka. Przy progach od 0.7 macro-F1 spada, bo część klas jest słabo reprezentowana wśród zaakceptowanych przykładów.

## Intended use
- Wstępne sortowanie recenzji (medycyna, hotelarstwo, język polski) z możliwością odrzucenia mało pewnych przypadków do ręcznej oceny.
- Narzędzie diagnostyczne: model liniowy pozwala obejrzeć wagi cech.

## Not intended for
- Zastosowań komercyjnych: dane mają licencję CC BY-NC-SA 4.0, więc model trenowany na nich traktujemy jako użytek niekomercyjny.
- Wiarygodnego rozpoznawania klasy `amb` (F1 ok. 0.5) i decyzji, w których pomyłka jest kosztowna.
- Innych domen (np. media społecznościowe, produkty, polityka), innych okresów czasowych i tekstów bez podobnej pretokenizacji.
- Tekstów bez polskich znaków diakrytycznych: jakość wyraźnie spada.

## Limitations
- Wszystkie wyniki dotyczą tylko oficjalnych splitów PolEmo 2.0-IN.
- Zbiór zawiera teksty pretokenizowane, ucięte lub uszkodzone oraz pary near-duplicate (w tym jedną między train a test); mogą one lekko zawyżać wyniki.
- Etykieta `amb` bywa dyskusyjna, więc jej część to szum etykiet, a nie błąd modelu.
- Czas treningu i inference to pomiary na jednym komputerze, bez uwzględnienia obciążenia. Wartości bezwzględne są orientacyjne.
