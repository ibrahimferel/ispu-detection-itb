# ISPU Jakarta Classification - Datavidia 10.0

Proyek ini merupakan dokumentasi pengerjaan kompetisi time series dan klasifikasi kualitas udara pada Datavidia 10.0 (ITB).

Fokus utamanya adalah memprediksi kategori ISPU harian per stasiun pemantau di DKI Jakarta dengan pendekatan yang tahan terhadap:

- Data multi-sumber dan format yang tidak seragam antar tahun.
- Distribusi label yang tidak seimbang (imbalanced classes).
- Sifat data yang berbasis waktu (risiko data leakage bila split tidak benar).

## 1. Latar Belakang Masalah

Penurunan kualitas udara Jakarta adalah isu kesehatan publik dan lingkungan yang krusial. Kategori polusi ekstrem memang lebih jarang muncul, tetapi dampaknya paling besar. Karena itu, model perlu:

- Tetap akurat pada kelas mayoritas.
- Tetap sensitif pada kelas minoritas yang kritis.
- Stabil saat diterapkan pada horizon prediksi ke depan.

## 2. Tujuan Proyek

- Menyusun pipeline data ISPU 2010-2025 yang konsisten lintas tahun.
- Membangun model klasifikasi kategori ISPU yang robust untuk data temporal.
- Mengurangi dampak imbalance class melalui rekayasa data dan evaluasi metrik yang tepat.
- Menyiapkan skenario prediksi ke depan (future horizon) berbasis kombinasi forecasting + classification.

## 3. Dataset yang Digunakan

Sumber kompetisi:

- https://www.kaggle.com/competitions/penyisihan-datavidia-10/overview

Dataset yang tersedia pada kompetisi mencakup:

- ISPU historis (fitur polutan + target kategori)
- NDVI
- Cuaca harian
- Jumlah penduduk
- Kualitas air sungai
- Libur nasional dan weekend

Catatan implementasi pada notebook saat ini:

- Pipeline utama menggunakan ISPU historis sebagai sinyal inti.
- Informasi kalender (libur, weekend, day name) sudah diintegrasikan.
- NDVI, cuaca, demografi, dan kualitas air sudah dimuat pada tahap awal, namun belum dipakai penuh pada model final dalam notebook ini.

## 4. Ringkasan Solusi

Arsitektur solusi dilakukan dalam dua lapis:

1. Klasifikasi kategori ISPU dari fitur historis dan fitur turunan waktu.
2. Forecasting nilai polutan horizon masa depan, lalu hasilnya dipetakan lagi ke kategori menggunakan classifier final.

Pendekatan ini dipilih agar model tetap bisa menghasilkan submission untuk periode yang tidak punya label ground-truth.

## 5. Alur Pengerjaan di Notebook

### A. Data Cleaning dan Standardisasi Antar Tahun

- Merge data ISPU per tahun (2010-2025).
- Menyamakan nama kolom yang berbeda antar file.
- Normalisasi format tanggal (termasuk format campuran dan serial date Excel).
- Ekstraksi kode stasiun ke format konsisten: DKI1 sampai DKI5.
- Pembersihan nilai anomali dan konversi numerik kolom polutan.

### B. Penanganan Label dan Imbalance

- Mapping label menjadi 3 kelas operasional:
  - BAIK
  - SEDANG
  - TIDAK SEHAT
- Label BERBAHAYA dan SANGAT TIDAK SEHAT digabung ke TIDAK SEHAT.
- Baris TIDAK ADA DATA dibuang dari data latih.
- Undersampling pada kelas dominan untuk memperbaiki keseimbangan distribusi.

### C. Feature Engineering Berbasis Waktu

Fitur yang dibangun:

- Fitur kalender: day, month, year, day_of_week.
- Fitur siklik musiman: sin/cos month dan sin/cos week.
- Fitur lag per stasiun: lag 1, 7, 30 hari untuk tiap polutan utama.
- Fitur rolling statistic per stasiun: mean, max, std pada window 7 dan 30 hari.
- Rasio antar polutan: pm10/so2 dan co/o3.

### D. Validasi Temporal

- Split train-validasi berbasis tanggal (bukan random split):
  - Train: sebelum 2024-01-01
  - Validasi: mulai 2024-01-01
- TimeSeriesSplit digunakan saat proses tuning untuk meminimalkan leakage.

### E. Benchmark Model Klasifikasi

Beberapa model diuji menggunakan metrik weighted F1-score.

Model yang dievaluasi meliputi kombinasi model linear dan boosting (termasuk XGBoost dan CatBoost). Pada hasil benchmark notebook, model terbaik adalah XGBoost dengan weighted F1-score tertinggi sekitar 0.7403.

### F. Hyperparameter Tuning

- Tuning XGBoost menggunakan Optuna.
- Target objective: memaksimalkan macro F1 pada skema TimeSeriesSplit.
- Pembersihan label anomali dilakukan sebelum fitting final model.

### G. Forecasting untuk Horizon Submission

- Forecasting polutan dilakukan dengan Temporal Fusion Transformer (TFT) per lokasi dan per target polutan (pm10, so2, co, o3, no2).
- Horizon prediksi yang dipakai: 2025-09-01 sampai 2025-11-30.
- Hasil forecast diubah menjadi fitur turunan (lag, rolling, kalender) dengan skema yang konsisten terhadap training.
- Classifier XGBoost final digunakan untuk menghasilkan kategori akhir ISPU.
- File submission dihasilkan dalam format id, categori.

## 6. Struktur Repository

```text
ispu-detection-itb/
  datavidia_ispu.ipynb
  README.md
```

## 7. Cara Menjalankan

Karena pipeline ada di notebook, jalankan secara berurutan dari atas ke bawah.

Dependensi utama yang digunakan di notebook:

- pandas
- numpy
- scikit-learn
- xgboost
- catboost
- optuna
- pytorch-lightning
- pytorch-forecasting

Contoh instalasi cepat:

```bash
pip install pandas numpy scikit-learn xgboost catboost optuna pytorch-lightning pytorch-forecasting
```

## 8. Strategi Menanggulangi Permasalahan Kompetisi

Inti strategi yang dipakai pada proyek ini:

1. Rapikan data dulu, baru modeling. Konsistensi skema lintas tahun adalah fondasi utama agar model tidak belajar noise format.

2. Tangani imbalance secara eksplisit. Dilakukan dengan mapping kelas kritis, filtering label invalid, dan balancing pada data latih.

3. Gunakan validasi yang menghormati waktu. Time-based split dan TimeSeriesSplit lebih realistis untuk data polusi harian.

4. Gabungkan forecasting + classification. Forecasting polutan memberi sinyal masa depan, classifier menerjemahkan ke kategori ISPU.

5. Optimasi metrik yang relevan. Weighted F1 dipilih agar performa tidak bias kelas mayoritas semata.

## 9. Pengembangan Lanjutan

Beberapa peningkatan yang paling potensial:

- Integrasi penuh fitur cuaca, NDVI, demografi, dan kualitas air ke pipeline final.
- Cost-sensitive learning atau class-weighting untuk meningkatkan sensitivitas kelas kritis.
- Ensemble model (stacking/blending) antara model tree-based dan temporal deep learning.
- Interpretabilitas model (feature importance, SHAP) untuk insight kebijakan lingkungan.

## 10. Penutup

Proyek ini menunjukkan bahwa tantangan kualitas udara dapat ditangani dengan pendekatan data science yang sistematis: data cleaning yang disiplin, feature engineering temporal yang tepat, validasi anti-leakage, dan pemodelan hibrida untuk prediksi masa depan.

Jika Anda ingin, README ini bisa saya lanjutkan ke versi portofolio kompetisi (dengan section leaderboard, error analysis, dan ablation study) agar lebih kuat untuk presentasi final atau GitHub showcase.
