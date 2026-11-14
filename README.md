<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Deteksi Gambar Hewan dengan Kamera</title>
    <!-- Memuat TensorFlow.js dan Model MobileNet dari CDN -->
    <script src="https://jsdelivr.net"></script>
    <script src="https://jsdelivr.net"></script>
    <style>
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background-color: #f0f4f8;
            color: #333;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            min-height: 100vh;
            margin: 0;
            padding: 20px;
        }
        h1 {
            color: #1e3a8a;
            margin-bottom: 10px;
            text-align: center;
        }
        p.subtitle {
            color: #64748b;
            margin-bottom: 20px;
            text-align: center;
        }
        .camera-container {
            position: relative;
            width: 100%;
            max-width: 640px;
            background: #000;
            border-radius: 12px;
            overflow: hidden;
            box-shadow: 0 10px 25px rgba(0,0,0,0.15);
        }
        video {
            width: 100%;
            height: auto;
            display: block;
        }
        .result-box {
            margin-top: 20px;
            padding: 20px;
            background: #ffffff;
            border-radius: 12px;
            box-shadow: 0 4px 15px rgba(0,0,0,0.08);
            width: 100%;
            max-width: 600px;
            text-align: center;
        }
        #status {
            font-size: 16px;
            color: #d97706;
            font-weight: bold;
        }
        #prediction {
            font-size: 24px;
            font-weight: bold;
            color: #059669;
            margin: 10px 0 5px 0;
        }
        #confidence {
            font-size: 16px;
            color: #475569;
        }
    </style>
</head>
<body>

    <h1>Pendeteksi Hewan Real-Time</h1>
    <p class="subtitle">Arahkan gambar atau objek hewan ke kamera laptop Anda</p>

    <div class="camera-container">
        <video id="webcam" autoplay playsinline muted></video>
    </div>

    <div class="result-box">
        <div id="status">Memuat model kecerdasan buatan (AI)...</div>
        <div id="prediction">-</div>
        <div id="confidence"></div>
    </div>

    <script>
        const video = document.getElementById('webcam');
        const statusText = document.getElementById('status');
        const predictionText = document.getElementById('prediction');
        const confidenceText = document.getElementById('confidence');

        let model;

        // 1. Fungsi untuk mengaktifkan kamera laptop
        async function setupCamera() {
            try {
                const stream = await navigator.mediaDevices.getUserMedia({
                    video: { facingMode: 'environment' },
                    audio: false
                });
                video.srcObject = stream;
                return new Promise((resolve) => {
                    video.onloadedmetadata = () => {
                        resolve(video);
                    };
                });
            } catch (error) {
                statusText.innerText = "Gagal mengakses kamera. Pastikan izin kamera diberikan.";
                statusText.style.color = "#dc2626";
                console.error("Error kamera:", error);
            }
        }

        // 2. Fungsi untuk mendeteksi objek secara terus-menerus
        async function detectFrame() {
            if (model && video.readyState === 4) {
                // Melakukan klasifikasi gambar dari video
                const predictions = await model.classify(video);

                if (predictions && predictions.length > 0) {
                    const topResult = predictions[0];
                    const probability = (topResult.probability * 100).toFixed(1);

                    statusText.innerText = "Kamera Aktif - Mendeteksi:";
                    statusText.style.color = "#2563eb";
                    predictionText.innerText = topResult.className.toUpperCase();
                    confidenceText.innerText = `Tingkat Akurasi: ${probability}%`;
                }
            }
            // Mengulangi deteksi pada frame berikutnya
            requestAnimationFrame(detectFrame);
        }

        // 3. Fungsi utama untuk menjalankan aplikasi
        async function init() {
            await setupCamera();
            video.play();

            // Memuat model MobileNet
            model = await mobilenet.load();
            
            // Mulai proses deteksi
            detectFrame();
        }

        // Jalankan aplikasi saat halaman dimuat
        init();
    </script>

</body>
</html>
