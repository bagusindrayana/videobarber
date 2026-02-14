# VideoBarber

Aplikasi web untuk memotong video langsung di browser. Dibangun dengan SvelteKit dan FFmpeg WASM.

## Fitur

- **Upload Video** - Unggah file video dari perangkat Anda
- **Trim Video** - Tentukan titik awal dan akhir dengan timeline visual
- **Preview Frame** - Lihat preview frame pada posisi tertentu
- **Proses di Browser** - Tidak perlu server, semua diproses di browser menggunakan FFmpeg WASM
- **PWA Support** - Dapat diinstall sebagai aplikasi di perangkat

## Teknologi

- SvelteKit + TypeScript
- TailwindCSS
- FFmpeg WASM
- Progressive Web App (PWA)

## Development

```bash
# Install dependencies
npm install

# Start development server
npm run dev

# Build for production
npm run build
```

## Cara Penggunaan

1. Buka aplikasi di browser
2. Upload file video Anda
3. Atur titik awal dan akhir pada timeline
4. Klik "Trim Video" untuk memproses
5. Download hasil video yang sudah dipotong
