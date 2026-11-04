# Model Card: baseline sentymentu PolEmo 2.0 (char TF-IDF + Logistic Regression)

## Model
Pipeline `scikit-learn`:

| krok | konfiguracja |
|---|---|
| `TfidfVectorizer` | `analyzer="char_wb"`, `ngram_range=(3, 5)`, `min_df=2`, `max_features=50_000`, `sublinear_tf=True` |
| `LogisticRegression` | `C=7.0`, `max_iter=1000`, `random_state=42` |

- Wejście: kolumna `text_norm` (NFKC + lowercase + redukcja białych znaków).
- Plik: `artifacts/FINAL_char_lr.joblib` (model trenowany na `train`, używany w Arenie). Model oceniany na teście był trenowany od nowa na `train + validation`.

## Task
Klasyfikacja sentymentu całych recenzji do 4 klas: `plus`, `minus`, `zero`, `amb` (pozytywny, negatywny, neutralny, niejednoznaczny). Model zwraca klasę oraz prawdopodobieństwa (`predict_proba`).

## Training data
PolEmo 2.0-IN (KLEJ, Allegro): recenzje konsumenckie z medycyny i hotelarstwa.

## Splits and metric

| split | n | użycie |
|---|---|---|
| train | 5783 | uczenie |
| validation | 723 | wybór reprezentacji i hiperparametrów |
| test | 722 | ocena końcowa |

- Główna metryka: `macro-F1` (słaba jakość na małej klasie nie chowa się za dużą). Metryki pomocnicze: accuracy, macro-precision, macro-recall.
- Przedziały ufności: bootstrap 95% (1000 próbek).
- Reprezentacja (word vs char) i `C` wybrane tylko na validation. Test: po wyborze konfiguracji trening na `train + validation` (6506), ocena na teście.

## Why this model
- Najwyższe macro-F1 na validation spośród testowanych wariantów bez `class_weight` (0.805 vs 0.800 dla word + LR). Sama różnica mieści się w przedziale ufności.
- Wyraźnie większa odporność na brak polskich znaków (spadek macro-F1 o 0.033 vs 0.094 dla word).
- Lepszy wynik na krótkich tekstach (0.793 vs 0.768).

## Results

| zbiór | accuracy | macro-F1 | macro-F1 CI95 |
|---|---:|---:|---|
| validation (trening: train) | 0.830 | 0.805 | [0.777, 0.834] |
| test (trening: train + validation) | 0.839 | 0.820 | [0.791, 0.849] |

Test: macro-precision 0.826, macro-recall 0.814.

F1 na validation według klas:

| klasa | F1 |
|---|---:|
| plus | 0.826 |
| minus | 0.874 |
| zero | 0.992 |
| amb | 0.528 |

## Cost

| miara | wartość |
|---|---:|
| czas treningu (`train`) | 7.0 s |
| czas treningu (`train + validation`) | 6.7 s |
| inference | 0.50 ms/dokument (`predict` na całej validation, średnia z 3 powtórzeń) |
| rozmiar pliku `.joblib` | 3.18 MB |

Czasy to pojedyncze pomiary na jednym komputerze, więc wartości są orientacyjne.

## Failure analysis
- Najczęstsze pomyłki to klasa `amb` mylona z `minus` i `plus`. Klasa `zero` jest rozpoznawana prawie bezbłędnie.
- Najtrudniejsza klasa to `amb` (F1 ok. 0.53): recenzje mieszane zawierają słownictwo z mieszanym sentymentem, a model liniowy sumuje wagi cech.
- Ręczna taksonomia 15 błędów (10 losowych + 5 o najwyższej pewności), wykonana na wariancie word + LR: `MIXED_SENTIMENT` 6, `AMBIGUOUS_GOLD` 5, `LONG_CONTEXT`, `DOMAIN_TERM`, `NEGATION`, `SPELLING_NOISE` po 1. 
- Analiza wag cech (wariant word + LR) pokazała, że obok sensownych cech (`nie polecam`, `polecam`, `ale`) model korzysta też z artefaktów: pozostałości pretokenizacji (`em`) i skrótów z artykułów prasowych w klasie `zero` (`proc`, `pap`). To skrót, a nie rozumienie sentymentu.
- Odporność: po usunięciu polskich znaków macro-F1 spada o 0.033 (z 0.805 do 0.772). Literówki (ok. 2% znaków) nie pogarszają wyniku (+0.003).

## Confidence threshold
Confidence threshold ma sens. Pewność modelu odróżnia trafienia od pomyłek (wariant word + LR, validation: średnie maksimum `predict_proba` 0.664 dla poprawnych, 0.483 dla błędnych).

| próg | coverage | accuracy | macro-F1 |
|---:|---:|---:|---:|
| 0.40 | 0.923 | 0.855 | 0.810 |
| 0.50 | 0.722 | 0.929 | 0.852 |
| 0.60 | 0.556 | 0.950 | 0.806 |
| 0.70 | 0.385 | 0.971 | 0.740 |

Sensowny kompromis to próg ok. 0.5: model odpowiada na ok. 72% przykładów z accuracy 0.93, a resztę kieruje do człowieka. Próg trzeba dobrać osobno dla finalnego modelu char, bo inne modele mają inny rozkład pewności.

## Intended use
- Wstępne sortowanie recenzji (medycyna, hotelarstwo, język polski) z możliwością odrzucenia mało pewnych przypadków do ręcznej oceny.
- Teksty o nieformalnej pisowni (bez polskich znaków, z literówkami): model jest na nie wyraźnie odporniejszy niż wariant słowny.

## Not intended for
- Wiarygodnego rozpoznawania klasy `amb` (F1 ok. 0.53) i decyzji, w których pomyłka jest kosztowna.
- Innych domen niż medycyna i hotelarstwo (np. media społecznościowe, produkty, polityka).
- Zastosowań wymagających bardzo szybkiej inferencji z bardzo dużą ilością danych: zastosowany tutaj wariant vectorizera `char` jest zauważalnie wolniejszy niż `word`.
- Wyjaśniania decyzji użytkownikom końcowym: cechy to fragmenty słów, trudniejsze do interpretacji niż słowa.

