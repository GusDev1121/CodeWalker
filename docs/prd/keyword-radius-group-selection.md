# Keyword Radius Group Selection (Key H)

Status: Draft · Date: 2026-11-15 · Owner: GusDev1121

## Problem
Map editor yang mau mengedit banyak props sejenis (mis. semua bangku atau lampu jalan di satu area) sekarang harus Ctrl+klik satu per satu di WorldForm. Di area padat, cara ini lambat dan mudah salah: props lain yang tidak diinginkan ikut terpilih, atau ada props yang terlewat. Fitur ini menambahkan brush seleksi berbentuk bulatan radius yang hanya menambahkan props dengan nama model yang cocok dengan kata kunci ke group selection (`MultipleSelectionItems`).

## Goals
- Memilih N props dengan nama model yang cocok di satu area cukup dengan satu kali sapuan mouse, bukan N kali Ctrl+klik.
- Hanya props yang nama archetype-nya mengandung kata kunci yang ikut masuk seleksi. Tidak boleh ada false positive dari props lain.
- Hasil seleksi memakai group selection yang sudah ada, sehingga Move/Rotate/Scale, Delete, Copy dan EditMultiPanel langsung bisa dipakai tanpa perubahan.

## Non-goals
- Pencocokan wildcard/regex atau pencocokan persis. Versi ini hanya mendukung "mengandung" tanpa membedakan huruf besar-kecil.
- Seleksi tipe selain `YmapEntityDef`, seperti car generator, grass, collision, nav mesh, path atau light.
- Mode brush untuk mengurangi seleksi (deselect) dengan sapuan.
- Undo/redo untuk seleksi. Seleksi memang tidak masuk ke undo stack saat ini.
- Perubahan pada format file. Fitur ini hanya ada di editor.

## Users & scenarios
- Map editor ingin memilih semua `prop_bench_*` di sebuah taman supaya bisa memindahkan atau menghapusnya sekaligus.
- Modder ingin memilih semua lampu jalan (`prop_streetlight`) di satu ruas jalan supaya bisa menyalinnya ke ymap project.
- Map editor ingin memperbesar atau memperkecil radius saat menyapu supaya tidak ikut menyentuh props di blok sebelah.

## Requirements
| ID | Requirement | Touches (file/class) | Acceptance check |
|----|-------------|----------------------|------------------|
| R1 | Tambah key binding baru "Toggle Keyword Select" dengan default `Keys.H`, tersimpan di settings dan bisa diubah. | `CodeWalker/Utils/InputUtils.cs` (`KeyBindings`: field, `SetBinding`, `GetSetting`), `CodeWalker/World/SettingsForm.cs` (`AddKeyBinding`), `Properties/Settings.settings` (nilai default `KeyBindings`) | Settings → Controls menampilkan "Toggle Keyword Select: H". Ubah ke J, restart, binding tetap J. |
| R2 | Menekan H di WorldForm akan menyalakan atau mematikan mode keyword select (toggle). Saat ON, tombol toolbar terkait dalam keadaan checked. | `CodeWalker/WorldForm.cs` (key-down handler, di sebelah penanganan `ToggleMouseSelect`), state baru `KeywordSelectEnabled` | Tekan H: mode ON dan tombol checked. Tekan H lagi: mode OFF. Mengetik di TextBox mana pun tidak ikut men-toggle. |
| R3 | Saat mode ON dan mouse mengenai collision, sebuah lingkaran radius dirender di titik hit (warna berbeda dari instance brush yang hijau). | `WorldForm.cs` (`RenderSelection` / blok render `ControlBrushEnabled` di sekitar baris 1329), memakai `Renderer.RenderBrushRadiusOutline` yang sudah ada | Lingkaran mengikuti kursor di permukaan tanah dan diameternya sesuai nilai radius. |
| R4 | Di toolbar WorldForm ada TextBox kata kunci dan NumericUpDown radius (default 5.0, min 0.5, max 200). | `WorldForm.cs` + `WorldForm.Designer.cs` (toolbar) | Kedua kontrol tampil, nilainya terbaca saat mode berjalan, dan nilai di luar batas ditolak. |
| R5 | Ctrl+scroll mouse saat mode ON mengubah radius dan tidak men-zoom kamera. Scroll tanpa Ctrl tetap berfungsi seperti biasa. | `WorldForm.cs` (`WorldForm_MouseWheel`, baris ~6428) | Ctrl+scroll: angka radius dan lingkaran berubah, kamera diam. Scroll biasa: kamera zoom. |
| R6 | Saat mode ON dan klik kiri ditahan, setiap frame semua `YmapEntityDef` di ymap yang terlihat yang bounding sphere-nya (`BSCenter`, `BSRadius`) beririsan dengan bulatan brush **dan** nama archetype-nya mengandung kata kunci (case-insensitive) akan ditambahkan ke group selection. | `WorldForm.cs` (iterasi `Renderer.VisibleYmaps`, lalu `SelectItem(..., addSelection: true)` / `SelectMulti`), `CodeWalker.Core/GameFiles/FileTypes/YmapFile.cs` (`YmapEntityDef`, hanya dibaca) | Kata kunci `bench`, radius 10, sapu taman: hanya props yang namanya mengandung "bench" yang masuk `SelectedItem.MultipleSelectionItems`. Props lain di dalam radius tidak ikut. |
| R6c | Di toolbar ada pilihan bentuk brush: **Bola 3D** (default) atau **Lingkaran datar**. Bola 3D mengukur jarak penuh XYZ. Lingkaran datar hanya mengukur jarak XY, sehingga ketinggian diabaikan. Pilihan ini tersimpan di settings. | `WorldForm.cs` + `WorldForm.Designer.cs` (ComboBox/toggle di toolbar), `Properties/Settings.settings` (setting baru) | Lingkaran datar, sapu di bawah jembatan: props di atas jembatan dengan nama yang cocok ikut terpilih. Bola 3D: props itu tidak ikut. Restart: pilihan tetap sama. |
| R6d | Kinerja: pemindaian hanya dijalankan saat klik kiri ditekan dan setiap kali pusat brush bergeser ≥ 25% dari radius, tidak di setiap frame. Setiap entity diperiksa dengan tes jarak yang murah dulu, baru kemudian namanya. Culling memakai extents ymap (`entitiesExtentsMin/Max`) **tidak dipakai**: extents hanya dihitung ulang secara manual (`CalcExtents`), jadi props yang dipindah keluar dari extents lama bisa terlewat. | `WorldForm.cs` (`KeywordSelectSweep`, `KeywordSelectMatches`) | Sapu terus-menerus di pusat kota Los Santos dengan radius 50: FPS tidak turun lebih dari 20% dibanding mode OFF. |
| R6a | Hanya entity versi detail penuh yang ikut: `_CEntityDef.lodLevel` bernilai `LODTYPES_DEPTH_HD` atau `LODTYPES_DEPTH_ORPHANHD`. Entity LOD/SLOD dilewati walaupun namanya cocok. | `WorldForm.cs` (filter di loop R6), `rage__eLodType` di `CodeWalker.Core/GameFiles/MetaTypes/MetaTypes.cs` | Sapu area yang punya model HD dan model LOD dengan nama yang mirip: hanya model HD yang masuk seleksi. |
| R6b | Seleksi tidak bergantung pada project. Fitur ini tetap berjalan tanpa ProjectForm terbuka, entity dari ymap mana pun yang sedang dimuat bisa dipilih, dan ymap-nya tidak otomatis ditambahkan ke project. | `WorldForm.cs` (`SelectItem` / `SelectMulti`, `notifyProject`) | Tutup ProjectForm lalu sapu: props terpilih. Jumlah ymap di project tidak berubah. |
| R7 | Entity yang sudah terseleksi tidak ditambahkan lagi (tidak ada duplikat). Seleksi yang sudah ada sebelum menyapu tetap dipertahankan. | `WorldForm.cs` (`SelectItem` sudah punya logika add, perlu dicek duplikatnya) | Sapu area yang sama dua kali: jumlah item di seleksi tidak bertambah. |
| R8 | Jika kata kunci kosong, sapuan tidak menyeleksi apa pun dan status bar menampilkan pesan. | `WorldForm.cs` (status label) | Kosongkan kata kunci lalu sapu: seleksi tidak berubah dan pesan muncul. |
| R9 | Pencocokan memakai nama archetype: `Archetype?.Name` jika tersedia, atau `_CEntityDef.archetypeName` jika archetype belum dimuat. | `YmapEntityDef.Archetype` / `_CEntityDef.archetypeName` | Entity yang archetype-nya belum ter-cache tetap bisa cocok lewat nama hash yang sudah di-resolve. |
| R10 | Mode keyword select tidak boleh aktif bersamaan dengan instance brush (`ControlBrushEnabled`). Instance brush hanya aktif selama Ctrl ditahan, jadi selama keyword select ON, instance brush tidak bisa aktif. | `WorldForm.cs` (`BeginMouseHitTest`) | Keyword select ON, panel instance brush di project aktif, tahan Ctrl: lingkaran hijau tidak muncul dan tidak ada instance yang dicat. |
| R11 | Selama mode ON, kamera tetap bisa digerakkan dengan WASD dan klik kanan. Klik kiri dipakai untuk menyapu, bukan untuk seleksi tunggal. | `WorldForm.cs` (mouse down/move handler) | Saat mode ON, kamera bisa bergerak dan klik kiri tidak memicu `SelectItem` tunggal. |

## Game-file & compatibility impact
Fitur ini tidak membaca atau menulis format baru. Fitur hanya membaca `YmapEntityDef` dari ymap yang sudah dimuat (gen8 dan gen9 sama saja) dan tidak memengaruhi round-trip XML. Satu-satunya data yang tersimpan adalah `Settings.Default.KeyBindings`. Settings lama yang belum punya entri baru harus tetap jalan dengan default H. Constructor `KeyBindings` sudah aman untuk ini karena memakai default field lalu menimpanya hanya dengan entri yang ada. Risiko kehilangan data saat save tidak ada.

## UX
- **Shortcut:** H men-toggle mode keyword select dan bisa diatur ulang di Settings → Controls.
- **Toolbar WorldForm:** satu tombol toggle "Keyword Select (H)", TextBox "Keyword", NumericUpDown "Radius", dan pilihan bentuk brush "Bola 3D / Lingkaran datar".
- **Viewport:** lingkaran radius mengikuti kursor. Klik kiri lalu tahan dan sapu untuk menambah seleksi. Ctrl+scroll mengubah radius.
- **Status bar:** menampilkan jumlah item yang terseleksi setelah sapuan dan pesan saat kata kunci kosong.
- Hasil seleksi muncul di panel yang sudah ada (`SelEntityPropertyGrid` / `EditMultiPanel` di ProjectForm).

## Risks & open questions
- **Performa (diputuskan, lihat R6d):** ambang FPS 20% di R6d hanyalah perkiraan. Kalau pengukuran nyata menunjukkan masih berat, langkah berikutnya adalah membuat index spasial. Langkah ini sengaja tidak dimasukkan sekarang.
- **Definisi "menyentuh" (diputuskan):** props terpilih kalau batas luar modelnya (bounding sphere, `BSCenter`/`BSRadius`) bersinggungan dengan brush (R6).
- **Bentuk brush (diputuskan, lihat R6c):** user memilih sendiri antara Bola 3D dan Lingkaran datar.
- **Entity di luar project (diputuskan: bebas, lihat R6b):** perlu dicek apakah aksi lanjutan seperti move, delete dan copy pada entity yang ymap-nya tidak ada di project berjalan sama seperti seleksi tunggal di CodeWalker sekarang, atau malah memunculkan prompt untuk menambahkan ymap ke project. *Dijawab oleh: developer.*
- **Konflik key H:** tidak ada binding H di WorldForm saat ini (H hanya dipakai di `ExploreForm`). Tetap perlu dicek di `PedsForm` dan `ModelForm` jika keduanya memakai handler yang sama.

## Milestones
1. Key binding H, toggle mode dan render lingkaran brush (R1, R2, R3, R10).
2. Kontrol kata kunci dan radius di toolbar dan Ctrl+scroll (R4, R5).
3. Logika sapuan, pencocokan dan penambahan ke group selection (R6, R6a, R6b, R6c, R6d, R7, R8, R9, R11).
4. Profiling di area padat (mis. pusat kota Los Santos) dan optimasi jika ada kebutuhan.
