Proszę, oto kompletna dokumentacja w formacie Markdown. Skopiuj poniższy blok kodu i zapisz go jako plik z rozszerzeniem .md (np. mali_immortalis_vulkan_guide.md):Markdown# Optymalizacja GPU Mali Immortalis-G720 MC12 i Vulkan na Androidzie (Xiaomi 14T Pro / Dimensity 9300+)

Dokumentacja techniczna dotycząca optymalizacji natywnych aplikacji, portów oraz rekompilacji (np. `vf2recomp` / SDL) na procesorach MediaTek Dimensity 9300+ z układem graficznym Mali Immortalis-G720.

---

## 1. Problemy architektoniczne Mali (TBDR)

Układ Mali wykorzystuje architekturę **Tile-Based Deferred Rendering (TBDR)**. Niewłaściwa obsługa pamięci kafelkowej SRAM w kodzie renderera powoduje drastyczne wąskie gardła:

* **Tile Reloads (Przeładowanie kafelków):** Jeśli bufor klatki nie jest jawnie czyszczony na początku przebiegu, GPU jest zmuszone do przepisywania danych z RAM/VRAM do pamiąci SRAM.
* **V-Sync / Fence Stalls:** W trybie podwójnego buforowania sterownik MediaTek agresywnie uśpiewa GPU, oczekując na płotek synchronizacji ekranu (`Fence`).
* **PowerHAL Throttling:** Niezidentyfikowane aplikacje (spoza listy gier 3D w HyperOS) są traktowane jako procesy tła, co sztucznie ogranicza zegary GPU z 1300 MHz do ~300–400 MHz.

---

## 2. Kluczowe optymalizacje API Vulkan

### Zarządzanie buforami (`VkRenderPass`)
* **Ustawienie Load/Store Operations:** Na początku każdego `VkRenderPass` ustawiaj:
  ```c
  VkAttachmentDescription colorAttachment{};
  colorAttachment.loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR; // lub VK_ATTACHMENT_LOAD_OP_DONT_CARE
  colorAttachment.storeOp = VK_ATTACHMENT_STORE_OP_STORE;
Unikaj stosowania VK_ATTACHMENT_LOAD_OP_LOAD dla buforów wyjściowych.Podprzepusty (Subpasses)Jeśli gra wykonuje post-processing, łącz operacje wewnątrz jednego VkRenderPass za pomocą subpassów z flagą VK_DEPENDENCY_BY_REGION_BIT.Konfiguracja Swapchaina (Triple Buffering & Mailbox)minImageCount: Ustaw sztywno na 3 (Potrójne buforowanie).presentMode: Preferuj VK_PRESENT_MODE_MAILBOX_KHR zamiast VK_PRESENT_MODE_FIFO_KHR.3. Konfiguracja SDL2 / SDL3W przypadku portów opartych na bibliotece SDL, wymuś poniższe ustawienia w kodzie przed utworzeniem okna:C// Wymuszenie backendu Vulkan
SDL_SetHint(SDL_HINT_RENDER_DRIVER, "vulkan");

// Wyłączenie narzutu V-Sync na poziomie SDL
SDL_SetHint(SDL_HINT_RENDER_VSYNC, "0");

// Bezpośrednie przekazywanie klatek
SDL_SetHint(SDL_HINT_VIDEO_DOUBLE_BUFFER, "1");
4. Konfiguracja Projektu Android (AndroidManifest & Gradle)AndroidManifest.xmlNależy jawnie poinformować system HyperOS, że aplikacja jest grą 3D korzystającą z Vulkan:XML<manifest xmlns:android="[http://schemas.android.com/apk/res/android](http://schemas.android.com/apk/res/android)">
    
    <uses-feature 
        android:name="android.hardware.vulkan.version" 
        android:version="0x00403000" 
        android:required="true" />

    <application
        android:appCategory="game"
        android:hasFragileUserData="false"
        ...>
        ...
    </application>
</manifest>
build.gradle (Kompilacja NDK)Dimensity 9300+ nie posiada fizycznych rdzeni 32-bitowych. Aplikacja musi być skompilowana wyłącznie pod architektury 64-bitowe:Groovyandroid {
    defaultConfig {
        ndk {
            abiFilters 'arm64-v8a'
        }
    }
}
5. Optymalizacja w systemie HyperOS / MIUIPreferencje sterownika graficznego:Przejdź do: Opcje programistyczne $\rightarrow$ Preferencje sterownika graficznego.Znajdź aplikację i zmień z Domyślny na Sterownik graficzny systemu.Game Turbo (Panel dotykowy):Dodaj aplikację do usługi Game Turbo.W ustawieniach wydajności gry włącz Tryb wysokiej wydajności oraz Optymalizację wydajności GPU.Wymuszenie stałych zegarów (ADB - Testy):Wykonaj polecenie w terminalu, aby uniemożliwić agresywne oszczędzanie energii przez MediaTek PowerHAL:Bashadb shell setprop persist.vendor.powerhal.perf_mode 1

<FollowUp>
Czy potrzebujesz, żebym dodał do tego dokumentu specyficzne skrypty budowania (np. dla CMake lub NDK Build) czy opis konfiguracji konkretnego narzędzia profilującego, takiego jak Arm Graphics Analyzer?
</FollowUp>
