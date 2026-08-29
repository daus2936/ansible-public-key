# Distribusi SSH Public Key ke Banyak Server (Ansible)

Menambahkan public key ke `~/.ssh/authorized_keys` di semua server, idempoten
(aman dijalankan berulang) dan **tidak menghapus key yang sudah ada**.

## Struktur

```
ansible-kg/
├── ansible.cfg                    # forks, ControlPersist, IdentitiesOnly, host key
├── requirements.yml               # collection ansible.posix
├── quick.yml                      # playbook (1 task, tanpa fact gathering)
├── inventory/
│   ├── hosts.ini                  # daftar server
│   └── group_vars/all.yml         # user, key login, key yang dipasang
└── files/pubkeys/
    ├── romi.pub                # nama file -> baris "#romi" di authorized_keys
    └── daus.pub
```

## Setup di WSL (controller)

Ansible tidak berjalan native di Windows.

```bash
sudo apt update && sudo apt install -y ansible
```

`sshpass` hanya dibutuhkan kalau login ke server masih memakai password (`--ask-pass`).
Kalau sudah pakai private key, tidak perlu.

```bash
ansible-galaxy collection install -r requirements.yml
```

Salin SSH key dari Windows ke WSL — jangan tunjuk `ansible_ssh_private_key_file`
ke `/mnt/c/...` karena permission mount-nya ditolak ssh:

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh && cp /mnt/c/Users/62812/.ssh/id_ed25519 ~/.ssh/ && chmod 600 ~/.ssh/id_ed25519
```

Taruh project di filesystem WSL, bukan di `/mnt/e/` — I/O lewat DrvFs jauh lebih lambat:

```bash
cp -r /mnt/e/claude-code/ansible-kg ~/ansible-kg && cd ~/ansible-kg
```

## Pakai

1. Taruh public key, **satu file per orang**. Nama file menentukan komentarnya:
   ```bash
   cat ~/.ssh/id_ed25519.pub > files/pubkeys/romi.pub
   ```
   Lalu daftarkan di `ssh_key_files` (`inventory/group_vars/all.yml`).

2. Isi daftar server di `inventory/hosts.ini`.

3. Cek koneksi:
   ```bash
   ansible all -m ping
   ```

4. Dry-run, lalu eksekusi:
   ```bash
   ansible-playbook quick.yml --check --diff
   ```
   ```bash
   ansible-playbook quick.yml
   ```

## Format hasil di server

```
#romi
ssh-rsa AAAAB3Nza... ansible@app
#daus
ssh-ed25519 AAAAC3Nza... daus@laptop
```

Baris `#nama` diambil dari nama file `.pub`, bukan dari isi key-nya. Modul
`authorized_key` sendiri tidak bisa menulis baris komentar, jadi playbook memasang key
lebih dulu lalu menyisipkan komentarnya dengan `lineinfile`. Itu sebabnya ada dua task,
bukan satu — tetap tanpa fact gathering, jadi tambahannya hanya satu round trip.

## Dua variabel "user" yang beda peran

| Variabel | Artinya |
|---|---|
| `ansible_user` | user untuk **login SSH** |
| `ssh_key_user` | user yang `authorized_keys`-nya **diubah** |

| Skenario | Konfigurasi |
|---|---|
| Login ubuntu, key untuk ubuntu | `ansible_user: ubuntu` + `ssh_key_user: ubuntu` + `ssh_key_become: false` |
| Login admincitis, key untuk ubuntu | `ssh_key_user: ubuntu` + `ssh_key_become: true` |

Path-nya tidak ditebak dari nama user — modul membaca home directory sebenarnya dari
`/etc/passwd` server.

## Skenario lain

**Kalau server BELUM punya key Anda sama sekali** — hanya untuk kasus ini, dan hanya
sekali. Kalau akses via private key sudah jalan, lewati saja (butuh `sshpass`):
```bash
ansible-playbook quick.yml --ask-pass
```

**Ulangi hanya host yang gagal:**
```bash
ansible-playbook quick.yml --limit @retry/quick.retry
```

**Uji ke sebagian server dulu:**
```bash
ansible-playbook quick.yml --limit web
```

**Cabut key (offboarding):**
```bash
ansible-playbook quick.yml -e ssh_key_state=absent -e ssh_key_files='["files/pubkeys/mantan-staff.pub"]'
```

**Server dengan user berbeda** — bikin `inventory/group_vars/<nama_grup>.yml`:
```yaml
ansible_user: ubuntu
ssh_key_user: ubuntu
```

## Catatan penting

- `exclusive: false` di `quick.yml` — **jangan** diubah ke `true`. Kalau `true`, semua
  key lain di `authorized_keys` terhapus di seluruh server sekaligus.
- **Playbook ini tidak memvalidasi isi file `.pub`.** Apa pun isinya ikut ditulis ke
  `authorized_keys`. Cek semua file sekaligus sebelum run — kalau ada yang error,
  jangan dijalankan dulu:
  ```bash
  for f in files/pubkeys/*.pub; do echo -n "$f: "; ssh-keygen -l -f "$f"; done
  ```
- **Satu key per file `.pub`, tanpa baris `#` di dalamnya.** Komentar `#nama` disisipkan
  oleh playbook berdasarkan nama file, dan pencocokannya memakai isi file sebagai
  patokan posisi. File berisi lebih dari satu key membuat penyisipan komentar meleset.
- `manage_dir: true` — `~/.ssh` (0700) dan `authorized_keys` (0600) dibuat dengan owner
  yang benar. Permission salah = SSH menolak key tanpa pesan yang jelas.
- `IdentitiesOnly=yes` di `ansible.cfg` membuat ssh hanya memakai key yang ditunjuk.
  Tanpa itu, ssh menawarkan semua key di `~/.ssh` satu per satu dan bisa kena
  `Too many authentication failures` — gejalanya gagal acak di sebagian host.
- **Host key**: `StrictHostKeyChecking=accept-new`. Host baru diterima otomatis, tapi
  host yang sudah dikenal dan tiba-tiba ganti key akan ditolak. Jangan set
  `host_key_checking = False`, itu menghilangkan proteksi tersebut.
