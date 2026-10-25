# LPU Touch (v23.50) — Full Dark Mode Theme Architecture & Split-APK QR Crash Fix

**Submitted for:** LPU EDU Revolution (Project / Custom Application & Division of Infotech Contribution)  
**Target Application:** LPU Touch Android App (`ums.lovely.university`, v23.50)  
**Tech Stack:** Angular 16 + Ionic 7 + Capacitor 5 + Android Native (`classes.dex` / `resources.arsc`)

---

## 1. Executive Summary

The official **LPU Touch** mobile application (`v23.50`) ships with a light-only interface (`color-scheme` meta tag commented out in `index.html`, light Ionic `:root` variables, and 450+ component-scoped hardcoded light/dark colors across 95+ lazy-loaded Angular screen modules).

This project delivers a **complete, production-ready Dark Mode architecture** for LPU Touch along with **three critical bug fixes** discovered during full-app verification:

1. **Global Ionic 7 Dark Palette (`02_variables.scss` & `03_global_dark_mode.scss`):** Establishes a cohesive OLED-friendly dark surface hierarchy (`#121212` primary background, `#1e1e1e` elevated cards/items, `#262626` interactive inputs/modals) while preserving LPU's signature peach-orange gradient accents (`#fe8573` to `#ffd57a`).
2. **Component-Level Hardcoded Color Audit (`04_Component_Color_Audit_Table.md`):** Identifies and resolves 450+ hardcoded dark text rules (`#000`, `#323232`, `#333`, `#404040`, `#606060`, `#573939`) and light container backgrounds (`#fff`, `#f4f5f8`, `#f2f2f2`, `#eee`) across all 206 webpack JS bundles that previously bypassed Ionic CSS variables.
3. **Ionic Floating Label & QR Code Scannability Fixes:** Fixes an Ionic DOM stacking issue on **Student Laundry** (`7320.js`) where opaque `<ion-input>` backgrounds occluded unfocused `<ion-label position="floating">` elements, and adds a high-contrast white quiet zone (`padding: 10px; background: #ffffff`) around generated QR codes inside `<ion-alert>` popups.
4. **Native Android BarcodeScanner Split-APK Crash Fix:** Diagnoses and resolves an immediate `Resources$NotFoundException` crash (`0x7f0c0022`, `res/layout/capture.xml`) in `com.google.zxing.client.android.CaptureActivity` affecting standalone APK installations.

---

## 2. Deep-Dive: The 3 Engineering Bugs Solved

### Bug 1: Immediate App Crash on "SCAN QR CODE" (`CaptureActivity` Missing Split Resource)
* **Affected Screens:** Placement Scanner (`6206.js`), Classroom Scanner (`1359.js`), Uni Hospital Helpline (`3246.js`), Mess Food Scanner (`3415.js`), Orientation (`5625.js`), and Hostel Attendance QR (`4569.js`).
* **Root Cause:** All 6 scanner modules invoke `this.barcodeScanner.scan()`, which launches `com.google.zxing.client.android.CaptureActivity` (from `barcodescanner-release-2.1.5.aar`). In `CaptureActivity.onCreate()`, the activity calls `setContentView(0x7f0c0022)` (`R.layout.capture`) and references `0x7f080128` (`R.drawable.ic_launcher`). Because `barcodescanner-release-2.1.5.aar` includes both `res/layout/capture.xml` and `res/layout-ldpi/capture.xml` (a density qualifier), Android App Bundle (`bundletool`) strips `res/layout/capture.xml` out of `base.apk` into `split_config.<density>.apk`, leaving `0x7f0c0022` and `0x7f080128` marked as `0xFFFFFFFF` (`MISSING_ENTRY`) in `base.apk`'s `resources.arsc`.
* **Fix Implemented:**
  * **In the Standalone APK:** Reconstructed the binary AXML `res/layout/capture.xml` containing all 14 required ZXing view IDs (`0x7f090198` `preview_view`, `0x7f09023b` `viewfinder_view`, `0x7f0901a5` `result_view`, `0x7f0901f0` `status_view`, `0x7f0900de` `flip_button`, `0x7f090220` `torch_button`, etc.) and linked `0x7f0c0022` and `0x7f080128` in `resources.arsc`.
  * **Recommended Source Fix for Division of Infotech (`android/app/build.gradle`):**
    ```groovy
    android {
        bundle {
            density {
                enableSplit = false // Prevents capture.xml from being stripped from base.apk
            }
        }
    }
    ```

### Bug 2: Student Laundry Items Invisible Under Floating Labels (`7320.js`)
* **Affected Screen:** Student Laundry (`StudentLaundryPageModule`, `7320.ad3cd2d1b4ae3c3a.js`).
* **Root Cause:** Each clothing row uses `<ion-item><ion-label position="floating">{{ e.name }}</ion-label><ion-input [formControlName]="'input_' + e.id"></ion-input></ion-item>`. When an input is unfocused and empty, Ionic translates `<ion-label position="floating">` downward (`translate3d(0, 29px, 0)`) into the `<ion-input>` bounding box. Because `<ion-input>` appears after `<ion-label>` in DOM order, applying a solid dark background to `ion-input` paints an opaque rectangle directly over the floating label text, making loaded laundry items appear blank.
* **Fix Implemented (`03_global_dark_mode.scss`):**
  ```scss
  ion-item ion-input,
  ion-item ion-textarea,
  ion-item ion-select {
    --background: transparent !important;
    background: transparent !important;
    background-color: transparent !important;
  }

  ion-item ion-label {
    --color: #e2e8f0 !important;
    color: #e2e8f0 !important;
    z-index: 2 !important;
    position: relative !important;
  }
  ```

### Bug 3: Hardcoded Low-Contrast Text in Angular Component Styles & `consts`
* **Affected Screens:** Exams Available / Sitting Plan (`6582.js`), View Marks (`9382.js` & `8824.js`), Announcement (`86.js`), Doctor Appointment (`9520.js`), RMS Status (`2825.js`), and 80+ other modules.
* **Root Cause:** Many Angular components define scoped styles without CSS variables—for example, `.course-name { color: #606060; }` in *View Marks* (`9382.js`), inline `consts: [[2, "font-weight", "bold", "color", "#573939"]]` for `Exam Date:` in *Sitting Plan* (`6582.js`), and `--ion-color-lightgray` on `Room No :` badges.
* **Fix Implemented:** Replaced all low-luminance hardcoded dark text values (`luminance < 0.50`) on dark surfaces with `#f1f5f9` (primary text), `#e2e8f0` (secondary text), and `#ffd57a` (warm gold accent for `Exam Date:`), while preserving `#1a1a1a` dark text on bright peach/orange gradient pills (`linear-gradient(90deg, #fe8573, #ffd57a)`).

---

## 3. Repository Structure

* `README.md` / `01_Technical_Writeup_and_README.md` — Full engineering case study and root-cause analysis.
* `02_variables.scss` — Drop-in Ionic 7 dark theme CSS custom properties for `src/theme/variables.scss`.
* `03_global_dark_mode.scss` — Drop-in global SCSS overrides for `src/global.scss`.
* `04_Component_Color_Audit_Table.md` — Screen-by-screen audit of hardcoded colors in Angular component files.
* `screenshots/` — Before (Light Mode) and After (Dark Mode v4) verification screenshots.
