## Summary

---

### 4.1 Spesifikasi Sistem

- Skip

### 4.2 Hasil Pra-Pemrosesan Data

- 2002 gambar + mask, resolusi 1280 x 720
- Mask pada dataset ga semuanya 0 dan 255, ada yg diantara 0 dan 255.
- Jadi, pake threshold 127
- Ada label "yes" dan "no" cuman dipake buat stratifikasi
- Rata" piksel 5.46% dan 2.44% show class imbalance

### 4.2.2. Hasil Standarisasi dan Encoding Data

- Kolom di CSV jadi ada: `image_name`, `presence`, dan `fg_pct`
- `presence` jadi kolom `has_plastic` yang isinya 1 dan 0.
- Di setiap baris ada `mask_path` dan `image_path`.

### 4.2.3. Hasil Transformasi dan Augmentasi Citra

- Resize sama Normalize ke semua data test
- Transform + Color Jitter only ke train
  - Alasannya:
    - Color Jitter dipakai di sini biar model terbiasa sama variasi cahaya yang mempengaruhi warna
    - Transform juga dipakai di train doang biar model terbiasa sama posisi foto
- Gambar diubah jadi 512 x 512
  - Alasannya:
    - Memory GPU (T4) gaakan kuat + training lama
    - Batch Size kemungkinan besar bakal butuh lebih banyak
    - 512 x 512 sering dipakai as default (standar)
    - 17 model pakai size yang sama jadi adil" aja dan fair comparison
    - Kelemahannya aspect rationya berubah jadi objek bakal gepeng
    - Detail" bakal ilang
    - Tapi karena semua model pakai hal yang sama jadi fair" aja
- mean ImageNet sama Standar Deviation dipake buat normalization
- gambar-nya di transform geometrically (rotate 90, or something probability wise)
  - Alasannya:
    - Geometrical transform karna UAV bisa
- gambar-nya abis tu di color jitter (diganti hue, tint, dll)
  - Hue-nya terlalu jauh tar takut plastik jadi kayak objek lain gitu.
- Kenapa poin Augmentasi-nya segini? ini jadi standard di computer vision augmentation, tapi kita selalu pakai yang sama di across 17 model jadi sebenernya ga terlalu ngaruh gimana" sih.

### 4.2.4 Hasil Pembagian Data

- Ratio Split: 70/15/15
- Kenapa val dan test itu 15/85?
  - Biar jumlah data split validasi dan test sama jumlahnya
- Kita ada kasih pos_weight juga biar loss terhadap piksel plastik di kali 18,37.
  - Kenapa rumus-nya rasio piksel latar belakang terhadap piksel plastik?
    - Rumus pos_weight itu negative class / positive class which's samething as above.

### 4.3 Hasil Perbandingan Model

- 17 konfigurasi model dari variasi arsitektur, encoder, dan pretrained weight.
- Semuanya dilatih pake settingan pembagian data yang sama, augmentasi, fungsi loss, dan random seed yang sama
- Total epoch default 40 epoch + early stop + no learning rate scheduler
- learning rate encoder dan decoder tetap pada pretrained raw (3 x 10^-4) karena biar kenceng aja belajarnya
- learning rate encoder pada pretrained imagenet gituan diubah jadi 1x10^-5 karena model udah belajar pada gambar (ga dari scratch) jadi kita set segini biar aman (ga overfit).
- Model terus dievaluasi dengan threshold 0,5
  - Biar balance aja ygy

### 4.3.1 Perbandingan Arsitektur dan Encoder

- Kita bandingin Unet, Segformer, sama DeepLabV3+
- Arsitektur dibandingin di pair sama encoder yang pakai pretrained weight imageNet aja.
  - Kenapa? karena dijelasin lebih lanjut di 4.3.3 dan pemenang dari perbandingan pretrained weight itu si imagenet.
- mIoU yang paling bagus MiT-B3
- mit-B3 punya precision yang tinggi = model semakin jarang menandai area non-plastik sebagai plastik
  - Kenapa MiT-b3 bagus? theoritically, ada self-attention yang bisa bikin model melihat hubungan antar-patch lebih jelas.
  - Parameter-nya juga lebih banyak dari segi encoder dibanding encoder yg lain.
- Unet ternyata lebih sering skip piksel plastik dibuktikan dengan recall yang lebih rendah.

### 4.3.2 Pengaruh Bobot Awal

- pretrained imageNet itu paling bagus dari segi mIoU terhadap keempat model.
- pretrained raw itu modelnya underfit karena 1400 gambar hasil split itu masih kurang buat bikin modelnya belajar.
  - buktinya, train loss masih tinggi pada epoch terakhir.
  - validation loss-nya ga beda jauh sama train loss ngebuktiin bukan overfit
  - MiT-B3 aja udah 13 epoch ga konvergen" which's nunjukin bukan kurang epoch tapi kurang data
- Kenapa ImageNet lebih bagus?
  - ImageNet general
  - Beberapa di antara-nya udah data plastik
  - Kalau yang lain (ADE, AERIAL, DRONE), di pretrain ke dataset khusus mereka
  - Jadinya, pretrain weight-nya lebih fokus ke objek lainnya yang bukan plastik.
  - Pretrained weight khusus plastik gaada

### 4.3.3 Pengaruh Ukuran Encoder

- MiT-B3 anggep setara sama B5 tapi dia lebih murah jadi ini sweet spot gais.
- Dari DeepLabV3, kita pakai ini buat ngebandingin resnet dan ternyata resnet yg lebih deep ga lebih bagus than one of another.
  - Ukuran encoder ga ngaruh
- MiT-B3 ImageNet itu paling bagus jadi kita GO with this one buat hyperparameter tuning pakai Optuna.
- Kenapa MiT-B5 ga dipake, malah pake MiT-B3? Kalau epoch lebih banyak bukannya lebih bagus?
  - dataset kita kecil, sementara ini model with a lot of params
  - computing power kita terbatas

### 4.4 Hasil Pencarian Hyperparameter tuning

### 4.4.1 Hasil Optimisasi

- 25 trial
- paling bagus trial ke 15
- tiap trial 12 epoch biar cepet
- Kenapa TPE?
  - Grid = datengin semua warung, satu per satu, per jalan. Pasti ketemu, tapi capek 💀
  - Random = datengin warung random. Lumayan, tapi ga belajar apa-apa dari warung sebelumnya.
  - TPE = cobain beberapa warung random dulu, terus “eh daerah sini enak-enak”, jadi fokus nyari di sekitar situ 🧠

### 4.4.2 Learning Rate Scheduler

- Our model need high learning rate di awal, and low learning rate di akhir
  - Val Loss ga stabil di awal, di akhir mulai stabil
- Curves ga show overfit karena valloss dan train loss berhasil turun ke titik terrendah.

### 4.5 Evaluasi Model

- Nilai metrik + Subjektif liat hasil prediksi itu cara evaluasi kita.
- Buat metrik kita pakai both IoU dan mIoU

#### TODO:

- Rata-rata luas plastik pada setiap citra adalah 5,46%, dan pada citra yang mengandung plastik nilai mediannya hanya 2,44%. (Cek ada code-nya atau engga -> kalau gaada, bikin script buat EDA bagian ini)
- Transfer Learning tambahin buat bab 2
- JANGAN LUPA BELAJAR BAHASA TECHNICAL CNN LIKE WHAT YOU MEAN BY LOCALITAS
- Nanti reword beberapa bab yang cmn nyebut seolah-olah mIoU yang kita jadiin patokan
- Pindahin ini ke limitasi deh:
  > Selain itu, citra asli yang berukuran 1280 × 720 piksel diubah ukurannya menjadi 512 × 512 piksel, sehingga objek kecil akan menjadi semakin kecil. SegFormer juga menghasilkan prediksi pada seperempat resolusi masukan sebelum diperbesar kembali. Akibatnya, batas dari objek kecil menjadi kurang tajam. Pada objek yang kecil, kesalahan beberapa piksel di batas objek sudah mengambil bagian yang besar dari luas objek tersebut. Hal ini membuat nilai IoU .cepat turun.
  > Ini juga, soal persentase sampah plastik yang kita tampilin itu based on foto. Kalau foto-nya di kecilin, persentase-nya juga tar jadi kecil.
- Why Segmentation? what's our model for tbh?
- Discuss with Alex:
  - Recommendation mending gimana? sekarang di hard code.
    - Bisa pake gemini (Vibe code dulu aja la ya) - HOLD DULU
    - Kemungkinan besar dihilangin
  - Use-case our model and application la

#### to be discussed:

> **Masalah utamanya:** mantau sampah plastik terapung secara manual itu lambat, capek, dan subjektif 😮‍💨

Dari Bab 1 kalian:

- Setiap tahun ±19–23 juta ton plastik masuk ke perairan (UNEP, 2021).
- Jadi dibutuhkan **pemantauan yang berkala dan terus-menerus**.
- Drone (UAV) udah bisa motret sungai atau danau dari atas, tapi fotonya tetap harus **dicek manusia satu per satu**. Kalau fotonya ratusan, ga scalable 💀

**Yang diselesaikan model kita:**

| Kebutuhan user                   | Tanpa model                | Dengan PDIS-Net                           |
| -------------------------------- | -------------------------- | ----------------------------------------- |
| Plastiknya **di mana**?          | Zoom foto satu-satu 👀     | Mask otomatis nunjukin lokasinya          |
| **Seberapa banyak**?             | Kira-kira doang, subjektif | Persentase cakupan yang terukur           |
| **Mana yang dibersihin duluan**? | Feeling 🤷                 | Kategori + rekomendasi (High = prioritas) |
| **Waktu**                        | Lama                       | ±3–5 detik per foto                       |

**Siapa usernya:** tim kebersihan sungai, dinas lingkungan, peneliti, atau komunitas peduli lingkungan yang punya drone.

**Versi satu kalimat buat sidang:**

> _"Model ini membantu pengguna menemukan lokasi dan memperkirakan luas sampah plastik terapung pada citra udara secara otomatis, sehingga pemantauan dapat dilakukan lebih cepat dan konsisten, serta membantu menentukan area yang perlu diprioritaskan untuk dibersihkan."_

⚠️ Batasannya juga harus disebut biar ga overclaim:

- Hasilnya **bukan ukuran kualitas air**. Kategori cuma dibandingin sama dataset FloPWD.
- Model masih bisa **salah di tanaman atau rumput**, dan bisa **kelewat plastik transparan**. Jadi posisinya sebagai **alat bantu** buat manusia, bukan pengganti 100% 🫡

Push-nya masih kena 403. GitHub udah ga ngasih akses ke `williamtheodoruswijaya/PDIS-Net`, jadi commit `f52ddfd` belum masuk.

Biar bisa ke-push, pilih salah satu:

- Reconnect GitHub di https://claude.ai/connect-github
- Pindah ke session baru pakai `PDIS-Net/PDIS-Net`, terus apply patch yang udah aku kirim

Selama itu belum beres, aku ga bisa push dari session ini.

### Notes:

- Kalau di cook pas sidang, tau darimana ini menang gegara hyperparameter bukan karna seeds/randomness?
  - Kita ga pernah nyebut menang/setara. (Jangan fking overclaim)
  - Secara komputasi juga mahal, 17 konfigurasi di retrain pake beda seed. Not that worth it la.
- Scan paper kita kalau ada overclaim soal one model win at another because of hyperparameter
  - Suru CLAUDE ubah" wordingnya
