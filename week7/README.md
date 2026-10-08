# Week 7: YOLOv8 + ROS 2 + Gazebo SAUVC

Panduan onboarding Ubuntu untuk simulasi BlueROV2 dan kamera YOLO. Python untuk latihan menggunakan `week7/.venv`, bukan Conda `(base)`.

Jalankan perintah repository dari **root repository**, yaitu folder yang berisi `week7/`, `week6/`, dan README utama.

## Jalur yang Direkomendasikan

Gunakan **Ubuntu 24.04 LTS + ROS 2 Jazzy + Gazebo Harmonic**. Ini pasangan resmi ROS/Gazebo dan jalur rekomendasi untuk praktik ini. ROS dan Gazebo dipasang sebagai paket sistem melalui `apt`; paket YOLO Python dipasang terpisah di `week7/.venv`.

| Ubuntu | ROS 2 | Gazebo yang dipasangkan | Catatan |
|---|---|---|---|
| 24.04 LTS | Jazzy | Harmonic | Jalur rekomendasi untuk praktik ini |
| 22.04 LTS | Humble | Fortress | Pasangan resmi; world proyek belum diuji pada kombinasi ini |
| 26.04 LTS | Lyrical | Jetty / Sim 10 | Ikuti dokumentasi Lyrical dan pastikan paket tersedia untuk rilis Ubuntu tersebut |

Jangan mencampur distro ROS atau versi Gazebo. Referensi resmi: [ROS 2 Jazzy](https://docs.ros.org/en/jazzy/Installation/Ubuntu-Install-Debs.html), [Gazebo Harmonic](https://gazebosim.org/docs/harmonic/install/), dan [kombinasi ROS-Gazebo](https://gazebosim.org/docs/latest/ros_installation/).

> **Penting:** jangan jalankan `week7/setup_week7.sh` untuk praktik ini. Skrip lama tersebut memasang Miniconda/Conda, sedangkan panduan ini menggunakan `.venv`.

## 1. Prasyarat

- Ubuntu Desktop 24.04 LTS 64-bit dengan koneksi internet.
- Disarankan ruang disk kosong minimal 8 GB untuk Python, PyTorch CPU, dan dependensi YOLO.
- Repository sudah diunduh/di-clone; pastikan `week7/best.pt` tersedia.

Periksa versi Ubuntu dan ruang disk:

```bash
lsb_release -a
df -h /
```

Jika prompt terminal menampilkan `(base)`, keluar dari Conda. Ulangi perintah sampai `(base)` hilang:

```bash
conda deactivate
```

Conda tidak diperlukan setelah langkah ini.

## 2. Instal ROS 2 Jazzy dan Gazebo Harmonic

Tambahkan repository resmi ROS 2. Jalankan blok ini di Ubuntu 24.04:

```bash
sudo apt update
sudo apt install -y software-properties-common curl
sudo add-apt-repository -y universe
export UBUNTU_CODENAME=$(. /etc/os-release && echo "$UBUNTU_CODENAME")
export ROS_APT_SOURCE_VERSION=$(curl -s https://api.github.com/repos/ros-infrastructure/ros-apt-source/releases/latest | grep -F "tag_name" | awk -F\" '{print $4}')
curl -L -o /tmp/ros2-apt-source.deb "https://github.com/ros-infrastructure/ros-apt-source/releases/download/${ROS_APT_SOURCE_VERSION}/ros2-apt-source_${ROS_APT_SOURCE_VERSION}.${UBUNTU_CODENAME}_all.deb"
sudo dpkg -i /tmp/ros2-apt-source.deb
sudo apt update
```

Pasang ROS Desktop, ROS-Gazebo bridge, dan viewer gambar:

```bash
sudo apt install -y ros-jazzy-desktop ros-jazzy-ros-gz ros-jazzy-rqt-image-view
sudo apt install -y python3-venv python3-pip python3-dev
```

`ros-jazzy-ros-gz` memasang bridge serta paket Gazebo vendor yang cocok dengan Jazzy, yaitu Gazebo Harmonic. Jalur ini tidak memerlukan repository Gazebo terpisah. Untuk distro lain, ikuti [panduan resmi pasangan ROS-Gazebo](https://gazebosim.org/docs/latest/ros_installation/) dan jangan memasang bridge dari distro yang berbeda.

Verifikasi ROS dan Gazebo:

```bash
source /opt/ros/jazzy/setup.bash
ros2 --help
ros2 pkg prefix ros_gz_bridge
gz sim --version
```

Jalankan perintah `source` di setiap terminal kerja; panduan ini tidak menambahkan ROS otomatis ke `.bashrc`.

## 3. Buat Python `.venv`

ROS memasang binding `rclpy` untuk Python sistem. Buat venv menggunakan `/usr/bin/python3` dan opsi `--system-site-packages`, agar venv bisa menggunakan binding ROS sistem. Jangan membuat venv dari Python Conda.

```bash
/usr/bin/python3 -m venv --system-site-packages week7/.venv
source /opt/ros/jazzy/setup.bash
source week7/.venv/bin/activate
python -m pip install --upgrade pip
```

Prompt seharusnya menampilkan `(.venv)`, bukan `(base)`. Pasang PyTorch **CPU-only** terlebih dahulu agar pip tidak mengunduh paket CUDA berukuran besar:

```bash
python -m pip install --no-cache-dir torch torchvision --index-url https://download.pytorch.org/whl/cpu
```

Pasang dependensi Python proyek:

```bash
python -m pip install --no-cache-dir -r week7/requirements.txt
```

Jika pip gagal karena disk penuh, jalankan `python -m pip cache purge`, periksa ruang dengan `df -h /`, lalu ulangi instalasi. Jangan menghapus folder repository atau bobot `best.pt`.

Verifikasi ROS dan YOLO terlihat dari interpreter `.venv` yang sama:

```bash
source /opt/ros/jazzy/setup.bash
source week7/.venv/bin/activate
python -c "import rclpy; print('ROS OK')"
python -c "import torch; from ultralytics import YOLO; print('YOLO OK; torch', torch.__version__)"
```

Jalankan tes model sebelum membuka simulasi:

```bash
python week7/test_yolo_inference.py
```

Tes harus menampilkan `Model loaded successfully`. Gambar hasil disimpan di `week7/autonomous_detection_test.jpg`.

## 4. Jalankan Simulasi dengan Tiga Terminal

Buka tiga terminal dari root repository. Jangan aktifkan Conda. Jalankan `source /opt/ros/jazzy/setup.bash` pada setiap terminal; aktifkan `.venv` hanya pada Terminal 3.

### Terminal 1: Gazebo

```bash
source /opt/ros/jazzy/setup.bash
export GZ_SIM_RESOURCE_PATH="$PWD/week7/sauvc26-world/models:${GZ_SIM_RESOURCE_PATH:-}"
env -u GTK_PATH -u GTK_MODULES gz sim -v 3 -r week7/sauvc26-world/worlds/sauvc_qualification.world
```

Tunggu sampai kolam, gate, ROV, dan GUI selesai dimuat. Biarkan terminal tetap berjalan.

### Terminal 2: Bridge ROS-Gazebo

```bash
source /opt/ros/jazzy/setup.bash
ros2 run ros_gz_bridge parameter_bridge --ros-args \
  -p config_file:="$PWD/week7/sauvc26-world/params/autonomous_bridge.yaml"
```

Bridge meneruskan `/front_camera` dan `/overhead_camera` ke ROS, serta perintah thruster dan gambar anotasi ke Gazebo. Biarkan terminal tetap berjalan.

### Terminal 3: Controller dan ROV

```bash
source /opt/ros/jazzy/setup.bash
source week7/.venv/bin/activate
python week7/sauvc26-world/scripts/autonomous_gate_yolo.py
```

Pilih mode saat diminta:

- `1`: mode AI; deteksi YOLO mengendalikan ROV.
- `2`: mode manual; tombol mengendalikan ROV, YOLO tetap membuat anotasi kamera.

Untuk praktik kendali ROV, pilih `2`. Fokuskan Terminal 3 untuk tombol berikut:

| Tombol | Perintah |
|---|---|
| `W` / `S` | Maju / mundur |
| `A` / `D` | Belok kiri / kanan |
| `Q` / `E` | Naik / turun |
| `X` | Berhenti |
| `I` | Pindah ke mode AI |
| `R` | Pindah ke mode manual |

Gerakan manual berupa pulsa singkat. Fokus harus tetap di Terminal 3 ketika menekan tombol.

## 5. Kamera dan Bounding Box

Pada panel **Image Display** Gazebo, pilih topik yang sesuai:

| Tampilan | Topik | Keterangan |
|---|---|---|
| POV ROV dengan bounding box YOLO | `/yolo/annotated_image` | Prediksi kamera depan; tersedia pada mode AI dan manual |
| Kamera depan mentah | `/front_camera` | Gambar POV tanpa kotak |
| Overhead dengan penanda gate | `/yolo/overhead_annotated_image` | Posisi gate dari geometri simulasi, bukan prediksi YOLO |
| Kamera overhead mentah | `/overhead_camera` | Gambar kamera atas tanpa anotasi |

Bounding box YOLO hanya ada pada `/yolo/annotated_image`. Jika ROV tidak menghadap gate, kotak mungkin tidak tampak. Log `[ROV POV] detections=...` menunjukkan jumlah deteksi pada frame, tetapi kotak dilihat melalui panel Image Display. Jika dropdown tidak menampilkan topik anotasi, pastikan Terminal 2 dan 3 aktif, tekan tombol refresh oranye, lalu pilih `/yolo/annotated_image` (bukan `/front_camera`).

## 6. Verifikasi dan Troubleshooting

Di terminal tambahan:

```bash
source /opt/ros/jazzy/setup.bash
source week7/.venv/bin/activate
ros2 node list
ros2 topic list -t
ros2 topic hz /front_camera
ros2 topic hz /yolo/annotated_image
```

Periksa koneksi kamera dan anotasi:

```bash
ros2 topic info /front_camera -v
ros2 topic info /yolo/annotated_image -v
```

| Gejala | Langkah pemeriksaan |
|---|---|
| `ros2: command not found` | Jalankan `source /opt/ros/jazzy/setup.bash` dan pastikan paket ROS terpasang. |
| Prompt menampilkan `(base)` | Jalankan `conda deactivate` sampai `(base)` hilang. Conda tidak dipakai untuk latihan ini. |
| Prompt menampilkan `(.venv)` tetapi `bin/python` tidak ditemukan | Venv lama sudah dihapus/tidak lengkap. Buat ulang dengan perintah di Bagian 3. |
| `No module named rclpy` | Venv harus dibuat dari `/usr/bin/python3` dengan `--system-site-packages`; source ROS sebelum menjalankan node. |
| `No module named ultralytics`, `torch`, atau `cv2` | Aktifkan `week7/.venv`, instal PyTorch CPU, lalu instal `week7/requirements.txt`. |
| Disk penuh saat instalasi pip | Jalankan `python -m pip cache purge`, periksa `df -h /`, lalu ulangi perintah instalasi dengan `--no-cache-dir`. |
| Topik kamera tidak ada | Pastikan Gazebo berjalan dan Terminal 2 memakai `autonomous_bridge.yaml`. |
| `detections=0` | Model tidak menemukan gate pada frame itu. Arahkan kamera POV ROV ke gate dan periksa `/yolo/annotated_image`. |
| Image Display abu-abu | Pastikan bridge dan controller berjalan, cek `ros2 topic hz /yolo/annotated_image`, lalu refresh panel dan pilih topik anotasi yang benar. |
| Gazebo GUI tidak terbuka dari VS Code | Coba jalankan perintah Terminal 1 dari terminal Ubuntu biasa. |

## 7. Menghentikan Simulasi

Tekan `Ctrl+C` pada Terminal 3, lalu Terminal 2. Tutup Gazebo dari GUI atau hentikan Terminal 1 dengan `Ctrl+C`. Saat dihentikan dengan normal, controller mengirim perintah thruster nol.

## File Penting

```text
week7/
├── .venv/                                  # Python environment lokal; jangan commit
├── best.pt                                 # Bobot YOLO custom
├── requirements.txt                        # Dependensi Python inferensi
├── test_yolo_inference.py                  # Tes model offline
└── sauvc26-world/
    ├── params/autonomous_bridge.yaml       # Konfigurasi bridge ROS-Gazebo
    ├── scripts/autonomous_gate_yolo.py     # Controller AI/manual dan anotasi kamera
    ├── scripts/yolo_worker.py              # Worker inferensi YOLO
    └── worlds/sauvc_qualification.world    # Arena dan panel kamera
```
