# AsciiFlow

Interpreter języka dziedzinowego, który zamienia obraz, GIF i wideo na klatki złożone ze znaków ASCII.

Skrypt `.first` opisuje źródło i parametry renderu. Silnik sam rozpoznaje typ pliku, wycina klatki i mapuje jasność pikseli na zestaw znaków. Wynik to plik tekstowy albo katalog `frame_00000.txt` wraz z manifestem JSON, który odtwarza terminalowy player.

## Rola w projekcie

Odpowiadałem za algorytm, który rozpoznaje wchodzący materiał i przetwarza go na klatki wykorzystując znaki ASCII.

Ta część obejmuje:

- rozpoznanie typu wejścia (obraz statyczny, animowany GIF, wideo),
- próbkowanie klatek i odczyt czasu trwania każdej z nich,
- skalowanie kadru z korektą proporcji znaku,
- pomiar jasności piksela i dobór znaku z zestawu posortowanego według pokrycia atramentem,
- zapis sekwencji klatek oraz manifestu do odtwarzania.

## Jak działa przetwarzanie

1. **Rozpoznanie materiału.** Rozszerzenie pliku decyduje o ścieżce: PNG/JPEG idą jako pojedyncza klatka, GIF jest czytany klatka po klatce, a MP4, WebM, AVI, MOV i MKV są rozbijane przez FFmpeg.
2. **Próbkowanie.** `sample every N frames` pomija klatki pośrednie. Dla GIF-a czasy opóźnień z metadanych są sumowane, żeby animacja nie przyspieszała. Dla wideo czas klatki wynika z ustawionego FPS.
3. **Raster do znaków.** Kadr jest skalowany do zadanej szerokości (wysokość uwzględnia to, że znak terminala jest wyższy niż szerszy). Jasność liczona jest wagami postrzeganymi (`0.2126 R + 0.7152 G + 0.0722 B`). Opcjonalny próg zamienia obraz na czerń i biel, a flaga `invert` odwraca mapowanie.
4. **Zestaw znaków.** Znaki z `charset` są rysowane wybraną czcionką monospace i sortowane według rzeczywistego pokrycia atramentem. Ciemniejszy piksel dostaje gęstszy znak, jaśniejszy — rzadszy.
5. **Eksport.** Obraz statyczny trafia do jednego `.txt` i podglądu HTML. Animacja trafia do katalogu klatek i manifestu JSON (`file`, `durationMs`).

## Stos

| Warstwa | Technologia |
| --- | --- |
| Język skryptów | gramatyka [ANTLR 4](https://www.antlr.org/) (`AsciiFlow.g4`) |
| Wykonanie | visitor, tablica symboli, wyrażenia, `if` / `for` |
| Obraz i GIF | Java ImageIO |
| Wideo | FFmpeg (wywoływany jako proces) |
| Odtwarzanie | player terminalowy oparty o manifest JSON |

## Szybki start

Wymagane: JDK z `javac` w `PATH`. Dla wideo dodatkowo [FFmpeg](https://ffmpeg.org/).

```bat
build.bat
java -cp "out;antlr-4.11.1-complete.jar" interpreter.Start we.first
```

Odtworzenie wyeksportowanej animacji:

```bat
java -cp out player.StartPlayer path\to\manifest.json --loop
```

`build.bat` pobiera `antlr-4.11.1-complete.jar`, jeśli go nie ma, i kompiluje parser oraz źródła do katalogu `out/`.

## Przykład skryptu

```text
let targetWidth = 100;

source "clip.mp4";
sample every 2 frames;
fps 12;

set width = targetWidth;
set charset = "@%#*+=-:. ";
set fontName = "Consolas";
set fontSize = 16;
set invert = true;
set ffmpegPath = "ffmpeg";

filter grayscale;

export ascii to "out/frames";
export json to "out/manifest.json";
```

Dla animacji `export ascii` oczekuje katalogu. `export json` zapisuje manifest dopiero po eksporcie klatek. Obraz statyczny można zapisać od razu do pojedynczego pliku `.txt`.

### Instrukcje języka

| Instrukcja | Znaczenie |
| --- | --- |
| `source "plik"` | ścieżka do obrazu, GIF-a lub wideo |
| `sample every N frames` | co która klatka wchodzi do wyniku |
| `fps N` | FPS ekstrakcji wideo i domyślny czas klatki |
| `set width / charset / fontName / fontSize` | rozmiar i wygląd rastra |
| `set invert / threshold / ffmpegPath` | odwrócenie, próg i ścieżka do FFmpeg |
| `filter grayscale \| invert \| threshold(n)` | filtry jasności |
| `export ascii to "..."` | klatki tekstowe albo jeden plik |
| `export json to "..."` | manifest odtwarzacza lub opis renderu |
| `let`, `if`, `for` | zmienne i sterowanie |

## Struktura

```text
src/grammar/AsciiFlow.g4          gramatyka języka
src/interpreter/                  visitor, plan renderu, algorytm klatek
src/player/                       odtwarzacz manifestu w terminalu
src/SymbolTable/                  zakresy zmiennych
*.first                           przykładowe skrypty
build.bat                         kompilacja
```

Katalog `ascii-movie-1.9.7/` to osobne, zewnętrzne narzędzie [ascii-movie](https://github.com/gabe565/ascii-movie) (odtwarzacz gotowych filmów ASCII przez terminal, SSH i Telnet). Nie jest częścią interpretera AsciiFlow.

## Nazwa repozytorium

Rekomendowana nazwa: **`ascii-flow`**.

Jest krótka, zgodna z nazwą gramatyki i od razu mówi, że to język oraz potok przetwarzania, a nie pojedynczy skrypt „video to ascii”.
