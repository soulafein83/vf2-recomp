# Virtua Fighter 2 Recomp na Androida

Notatki o przeniesieniu projektu [mr-finster/vf2-recomp](https://github.com/mr-finster/vf2-recomp) na Androida, ze szczególnym uwzględnieniem GPU Mali Immortalis-G720 MC12.

> Uwaga: opieram się na README projektu i opisach w sieci. Kodu źródłowego nie czytałem, więc szczegóły implementacyjne trzeba zweryfikować.

---

## 1. Android Studio na CachyOS

Android Studio to oficjalne IDE od Google (oparte na IntelliJ IDEA): edytor Kotlin/Java, emulator, debugger, profiler, SDK Manager i Gradle.

```bash
paru -S android-studio     # albo: yay -S android-studio
```

- **Emulator** wymaga KVM: `sudo usermod -aG kvm $USER`, potem restart sesji.
- **Wayland:** przy problemach z renderowaniem pomaga XWayland albo `-Dawt.toolkit.name=WLToolkit` w opcjach VM.
- **RAM:** minimum 8 GB, wygodnie 16 GB+.
- Do samego debugowania na telefonie wystarczy `android-tools` (adb/fastboot).
- NDK i CMake instaluje się z SDK Managera.

---

## 2. Czy port jest realny?

**Tak.** Co przemawia za:

- Recompiler generuje natywny kod **ARM64** lub x86-64 (SLJIT), a backend ARM64 jest już sprawdzony na Apple silicon. Telefony mają ten sam typ CPU.
- Host używa **SDL 3**, który oficjalnie wspiera Androida (okno, wejście, audio, kontrolery).
- Renderer GPU ma ścieżkę **Vulkan** przez SDL_GPU, a fallback na CPU już istnieje.
- Port VF2 na Nintendo Switch (Thorhax, VF2-NX-Modern, ARM64) osiąga stałe ok. 57,5 FPS. To inny kod, ale pokazuje, że wydajność CPU na ARM-ie wystarcza.

---

## 3. Co trzeba zrobić

1. **Build:** projekt używa `Makefile`, a Android potrzebuje NDK + CMake (`arm64-v8a`) i szkieletu Gradle z `android-project` z SDL3. Kod MAME i C++20 powinny się skompilować clangiem z NDK. Zależności (zlib, FreeType) trzeba zbudować pod NDK.
2. **JIT:** SLJIT wymaga pamięci wykonywalnej (`mmap` z `PROT_EXEC`). Na zwykłych aplikacjach zazwyczaj działa, ale warto to sprawdzić wcześnie na docelowym sprzęcie.
3. **ROM:** Android ogranicza dostęp do plików. `vf2.zip` należy wybrać przez Storage Access Framework (SDL ma file dialog) i skopiować do prywatnego katalogu aplikacji. Projekt robi już podobnie na Macu i Windowsie.
4. **Sterowanie:** nakładki dotykowe (guard / punch / kick + D-pad) oraz dostosowanie menu RmlUi do dotyku. Obsługa pada już jest, więc od niej warto zacząć.
5. **Wydajność:** rasteryzer na compute shaderach pod Adreno/Mali może zachowywać się różnie. Test porównawczy przy starcie wykrywa różnice i przełącza na CPU, więc zacznij od 1x i trybu CPU. Przydaje się przypinanie wątków do dużych rdzeni.

### Sensowna kolejność

1. Skompiluj samo `vf2run` (headless) przez NDK i uruchom przez `adb shell`. To sprawdzi JIT i emulację bez UI.
2. Dodaj host SDL3 z obrazem i dźwiękiem, sterowanie padem Bluetooth.
3. Dopiero potem dotyk, menu i GPU.

---

## 4. Mali Immortalis-G720 MC12: ustawienia i pułapki

Układ nie wymaga specjalnych flag. Obsługuje Vulkan 1.3 ze standardowym sterownikiem, więc SDL_GPU powinien się uruchomić bez dodatkowej konfiguracji. Uwagi dotyczą szczegółów.

### 4.1 Bit-dokładność rasteryzera (największe ryzyko)

Projekt wymaga, by GPU liczyło dokładnie jak CPU (bez fused multiply-add).

- Sterowniki Mali chętnie łączą mnożenie i dodawanie w FMA. W SPIR-V oznaczaj wyniki `NoContraction` (w HLSL/DXC: `precise`).
- Unikaj `RelaxedPrecision` (mediump), bo Mali liczy wtedy w fp16.
- Konwersja float→int dla wartości spoza zakresu na ARM daje inny wynik niż na x86, a README mówi o naśladowaniu zachowania x86.
- Jeśli test porównawczy przy starcie się nie zgodzi, gra po cichu spadnie na CPU i wydajność spadnie bez żadnego komunikatu błędu.

### 4.2 Compute na Mali

- Rozmiar grupy roboczej dobieraj jako wielokrotność 16 (szerokość warpa), np. 64.
- Pamięć współdzielona leży w cache, więc nie licz na duży zysk z jej nadużywania.
- TBDR (mocna strona Mali przy klasycznym renderowaniu) nic tu nie daje, bo rasteryzer działa na compute.

### 4.3 Integracja z Androidem

- Zapisuj `VkPipelineCache` na dysk, inaczej pierwsze uruchomienie shaderów będzie szarpać.
- Dla stabilnego klatkowania użyj biblioteki Swappy (Android Frame Pacing).
- Dodaj Android Performance Hint API, bo bez tego telefon dławi taktowanie przy dłuższej grze.
- Przypnij wątki emulacji do rdzeni X4 (G720 MC12 to zwykle Dimensity 9300).
- Wymuś orientację poziomą i sprawdź, jak SDL3 obsługuje pre-rotation.

### 4.4 Narzędzia diagnostyczne

- Arm Performance Studio (Streamline), do profilowania GPU
- Mali Offline Compiler, do analizy shaderów
- Warstwy walidacyjne Vulkana (`libVkLayer_khronos_validation.so` w APK)
- Vulkan Hardware Capability Viewer, żeby zobaczyć, jakie funkcje i formaty storage image ma sterownik

---

## 5. Zalecenie

Zacznij od trybu CPU. GPU dokładaj dopiero po rozwiązaniu problemu z FMA.

## Linki

- Repozytorium: https://github.com/mr-finster/vf2-recomp
- Strona projektu: https://vf2-recomp.pages.dev/
- Wymagany jest własny zestaw ROM `vf2.zip` z MAME (Version 2.1).
