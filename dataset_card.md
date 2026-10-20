# Dataset Card: PolEmo 2.0

## Dataset
PolEmo 2.0-IN z benchmarku KLEJ (Allegro). Recenzje konsumenckie z sektorów medycyny i hotelarstwa.

- Źródło: https://huggingface.co/datasets/allegro/klej-polemo2-in (pliki `train.csv`, `valid.csv`, `test.csv`)
- Publikacja: KLEJ, https://arxiv.org/abs/2005.00630

## Task
Klasyfikacja sentymentu całych tekstów do 4 klas. Na podstawie treści recenzji przewidujemy jej ogólny wydźwięk: pozytywny (plus), negatywny (minus), neutralny (zero) lub niejednoznaczny (amb). Etykieta dotyczy całego tekstu, a nie pojedynczych zdań.

## Schema

| kolumna | typ | opis |
|---|---|---|
| `text` | str | oryginalny tekst (`sentence` po zmianie nazwy), bez modyfikacji |
| `target` | str | etykieta po mapowaniu `LABEL_MAP` |
| `split` | str | `train` / `validation` / `test` |
| `text_norm` | str | NFKC + lowercase + redukcja białych znaków + trim; tylko do wykrywania duplikatów |
| `dup_within_split` | bool | duplikat `text_norm` w obrębie splitu |
| `dup_cross_split` | bool | ten sam `text_norm` w więcej niż jednym splicie |
| `near_dup_cross_split` | bool | near-duplicate (cosine char 3–5-gram ≥ 0.9) z rekordem z innego splitu |

## Labels
| oryginalna etykieta | etykieta | znaczenie |
|---|---|---|
| `__label__meta_plus_m` | `plus` | pozytywny |
| `__label__meta_minus_m` | `minus` | negatywny |
| `__label__meta_zero` | `zero` | neutralny |
| `__label__meta_amb` | `amb` | niejednoznaczny |

Oryginalne etykiety z pliku źródłowego zostały zamienione na krótsze.


## Splits

| split | przed czyszczeniem | po czyszczeniu |
|---|---|---|
| train | 5783 | 5783 |
| validation | 723 | 723 |
| test | 722 | 722 |

Zachowano oficjalny podział autorów zbioru; czyszczenie nie usunęło żadnego rekordu, więc liczebności splitów się nie zmieniły.


## Data quality
- Brak wartości brakujących w `text` i `target`; liczba rekordów zgodna ze źródłem.
- Bardzo krótkie / ucięte teksty: 1 rekord krótszy niż 20 znaków (`"P"`, train, `amb`).
- Długość: mediana 696 znaków (121 słów), p99 ~2335, max 5391. Wszystkie teksty są jednoliniowe.
- Szum: 9 tekstów z URL, 0 z adresem e-mail, 808 z serią `!` / `?`, 10 z przewagą wielkich liter (próg 0.5).
- Klasy są niezbalansowane: `minus` ~38–42%, `plus` ~27–29%, `zero` i `amb` po ~15–18%.
- Rozkład klas jest podobny między splitami.

## Cleaning

Zmieniono:

| operacja | split | efekt |
|---|---|---|
| usunięcie rekordów bez tekstu lub labelu | wszystkie | usunięto 0 |
| usunięcie exact duplicates wewnątrz train | train | usunięto 0 |
| usunięcie z train rekordów, których `text_norm` występuje w validation / test | train | usunięto 0 |
| flaga `dup_within_split` | wszystkie | oznaczono 0 |
| flaga `dup_cross_split` | wszystkie | oznaczono 0 |
| flaga `near_dup_cross_split` | wszystkie | oznaczono 2 |

Świadomie nie zmieniono:

- treści tekstów (bez lowercase, usuwania interpunkcji, sklejania),
- validation i test (zachowane oficjalne rozmiary, problemy tylko oflagowane),
- „dziwnych” tekstów (bardzo krótkich, uciętych, itd.),
- near-duplicate'ów: oznaczone, ale nie usunięte.

## Leakage audit
- Exact duplicates (`text_norm`): 0 w obrębie splitów, 0 między splitami.
- Near-duplicates (TF-IDF char_wb 3–5, cosine ≥ 0.9): 3 pary, 2 w obrębie jednego splitu, 1 między splitami. 

## License / provenance
Dane pochodzą ze zbioru PolEmo 2.0 (Politechnika Wrocławska / CLARIN-PL), udostępnionego w benchmarku KLEJ przez Allegro na Hugging Face. Metadane zbioru na Hugging Face podają licencję CC BY-SA 4.0, a treść karty zbioru podaje CC BY-NC-SA 4.0. Ponieważ licencje się różnią, przyjmujemy bardziej restrykcyjną CC BY-NC-SA 4.0: dane wolno wykorzystywać wyłącznie niekomercyjnie, z uznaniem autorstwa, a utwory zależne trzeba udostępniać na tej samej licencji. W karcie zbioru na Hugging Face jest też błąd w tabeli rozkładu klas: `minus` opisano jako *positive*, a `plus` jako *negative*. Udziały procentowe klas zgadzają się z naszymi danymi, więc przyjmujemy `plus` = pozytywny, `minus` = negatywny.

## Limitations
- Teksty są wstępnie stokenizowane (interpunkcja oddzielona spacjami, rozbite formy typu `"byli śmy"`), co widać w różnicy `n_words` i `n_words_split`. Wyniki uzyskane na tym zbiorze nie muszą przenosić się na tekst bez takiej obróbki.
- W danych są teksty ucięte lub uszkodzone, których etykieta nie wynika z treści. Pozostawiono je w zbiorze.
- Brak duplikatów dokładnych nie oznacza braku wycieku: znaleziono parę near-duplicate train ↔ test (ucięta kopia tego samego tekstu), która pozostała w danych.
- Uzyskane rezultaty dotyczą ściśle określonego zadania oraz oficjalnych splitów testowych. Nie należy ich bezposrednio ekstrapolować na nowe domeny, inny horyzont czasowy czy odmienne źródła danych.
