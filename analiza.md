# Analiza: `vf2 gemini.md` i `vf2-recomp-android.md`

Data: 2026-10-18. **Tylko analiza** — żadnego pliku projektu nie zmieniono.
Metoda: każde twierdzenie z obu notatek sprawdzone w kodzie tego repozytorium (odnośniki `plik:linia`)
i w pomiarach z piątego przebiegu na telefonie. Nie zgaduję — co dało się sprawdzić, jest sprawdzone;
co się nie dało, jest nazwane „niezweryfikowane".

Legenda:

- ✅ **trafne i już zrobione** — nie ma czego robić
- 🟡 **sensowne, ale bez skutku** — no-op na naszej ścieżce (albo szkodliwe tylko wtedy, gdy ktoś naprawdę to wpisze)
- 🔴 **błędne dla nas** — wprowadzenie tego zepsułoby działające rzeczy
- 🔵 **sensowne i niezrobione** — jedyna kategoria z realną propozycją
- ⚪ **niesprawdzalne tutaj** — twierdzenie o systemie/sterowniku, do sprawdzenia na telefonie

## Orzeczenie w jednym zdaniu

`vf2-recomp-android.md` to uczciwa notatka inżynierska (autor sam pisze, że kodu nie czytał), której **większość
punktów jest już w projekcie zrobiona**, a jej najlepsza myśl (Android Performance Hint API) to jedyna rzecz
z obu plików warta rozważenia. `vf2 gemini.md` to surowa odpowiedź LLM-a wklejona jako dokument — zepsuty
Markdown, artefakt czatu `<FollowUp>`, a jej konkretne recepty (manifest, hinty SDL, `minImageCount = 3`)
są **albo już u nas są, albo są no-opem, albo są wprost błędne** — z jednym wyjątkiem: ostrzeżenie o FMA/`mediump`,
które jest całkowicie słuszne i które koryguje moją własną wcześniejszą sugestię „spróbować fp16".

---

## 1. `vf2 gemini.md` — twierdzenie po twierdzeniu

### 1.1 Plik jako dokument

- ⚪ **Sam plik jest uszkodzony.** Wszystko od `## 2` w dół to jedna linia: brakuje przejść do nowej linii,
  bloki kodu są wklejone w zdania, w atrybucie XML siedzi link Markdown
  (`xmlns:android="[http://schemas.android.com/apk/res/android](http://...)"` — tak się nie skompiluje),
  a na końcu jest artefakt czatu: `<FollowUp> Czy potrzebujesz, żebym dodał… </FollowUp>`.
  To nie jest dokumentacja, tylko niesformatowana odpowiedź modelu. Sensu merytorycznego to nie przekreśla,
  ale **nie wolno z tego kopiować fragmentów do kodu** — trzeba przepisać ręcznie.

### 1.2 Architektura Mali / TBDR

| Twierdzenie | Ocena | Uzasadnienie w naszym kodzie |
|---|---|---|
| „Tile Reloads: bez jawnego czyszczenia bufora klatki GPU przepisuje dane z RAM do SRAM" | 🟡 | Kierunek prawdziwy (Arm zaleca `LOAD_OP_CLEAR`/`DONT_CARE`), ale **my nie tworzymy żadnego render passu** — 3D na GPU to wyłącznie pipeline'y compute (`src/gpu/gpu3d_sdlgpu.cpp`, brak `SDL_GPURasterizerState`, brak samplera), a 2D i prezentacja idą przez SDL_GPU, gdzie load/store ops wybiera SDL. Nie mamy gdzie tego ustawić. |
| „V-Sync / Fence Stalls: przy podwójnym buforowaniu sterownik MediaTek agresywnie usypia GPU" | 🟡 | Obszar trafny, ale to już **zmierzone i zamknięte**: SDL wymuszał 1 klatkę w locie (`SDL_render_gpu.c:1809`), zrobiliśmy z tego ustawienie `frames_in_flight` 1–3 i piąty przebieg pokazał 1 ≈ 2 ≈ 3. Płotki przy `vsync 0` to ~9,5 ms (wcześniej 14,24). Sformułowanie „podwójne buforowanie" jest tu luźne. |
| „PowerHAL: aplikacje spoza listy gier w HyperOS są traktowane jak tło i GPU spada z 1300 MHz do ~300–400 MHz" | ⚪🔵 | **Najciekawsze twierdzenie w tym pliku i jedyne, którego nie umiemy dziś potwierdzić ani zaprzeczyć** — bo nie czytamy zegarów GPU. Czytamy tylko CPU (`the cores at:`, `src/host/sysinfo.cpp:266`) i tam throttlingu nie widać (prime 1,4–2,1 z 3,40 GHz, X4 0,55–2,1 z 2,85). Uwaga: gdyby GPU naprawdę chodziło na 300–400 MHz, 294 ms na zbudowanie pipeline'ów i 9,3–10,5 ms na obrazek 1x byłyby podejrzanie dobre. Ale to **weryfikowalne na telefonie** (patrz §3.2). |

### 1.3 „Kluczowe optymalizacje API Vulkan"

| Recepta | Ocena | Dlaczego |
|---|---|---|
| `loadOp = CLEAR`/`DONT_CARE`, nie `LOAD` | 🟡 | Jak wyżej — nie piszemy render passów. |
| Subpassy z `VK_DEPENDENCY_BY_REGION_BIT` dla post-processingu | 🟡 | SDL_GPU **nie używa subpassów**, a nasz projekt nie mamy żadnego łańcucha post-processingu (3D = compute, 2D = SDL_Renderer). Poza zasięgiem bez porzucenia SDL_GPU. |
| `minImageCount = 3` na sztywno | 🟡→🔴 | Liczbę obrazów swapchaina **ustala SDL z „klatek w locie"** (`SDL_gpu_vulkan.c`), więc dziś sterujemy tym przez `frames_in_flight` — i pomiar powiedział, że 2 vs 3 nic nie zmienia. „Na sztywno 3" byłoby więc (a) nie do wyrażenia przez SDL_GPU i (b) sprzeczne z tym, co zmierzyliśmy. |
| `presentMode = MAILBOX` zamiast `FIFO` | ✅ | **Dokładnie to robimy** — `vsync 0` prosi o mailbox, potem immediate, dopiero potem vsync (kolejność SDL: `SDL_render_gpu.c:1610` `ChoosePresentMode`, jej `case 0`). Od dzisiaj w logu widać, co telefon naprawdę wybrał. |

### 1.4 „Konfiguracja SDL2 / SDL3" — tu jest najwięcej nieporozumień

| Recepta | Ocena | Dlaczego |
|---|---|---|
| `SDL_SetHint(SDL_HINT_RENDER_DRIVER, "vulkan")` | 🔴 | Dwa błędy naraz. (1) Hint jest czytany **tylko wtedy, gdy nazwy renderera nie podano** (`SDL_render.c:1107-1109`), a nasz kod podaje nazwę jawnie: `SDL_CreateRenderer(win, "gpu")` (`src/host/main.cpp:1166`) — czyli hint i tak nic nie zrobi. (2) Gdyby go posłuchać i przełączyć na „auto", `"vulkan"` to **inny, stary renderer** (`SDL_render_vulkan.c:4648: VULKAN_CreateRenderer, "vulkan"`), a nie backend Vulkan w SDL_GPU. Backend Vulkan w rendererze `gpu` wybiera się przez `SDL_HINT_GPU_DRIVER` / `SDL_PROP_GPU_DEVICE_CREATE_NAME_STRING` (`SDL_render_gpu.c:1734`) — i nasz kod już to robi w ścieżce awaryjnej (`main.cpp:1164, 1214`). Na Androidzie i tak domyślnie mamy Vulkan: log `SDL_GPU: vulkan, Mali-G720-Immortalis MC12`. |
| `SDL_SetHint(SDL_HINT_RENDER_VSYNC, "0")` | 🟡 | Hint istnieje (`SDL_hints.h:3386`) i ustawia **domyślną** wartość vsync dla nowo tworzonych rendererów (`SDL_render.c:1090-1092`). My ustawiamy vsync świadomie, przez `SDL_SetRenderVSync` w `apply_display_settings()` (`main.cpp:1247-1254`), bo to ustawienie gracza. Hint tylko by z tym walczył — a „wyłączenie vsync na sztywno" jest sprzeczne z tym, o co prosił gracz. |
| `SDL_SetHint(SDL_HINT_VIDEO_DOUBLE_BUFFER, "1")` | 🟡 | **No-op na Androidzie.** Ten hint czyta wyłącznie czwórka plików `kmsdrm`/`raspberry`/`wayland` w `third_party/sdl3/src/video/`, nic w ścieżce GPU/Vulkan. |
| `uses-feature android.hardware.vulkan.version … required="true"` + `android:version="0x00403000"` | 🟡→🔴 | **Mamy już to w manifeście, ale z `required="false"`** (`android/app/src/main/AndroidManifest.xml:22-23`) — i to jest poprawne, bo projekt ma świadomą ścieżkę awaryjną na CPU (tryb Auto potrafi zejść na CPU; renderer `opengl`/`software` też istnieje). `required="true"` zablokowałoby instalację na sprzęcie bez Vulkan 1.3 i **zaprzeczyłoby własnemu projektowi**. Dopisanie `android:version="0x00403000"` byłoby tylko informacyjne (filtr w sklepie), bo przy `required="false"` niczego nie wymusza. `0x00403000` = Vulkan 1.3 — poprawnie zakodowane. |
| `android:appCategory="game"` | ✅ | Już jest (`AndroidManifest.xml:41`). |
| `android:hasFragileUserData="false"` | 🟡 | Atrybut istnieje, ale nic nie daje tam, gdzie `allowBackup="false"` już jest (a jest). |
| `abiFilters 'arm64-v8a'` („Dimensity 9300+ nie ma rdzeni 32-bit") | ✅ | Prawda i **już zrobione** (`android/app/build.gradle:37`). |

### 1.5 „Optymalizacja w HyperOS / MIUI"

- ⚪ **Game Turbo / tryb wysokiej wydajności / „Sterownik graficzny systemu"** — sensowne, ale to działania gracza,
  nie kod. Nie umiem tego sprawdzić zdalnie; jeśli zadziała, należy do `android/README.md`, nie do kodu.
- 🔴 **`adb shell setprop persist.vendor.powerhal.perf_mode 1`** — nie zadziała na nie-zrootowanym telefonie
  (właściwości `persist.vendor.*` są chronione przez SELinux; `adb shell` to nie root). Do odrzucenia jako „droga".

---

## 2. `vf2-recomp-android.md` — twierdzenie po twierdzeniu

Plik zaczyna się od uczciwego zastrzeżenia („kodu źródłowego nie czytałem"). Większość jego punktów to rzeczy,
które w tym repo **już są**, co samo w sobie jest dobrą wiadomością: niezależna analiza doszła do tego samego planu.

### 2.1 Rozdział 1–2: „Czy port jest realny?"

| Twierdzenie | Ocena | Uzasadnienie |
|---|---|---|
| SLJIT ma backend ARM64, sprawdzony na Apple silicon | ✅ | Prawda; u nas działa natywnie (`i960 native` w logu, rekompilacja 880 funkcji w 75 ms). |
| Host używa SDL3, które wspiera Androida | ✅ | Prawda, cały port stoina SDL3. |
| Renderer GPU ma ścieżkę Vulkan przez SDL_GPU, fallback na CPU istnieje | ✅ | Prawda i to jest nasz główny temat. |
| Port na Switch (Thorhax, VF2-NX-Modern) robi 57,5 FPS | ⚪ | Nie do sprawdzenia tutaj i **nie jest dowodem dla Androida** (inny kod, inne GPU). Traktować jako ciekawostkę, nie argument. |

### 2.2 Rozdział 3: „Co trzeba zrobić"

| Punkt | Ocena | Stan u nas |
|---|---|---|
| Build: NDK + CMake + `arm64-v8a` + szkielet Gradle z SDL3 `android-project` | ✅ | Zrobione (Moduły 1–3). |
| JIT potrzebuje `PROT_EXEC` — sprawdzić wcześnie na sprzęcie | ✅ | Sprawdzone; działa i jest w logu (`i960 native`). |
| ROM przez SAF i do prywatnego katalogu aplikacji | ✅ | Zrobione (`romset.h`, picker; `main.cpp:744-748`). |
| Nakładki dotykowe + menu RmlUi pod dotyk | ✅ | Zrobione (Moduły 8–10; `src/host/touch_controls.h`, poprawka `asset_join()`). |
| Wydajność: rasteryzer na compute pod Mali bywa różny, test przy starcie przełącza na CPU, zacznij od 1x/CPU | ✅/🟡 | **Dokładnie tak działa Auto** (`main.cpp:1699` `g3_verify`, 1755, 1791). Uwaga: test nie jest „cichy" — log mówi wprost, na czym 3D poszło i dlaczego. |
| „Przydaje się przypinanie wątków do dużych rdzeni" | 🔵 | **Nie zrobione.** `sysinfo::tune()` (`sysinfo.cpp:215`) jest **tylko dla `_WIN32`** (throttling + hook na hybrydowe rdzenie); na Androidzie to puste wywołanie i żadnego `sched_setaffinity` w kodzie nie ma. Patrz §3.3. |
| Kolejność: najpierw `vf2run` headless przez `adb shell`, potem SDL, na końcu GPU | 🟡 | Rada inżynierska sensowna, ale **u nas już nieaktualna** — poszliśmy od razu na APK i to zadziałało, bo ścieżka APK była od początku oskryptowana. Warto pamiętać jako technikę diagnostyczną (mamy narzędzia `gpu3dbench`/`gpu3dcmp`), nie jako zaległość. |

### 2.3 Rozdział 4.1: bit-dokładność rasteryzera — **najważniejszy i w pełni słuszny punkt obu plików**

| Twierdzenie | Ocena | Uzasadnienie |
|---|---|---|
| „W SPIR-V oznaczaj wyniki `NoContraction` (w HLSL `precise`)" | ✅ | **Zrobione i udokumentowane w repo.** `src/gpu/gpu3d.hlsl` używa `precise` 44 razy, a `tools/gpu/shaders.sh` tłumaczy nawet dlaczego nie ma tam flagi `-Gis` dla SPIR-V: „IEEE strictness, for DXIL: SPIR-V's would need Vulkan 1.2; `precise` does it there". |
| „Unikaj `RelaxedPrecision` (mediump), bo Mali liczy wtedy w fp16" | ✅ **i korekta mojej notatki** | To prawda i to **obowiązuje**. Ścieżka SDL_GPU jest **zawsze „exact"**: `gpu3d_sdlgpu.cpp:120-121` — `set_quality()` jest puste („the exact way only"), więc `gpu_quality`/`VF2_GPU3D_FAST` jej nie dotyczy. Moja wcześniejsza sugestia z `docs/ARM-IMMORTALIS-G720.md` („try fp16 in the GPU 3D compute shader") była **błędna dla trybu exact** i należy ją wykreślić: fp16 łamie bit-dokładność, a sprawdzenie przy starcie odrzuci taki obraz. |
| „Konwersja float→int poza zakresem na ARM daje inny wynik niż x86" | ✅/⚪ | Kierunek prawdziwy, ale u nas złagodzony: weryfikacja akceptuje różnice **zaokrągleń** i tak też wypada w praktyce — piąty przebieg: „1 z 3304 x 1536 pikseli różni się o zaokrąglenie, kolory najwyżej o 5; zachowujemy GPU". |
| „Jeśli test się nie zgodzi, gra **po cichu** spadnie na CPU, bez komunikatu" | 🟡 | Połowicznie. W interfejsie gracz rzeczywiście nie dostaje komunikatu, ale **log mówi wszystko**: linia `the 3D (...) … : on the CPU/GPU` (`main.cpp:1791`), „keeping the GPU" (`1755`), powód w `render state (…)`. To nie „cicho", to „w logu". Jeśli kiedyś chcemy komunikat na ekranie, to drobiazg — ale nie usterka. |
| Zalecenie końcowe: „Zacznij od trybu CPU, GPU dopiero po rozwiązaniu problemu z FMA" | 🟡 | Dla **tego** projektu nadgorliwe: problem FMA jest rozwiązany (`precise` + test + fallback), a Auto sam wybiera. W praktyce CPU i tak wygrywa od 4x w górę, więc rada „zbliżona do prawdy", ale nie z powodu FMA. |

### 2.4 Rozdział 4.2: compute na Mali

| Twierdzenie | Ocena | Uzasadnienie |
|---|---|---|
| Grupa robocza wielokrotnością 16, np. 64 | ✅ | **Już tak jest**: `gpu3d.hlsl` — `[numthreads(64,1,1)]` (linie 363, 373) i `[numthreads(8,8,1)]` (430, 461, 519), czyli 64 wątki = 4 warpy Mali. |
| Pamięć współdzielona leży w cache — nie licz na zysk | ✅/🟡 | Nasze shadery **nie używają** groupshared, więc punkt bezprzedmiotowy, ale merytorycznie słuszny. |
| „TBDR nic tu nie daje, bo rasteryzer działa na compute" | ✅ | Zgadza się i pokrywa się z moim własnym ustaleniem z notatki o Arm: na CSF Mali compute i binning **nie mogą się nakładać**, a nasz rasteryzer to compute — stąd ~19 ms na obrazek i to, że CPU od 4x jest tańszy. To najlepsze zdanie w całym pliku. |

### 2.5 Rozdział 4.3: integracja z Androidem

| Recepta | Ocena | Dlaczego |
|---|---|---|
| „Zapisuj `VkPipelineCache` na dysk" | 🟡 | **Nie da się przez SDL_GPU** — `SDL_gpu.h` nie ma żadnego API pipeline cache (sprawdzone: brak wystąpień). Pipeline'y tworzymy raz, na starcie: „its pipelines ready in 294 ms" — to jednorazowy koszt startu, a sterowniki Mali mają własny dyskowy cache shaderów. |
| „Użyj Swappy (Android Frame Pacing)" | 🟡 | Swappy opakowuje **własny swapchain Vulkan** — nie da się go wpiąć pod SDL_GPU bez porzucenia SDL_GPU. Do tego byłoby to sprzeczne z projektem: emulator celowo idzie własnym rytmem 57,52 Hz (tempo automatów), a nie tempem ekranu. Cel (stabilne klatkowanie) jest realny, ale realizujemy go przez vsync/present mode/governor, nie przez Swappy. |
| „Dodaj Android Performance Hint API" | 🔵 | **Trafione i niezrobione.** To dokładnie API `android/performance_hint.h` z NDK (ADPF), „intended for periodic workloads, such as frame production" — czyli nasza klatka 17,4 ms. SDL tego **nie używa** (sprawdzone: brak w `third_party/sdl3/src/`), a `sysinfo::tune()` na Androidzie nic nie robi. To jest właściwa odpowiedź na „telefon dławi taktowanie" z obu notatek. Szczegóły — §3.1. |
| „Przypnij wątki emulacji do X4" | 🔵 | Niezrobione (patrz §2.2). Wątpliwość: Android i tak zarządza cpusetem i może to nadpisać. Eksperyment, nie oczywisty zysk — §3.3. |
| „Wymuś poziomą orientację i sprawdź pre-rotation" | ✅ | Orientacja: `android:screenOrientation="sensorLandscape"` + `configChanges` (manifest), okno 2712x1220 w logu. Co do pre-rotation: **SDL celowo nie robi pre-rotation na Androidzie** — ustawia `preTransform = VK_SURFACE_TRANSFORM_IDENTITY_BIT_KHR` wyłącznie dla Androida (`SDL_gpu_vulkan.c:4782-4786`), a gdzie indziej `currentTransform`. Obraca więc kompozytor/DPU (na Androidzie zwykle sprzętowo i tanio). Nasz bottleneck jest gdzie indziej, więc „naprawianie" pre-rotation nie ma tu za co się złapać. |

### 2.6 Rozdział 4.4: narzędzia

- ✅ **Arm Performance Studio (Streamline)** — zgodne z tym, co sam wpisałem do `docs/ARM-IMMORTALIS-G720.md`; to jest właściwe narzędzie do rozstrzygnięcia §1.2 (zegary i throttling GPU).
- ✅ **Mali Offline Compiler** — przydatne; nasze shadery to SPIR-V, a `malioc` przyjmuje SPIR-V, więc da się sprawdzić presję na rejestry i użycie fp16/fp32.
- 🟡 **Warstwy walidacyjne w APK** — mamy już `GPU3D_VALIDATE=1`; dorzucanie `libVkLayer_khronos_validation.so` do APK jest możliwe, ale to kilkadziesiąt MB i zerowa korzyść dla graczy. Ewentualnie osobny APK deweloperski — poza zakresem.
- ✅ **Vulkan Hardware Capability Viewer** — sensowne, żeby zobaczyć, jakie formaty `storage image` i funkcje ma sterownik Mali; to bezpośrednio dotyczy naszych tekstur w `gpu3d_sdlgpu.cpp`.

---

## 3. Co z tego naprawdę warto zrobić (i w jakiej kolejności)

### 3.1 🔵 Android Performance Hint API (ADPF) — jedyna rzecz nowa i uzasadniona

- Co: przy starcie utworzyć sesję `APerformanceHint_createSession` na wątkach, które produkują klatkę,
  ustawić `targetWorkDuration` = 17,4 ms (albo 1/57,52 s) i co klatkę raportować rzeczywisty czas
  (`APerformanceHint_reportActualWorkDuration`). System zaczyna wtedy **traktować nas jak grę**: podnosi
  i trzyma taktowania CPU/GPU, zamiast traktować proces jak tło.
- Dostępność: nagłówek jest w NDK r30, ale `APerformanceHint_getManager`/`createSession` są oznaczone
  `__INTRODUCED_IN(__ANDROID_API_T__)` = **API 33**, a `minSdk` mamy 26 → potrzebny warunek uruchomieniowy
  (`android_get_device_api_level() >= 33` albo słabe dowiązanie/`dlopen`), a bez niego ciche pominięcie.
- Koszt/ryzyko: małe i odwracalne (kilkadziesiąt linii w hoście, żadnych zmian w SDL ani w rdzeniu).
- Uwaga: to **nie podnosi sufitu** ścieżki GPU (to nadal §o przełączaniu compute/render), ale może odzyskać
  zegary tam, gdzie telefon dławi. Dowód: dopiero pomiar „przed/po" na telefonie.

### 3.2 ⚪ Sprawdzić zegary GPU — jedyny sposób, by rozstrzygnąć twierdzenie o PowerHAL

Dziś log pokazuje tylko zegary CPU (`the cores at:`). Do rozstrzygnięcia „1300 MHz vs 300–400 MHz"
potrzebny jest odczyt devfreq GPU. Na telefonie (przez `adb shell`, bez zmian w kodzie):

```
adb shell ls /sys/class/devfreq/
adb shell 'for d in /sys/class/devfreq/*; do echo "$d $(cat $d/cur_freq 2>/dev/null) / $(cat $d/max_freq 2>/dev/null)"; done'
```

Jeśli ścieżka Mali (`*.mali`, np. `13000000.mali`) pokaże ~300–400 MHz w trakcie gry, twierdzenie z `vf2 gemini.md`
jest prawdziwe i **ADPF (§3.1) jest właściwą odpowiedzią**. Jeśli pokaże zegary wysokie — twierdzenie upada
i nie ma czego gonić. (Aplikacja może nie mieć prawa czytać devfreq przez SELinux; `adb shell` zwykle ma.)
Dopiero po tym warto rozważać dodanie odczytu zegarów GPU do bloku `pace:` w logu, analogicznie do `cpu_frequencies()`.

### 3.3 🔵 Przypięcie wątków emulacji do dużych rdzeni — eksperyment, nie oczywistość

- Dziś: brak (`sysinfo::tune()` to `_WIN32`; `main.cpp:932`).
- Na Androidzie: `sched_setaffinity` na zbiór rdzeni X4 dla wątków roboczych. Ryzyko: scheduler Androida
  i cpuset mogą to ignorować albo karać (gorący klaster), a przy 7 wątkach przypięcie do 1–2 rdzeni może
  **zaszkodzić** — 3D na CPU używa 7 wątków i skaluje się dobrze (w piątym przebiegu: 21,09 ms).
- Wniosek: próbować tylko po §3.2 i tylko jako ustawienie do wyłączenia (`VF2_AFFINITY=1`), z pomiarem.

### 3.4 🟡 Higiena dokumentów (nie kod)

- `vf2 gemini.md` w obecnej formie nadaje się tylko jako źródło hipotez. Gdyby miał zostać w repo, trzeba go
  przepisać (choćby po to, żeby nie kopiować z niego zepsutego manifestu i hintów SDL — §1.4).
- `vf2-recomp-android.md` jest w porządku; po dopisaniu, że u nas FMA jest rozwiązane przez `precise`,
  a orientacja/ROM/dotyk są zrobione, mógłby zostać jako „jak myśleliśmy o porcie".

---

## 4. Podsumowanie: co odrzucić, co zostawić

**Odrzucić definitywnie (🔴):**
1. `SDL_SetHint(SDL_HINT_RENDER_DRIVER, "vulkan")` — nie działa (nazwa jawna) albo przełącza na stary renderer.
2. `uses-feature … required="true"` dla Vulkan — zaprzecza własnemu fallbackowi na CPU.
3. `adb shell setprop persist.vendor.powerhal.perf_mode 1` — wymaga roota.

**Uznać za bezskuteczne/no-op (🟡):** `SDL_HINT_VIDEO_DOUBLE_BUFFER`, `SDL_HINT_RENDER_VSYNC`,
`minImageCount = 3`, subpassy, ręczne load/store ops, `VkPipelineCache`, Swappy, `hasFragileUserData`.

**Uznać za prawdziwe i już zrobione (✅):** `precise`/`NoContraction`, zakaz `mediump`, grupy 64-wątkowe,
`presentMode = MAILBOX`, `appCategory="game"`, `abiFilters arm64-v8a`, `uses-feature vulkan` (z `required="false"`),
SAF na ROM, dotyk + menu, test przy starcie z fallbackiem, orientacja pozioma, narzędzia Arm/Streamline/CapsViewer.

**Zostawić do zrobienia (🔵):** ADPF (§3.1) — jedyna realna nowa rzecz; do tego eksperyment z afinicją (§3.3)
i odczyt zegarów GPU, który rozstrzygnie §3.2.

**Korekta mojej własnej notatki:** pomysł „spróbować fp16 w compute shaderze 3D" z
`docs/ARM-IMMORTALIS-G720.md` **był błędny** — `gpu3d_sdlgpu.cpp:120-121` jest zawsze „exact", a fp16 łamie
bit-dokładność, której pilnuje `precise` i test przy starcie. `vf2-recomp-android.md` ma tu rację, nie ja.
