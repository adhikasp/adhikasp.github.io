---
title: "Vulkan - Graphic Library Generasi Baru"
layout: single
excerpt: "Vulkan adalah graphic library generasi terbaru yang dikembangkan dari
OpenGL oleh konsorsium non-profit Amerika, Khronos Group. Vulkan diunggulkan
memiliki overhead rendah, cross-platform, serta optimal dalam multithreading
CPU."
sitemap: true
author_profile: false
category: "college"
tags:
  - "Indonesia"
  - "hardware"
---

Vulkan adalah graphic library generasi terbaru yang dikembangkan dari OpenGL
oleh konsorsium non-profit Amerika, Khronos Group. Vulkan diunggulkan memiliki
overhead rendah, cross-platform, serta optimal dalam multithreading CPU.
Seperti pendahulunya (OpenGL), Vulkan ditargetkan untuk digunakan dalam aplikasi
yang membutuhkan grafik 3D realtime seperti game dan berbagai media interaktif.


# Kelebihan

Vulkan didesain untuk mengatasi kekurangan-kekurangan dari OpenGL. Fitur-fitur
dari Vulkan antara lain adalah sebagai berikut:

-	Overhead rendah
-	Memberikan akses yang lebih leluasa terhadap hardware GPU
-	Penggunaan CPU yang lebih rendah
-	Pada OpenGL, shaders (pewarnaan suatu objek grafik) ditulis dengan menggunakan
  bahasa khusus bernama GLSL. Hal ini menyebabkan seluruh driver hardware yang
  ingin mendukung OpenGL harus membuat kompilernya masing-masing. Vulkan
  menggunakan format binary penengah SPIR-V, ini berarti shaders dapat
  di*pre-compile* ke format khusus oleh pembuat aplikasi sehingga penyedia
  driver tidak perlu menyediakan kompiler bahasa tertentu.
-	API yang cross-platform mendukung baik perangkat mobile maupun kartu grafis
  tingkat tinggi.
-	Mendukung multithreading, vulkan dapat memaksimalkan kinerjanya dengan
  menggunakan beberapa core CPU sekaligus.


# Sejarah Pengembangan

Khronos Group memulai proyek pengembangan API grafik generasi lanjut pada musim
panas 2014. Pada SIGGRAPH (Special Interest Group on GRAPHics and Interactive
Techniques) 2014, proyek ini diumumkan sekaligus mengundang partisipasi. Nama
Vulkan baru diumumkan pada Game Developers Conference 2015, walaupun sebelumnya
beredar rumor bahwa API ini disebut ‘glNext’, sebagai signifikasi bahwa ini
adalah generasi terbaru dari OpenGL.


# Kompatibilitas

Spesifikasi menyebutkan bahwa Vulkan dapat dijalankan pada semua hardware yang
mendukung OpenGL 4.x (GPU keluaran 2010) atau OpenGL ES 3.1 (mobile device
keluaran 2014 keatas). Tidak ada batasan OS, dengan kata lain Vulkan ditargetkan
untuk dapat bekerja baik di platform Windows, Linux, Mac, bahkan mobile seperti
Android.


# Dukungan Vendor

Vulkan didukung oleh perusahaan-perusahaan raksasa di bidang IT (Google,
  Qualcomm, Samsung, VMWare, NVDIA, AMD, Apple, ARM, Broadcom, Intel, dll),
  dari bidang hiburan (Pixar, Lucasfilm, Sony), serta studio game besar
  (EA, Blizzard, Oxide, Valve, Unity).


# Referensi

- https://www.khronos.org/vulkan, diakses 30 Agustus 2015
Kessenich, John. "An Introduction to SPIR-V" (<https://www.khronos.org/registry/spir-v/papers/WhitePaper.pdf>). Khronos Group. , diakses 30 Agustus 2015.
- "More on Vulkan and SPIR - V: The future of high-performance graphics" (PDF). Khronos Group.
Khronos Group (June 2015). "Vulkan Overview" (PDF).
