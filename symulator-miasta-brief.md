# Symulator miasta (nazwa robocza) — dokument startowy

> Brief projektowy podsumowujący ustalenia. Punkt wyjścia do dalszej pracy w Claude Code.
> Data: sierpień 2026

---

## 1. Przegląd

Mobilny city-builder **inspirowany** klasycznym SimCity z DOS-a (Maxis, 1989) — **nie remake**. Chodzi o oddanie ducha oryginału (strefowanie, wzrost miasta z prostych reguł, „miasto żyje samo"), ale we własnym silniku, z własnym kodem i własną grafiką.

- **Platforma docelowa:** telefony (Android / iOS).
- **Silnik:** Unity (6.3 LTS / 6.4 — Entities już w pudełku, gdyby kiedyś było potrzebne).
- **Tryb pracy:** solo, iteracyjnie (model sprawdzony przy flutter-rogue: zamknięty zakres → wydanie → devlogi).

---

## 2. Kluczowe decyzje

1. **Własny silnik symulacji od zera** — nie wykorzystujemy gotowego kodu Micropolisa.
2. **„Wzorować się", nie kopiować.** Mechaniki (idee) są wolne. Ale kod Micropolisa jest na **GPL**, więc uczymy się z *opisów*, a nie z plików `.c` — inaczej ryzykujemy dzieło pochodne i zarażenie licencją GPL przy zamkniętej/komercyjnej grze. Szczególnie unikamy przenoszenia oryginalnych tablic „magicznych liczb".
3. **Model symulacji: polowy (agregatowy) na v1** — mapa jako stos warstw skalarnych, bez pojedynczych agentów. To powód, dla którego SimCity dało się zrobić w pojedynkę.
4. **Agenci odłożeni.** Symulacja agentowa (widoczni mieszkańcy/pojazdy) była kusząca, ale to inna, dużo droższa liga (pathfinding na skalę, emergentne korki, budżet cieplny telefonu). Nie zamykamy jednak drzwi:
   - *wariant A (może później):* agenci **kosmetyczni** — warstwa wizualna podążająca za polem gęstości, bez wpływu na ekonomię;
   - *wariant B (poza zakresem):* agenci **napędzający ekonomię** — świadomie odrzucony na tym etapie.
5. **Sim oddzielony od renderowania.** Symulacja żyje na czystych danych; renderowanie tylko *czyta* te dane i maluje Tilemapę. (Ten sam wzorzec co w flutter-rogue.)
6. **Wydajność — inkrementalnie.** Start na zwykłych zarządzanych tablicach C#. Gorące przebiegi (dyfuzje, flood-fill) przeniesiemy do **Jobs + Burst** dopiero, gdy zajdzie potrzeba. **Pełne DOTS/ECS odłożone** — miałoby sens głównie przy agentach.

---

## 3. Zakres

### v1 (MVP) — „miasto ożywa"
Cel: minimalna pętla, w której stawiasz strefy i drogi, a miasto samo rośnie.

- Siatka kafelków + edycja terenu / stawianie kafelków.
- Strefowanie **R / C / I** (mieszkalne / komercyjne / przemysłowe).
- **Drogi** i połączenia.
- **Prąd** — flood-fill z elektrowni.
- **Popyt R/C/I** sterujący wzrostem stref.
- Prosty **budżet + podatki**.

### Później (kolejne warstwy)
- Zanieczyszczenie, land value, przestępczość (pola skalarne).
- Woda.
- Katastrofy (pożar / powódź — rozprzestrzenianie kafelek po kafelku).
- Ewentualnie kosmetyczni agenci (wariant A).

---

## 4. Architektura rdzenia symulacji (polowego)

### 4.1. Mapa jako stos warstw
Każda warstwa to płaska tablica o rozmiarze siatki (np. `byte[width*height]` albo `int[]`), indeksowana `y*width + x`. Warstwy oddziałują na siebie między tickami.

Warstwy startowe:
- **teren / typ kafelka** (woda, ląd, droga, strefa, budynek…),
- **strefa** (R/C/I + poziom zabudowy),
- **prąd** (zasilony / niezasilony),
- **gęstość ruchu** (pole),
- **land value** (pole skalarne),
- **zanieczyszczenie** (pole skalarne),
- **przestępczość** (pole skalarne),
- **populacja / poziom rozwoju kafelka**.

### 4.2. Pętla symulacji (tick)
Wzorzec z oryginału: jeden „zegar" dzielony na fazy — nie liczymy wszystkiego co klatkę ani wszystkiego naraz. Różne fazy odświeżają różne warstwy (rozłożenie kosztu w czasie). Odpowiednik 16-fazowego `Simulate()` z SimCity.

Szkic kolejności jednego pełnego obrotu:
1. **Prąd** — flood-fill z elektrowni po grafie połączeń; oznacz kafelki zasilone.
2. **Ruch** — próbkowane szacowanie natężenia między strefami; zapis do pola gęstości.
3. **Pola skalarne** — jeden/kilka przebiegów wygładzających (automat komórkowy): zanieczyszczenie, land value, przestępczość dyfundują do sąsiadów.
4. **Wzrost/upadek stref** — dla każdej strefy: warunki lokalne (prąd + dojazd + land value + zanieczyszczenie) → rośnie lub podupada.
5. **Popyt R/C/I** — przelicz globalne zapotrzebowanie na podstawie stanu miasta.
6. **Budżet** — raz na „miesiąc/rok": podatki, koszty utrzymania.

> Ważne: tick symulacji **odseparowany od klatek renderowania** — sim tyka np. co 0,25 s (albo szybciej przy przewijaniu czasu), render maluje niezależnie.

### 4.3. Wzorce implementacyjne
- **Bez `MonoBehaviour` na kafelek.** Tysiące GameObjectów = śmierć wydajności na telefonie. Kafelek to komórka w tablicy.
- Sim = model danych; Unity **Tilemap** = widok, który tylko czyta dane.
- Graf dróg trzymany osobno (węzły/krawędzie) — nawet w modelu polowym przydaje się do propagacji prądu i szacowania ruchu.

---

## 5. Renderowanie i UX mobilny

- **Tilemap** (Grid + Tilemap) czytający ze stanu symulacji; ewentualnie własny `CustomRenderer` przy niestandardowych efektach.
- Oryginalny, gęsty pasek narzędzi **nie przechodzi na dotyk**. Potrzebne:
  - duże cele dotykowe, sensowne menu (np. wysuwane / radialne),
  - płynny **zoom (pinch)** i **pan**,
  - przemyślany tryb **portretowy** z chowanym UI,
  - czytelne podświetlanie stref/warstw (podgląd land value, zanieczyszczenia itd.).

---

## 6. Aspekty prawne / nazewnictwo

- **Nie** używamy nazwy „SimCity" (znak towarowy EA) ani oryginalnych assetów (grafika, dźwięki, muzyka).
- Mechaniki i idee gry — wolne do odtworzenia.
- Micropolis traktujemy **wyłącznie jako materiał koncepcyjny**; kod na GPL trzymamy na dystans (patrz decyzja #2).

---

## 7. Materiały źródłowe (do nauki mechanik, nie kodu)

- **Chaim Gingold — „SimCity Reverse Diagrams"** — wizualne rozłożenie reguł symulacji (16-fazowy zegar, warstwy mapy i ich interakcje). Idealne „jak to myśli" bez dotykania kodu.
- **Chaim Gingold — „Building SimCity: How to Put the World in a Machine" (MIT Press)** — rozdział „How SimCity Works" + aneks z diagramami.
- **Micropolis / MicropolisCore / MicropolisJ** — tylko jako punkt odniesienia koncepcyjnego (pamiętać o GPL).

---

## 8. Pierwszy krok w Claude Code

Proponowana kolejność na start projektu Unity:

1. Ustaw projekt 2D + `Grid`/`Tilemap`; zdefiniuj rozmiar siatki (np. 120×100) i strukturę warstw (płaskie tablice).
2. Zrób **edycję**: malowanie terenu, dróg i stref R/C/I na siatce (input dotykowy + kamera z zoom/pan).
3. Zaimplementuj **pętlę ticku** oddzieloną od renderu (na razie pusty szkielet faz).
4. Dodaj **flood-fill prądu** jako pierwszą realną fazę — widoczny efekt: strefy przy elektrowni są zasilone.
5. Dodaj **prosty popyt R/C/I** + wzrost stref → zobacz, jak miasto zaczyna rosnąć.
6. Dopiero potem: pola skalarne (zanieczyszczenie/land value), budżet, kolejne warstwy.

> Zasada przewodnia: najpierw grywalna, skończona pętla (v1.0), potem warstwy. Wydajność (Jobs+Burst) i ewentualni agenci — jako świadome, późniejsze decyzje, nie na wejściu.
