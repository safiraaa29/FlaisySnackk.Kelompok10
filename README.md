# FlaisySnackk.Kelompok10
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Flaisy Snack - Basreng & Makaroni Pedas</title>
    <!-- Load Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- Load Lucide Icons for aesthetic icons -->
    <script src="https://unpkg.com/lucide@latest"></script>
    <style>
        /*
         * Konfigurasi Tailwind CSS untuk Font 'Inter' dan warna kustom
         * Sesuai permintaan untuk font estetik, kami akan menggunakan font yang lebih lembut.
         * Kami akan menggunakan 'Inter' sebagai default, dan 'Caveat' sebagai font lucu/dekoratif
         * untuk footer/slogan (menggantikan Joypops/Bubble Pop karena keterbatasan font di sandbox).
         */
        @import url('https://fonts.googleapis.com/css2?family=Caveat:wght@400;700&family=Inter:wght@100..900&display=swap');

        :root {
            --primary-color: #F87171; /* Merah Muda/Salmon, lebih soft dari merah cabai */
            --secondary-color: #FFDE59; /* Kuning Cerah/Mustard, untuk kontras */
            --background-soft: #FFF7ED; /* Creamy White/Sangat Soft */
            --text-dark: #374151; /* Dark Gray */
        }

        /* Konfigurasi untuk menimpa font default Tailwind */
        tailwind-config {
            theme: {
                extend: {
                    colors: {
                        'primary': 'var(--primary-color)',
                        'secondary': 'var(--secondary-color)',
                        'soft-bg': 'var(--background-soft)',
                    },
                    fontFamily: {
                        sans: ['Inter', 'sans-serif'],
                        display: ['Caveat', 'cursive'], /* Font estetis/imut untuk slogan */
                    }
                }
            }
        }

        /* --- Animasi Cabai Berjatuhan (Background Animation) --- */
        .chili-container {
            position: fixed;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            pointer-events: none;
            overflow: hidden;
            z-index: 0;
        }

        .chili-flake {
            position: absolute;
            color: var(--primary-color);
            font-size: 1.2rem;
            animation: fall linear infinite;
        }

        @keyframes fall {
            0% {
                transform: translateY(-10vh) translateX(0) rotate(0deg);
                opacity: 1;
            }
            100% {
                transform: translateY(100vh) translateX(20px) rotate(360deg);
                opacity: 0.5;
            }
        }

        /* --- Custom Scrollbar (Aesthetic) --- */
        ::-webkit-scrollbar {
            width: 8px;
        }

        ::-webkit-scrollbar-track {
            background: var(--background-soft);
        }

        ::-webkit-scrollbar-thumb {
            background: var(--primary-color);
            border-radius: 4px;
        }

        /* Style untuk modal custom, menggantikan alert() */
        .modal {
            transition: opacity 0.3s ease, transform 0.3s ease;
        }
        .modal.hidden {
            opacity: 0;
            pointer-events: none;
            transform: scale(0.95);
        }
        .modal-content {
            animation: bounceIn 0.5s ease-out;
        }
        @keyframes bounceIn {
            0% { transform: scale(0.5); opacity: 0; }
            100% { transform: scale(1); opacity: 1; }
        }
    </style>
    <!-- Inisialisasi Firebase (WAJIB menggunakan variabel global) -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, collection, query, onSnapshot, addDoc } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";
        import { setLogLevel } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // Setel tingkat log untuk debugging
        setLogLevel('Debug');

        // Variabel global dari Canvas
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        const firebaseConfig = JSON.parse(typeof __firebase_config !== 'undefined' ? __firebase_config : '{}');
        const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

        let db;
        let auth;
        let userId = null;

        // Fungsi inisialisasi dan autentikasi
        async function initFirebase() {
            try {
                const app = initializeApp(firebaseConfig);
                db = getFirestore(app);
                auth = getAuth(app);
                window.db = db; // Sediakan akses global untuk fungsi di script utama

                if (initialAuthToken) {
                    await signInWithCustomToken(auth, initialAuthToken);
                } else {
                    await signInAnonymously(auth);
                }

                userId = auth.currentUser?.uid || crypto.randomUUID();
                window.userId = userId; // Sediakan akses global untuk fungsi di script utama
                console.log("Firebase & Auth siap. User ID:", userId);

                // Setelah auth siap, muat data komentar
                loadComments();
            } catch (error) {
                console.error("Gagal menginisialisasi Firebase atau Otentikasi:", error);
                document.getElementById('status-message').textContent = "Kesalahan: Gagal menghubungkan ke database. Silakan coba muat ulang.";
            }
        }

        // --- Operasi Firestore ---
        const PUBLIC_COLLECTION_PATH = `artifacts/${appId}/public/data/comments`;

        function loadComments() {
            if (!db) {
                console.error("Database belum diinisialisasi.");
                return;
            }

            const commentsRef = collection(db, PUBLIC_COLLECTION_PATH);
            const q = query(commentsRef);

            // Listener real-time untuk komentar
            onSnapshot(q, (snapshot) => {
                const comments = [];
                snapshot.forEach((doc) => {
                    // Hanya mengambil 5 komentar terbaru (untuk tampilan awal)
                    comments.push({ id: doc.id, ...doc.data() });
                });

                // Urutkan berdasarkan tanggal (misalnya, jika ada field timestamp) atau ID (sementara)
                // Di sini kita hanya menampilkan yang ada
                const sortedComments = comments.reverse(); // Urutan terbaru di atas

                renderComments(sortedComments);
                document.getElementById('status-message').textContent = ""; // Hapus pesan status
            }, (error) => {
                console.error("Kesalahan saat memuat komentar:", error);
                document.getElementById('status-message').textContent = "Kesalahan saat memuat komentar.";
            });
        }

        function renderComments(comments) {
            const container = document.getElementById('comments-list');
            container.innerHTML = ''; // Kosongkan container

            if (comments.length === 0) {
                container.innerHTML = '<p class="text-gray-500 italic text-center py-4">Belum ada komentar. Jadilah yang pertama!</p>';
                return;
            }

            comments.slice(0, 5).forEach(comment => { // Tampilkan maks 5 komentar
                const commentElement = document.createElement('div');
                commentElement.className = 'bg-white p-4 rounded-xl shadow-lg border border-gray-100 mb-4 transition transform hover:shadow-xl';
                commentElement.innerHTML = `
                    <div class="flex items-center mb-2">
                        <span class="text-xl text-primary mr-3" data-lucide="star"></span>
                        <p class="font-semibold text-gray-800">${comment.name || 'Anonim'}</p>
                    </div>
                    <p class="text-sm text-gray-700 leading-relaxed">${comment.comment}</p>
                `;
                container.appendChild(commentElement);
                // Inisialisasi ikon Lucide
                lucide.createIcons();
            });
        }

        window.submitComment = async () => {
            const nameInput = document.getElementById('comment-name');
            const commentInput = document.getElementById('comment-text');

            const name = nameInput.value.trim();
            const comment = commentInput.value.trim();

            if (!name || !comment) {
                showCustomModal("Input Tidak Lengkap", "Nama dan Komentar wajib diisi.", "error");
                return;
            }

            if (!db) {
                showCustomModal("Error", "Koneksi database gagal. Coba muat ulang halaman.", "error");
                return;
            }

            try {
                await addDoc(collection(db, PUBLIC_COLLECTION_PATH), {
                    name: name,
                    comment: comment,
                    timestamp: new Date().toISOString(),
                    userId: userId,
                });
                showCustomModal("Berhasil!", "Terima kasih atas komentarnya!", "success");
                nameInput.value = '';
                commentInput.value = '';
            } catch (e) {
                console.error("Error menambahkan dokumen:", e);
                showCustomModal("Error", "Gagal mengirim komentar: " + e.message, "error");
            }
        };

        // Memanggil inisialisasi
        initFirebase();
    </script>
</head>
<body class="bg-soft-bg font-sans text-gray-800 antialiased relative">

    <!-- Kontainer Animasi Cabai -->
    <div class="chili-container" id="chili-flakes"></div>

    <!-- Modal Kustom (Menggantikan alert) -->
    <div id="custom-modal" class="modal hidden fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center p-4 z-50">
        <div class="modal-content bg-white p-6 rounded-xl shadow-2xl max-w-sm w-full">
            <h3 id="modal-title" class="text-xl font-bold mb-3 text-primary">Judul</h3>
            <p id="modal-message" class="text-gray-700 mb-4"></p>
            <div class="flex justify-end">
                <button onclick="closeCustomModal()" class="bg-primary text-white font-semibold py-2 px-4 rounded-lg hover:bg-red-600 transition duration-200 shadow-md">
                    Tutup
                </button>
            </div>
        </div>
    </div>

    <!-- --- Bagian Utama Konten (Z-index 10 untuk di atas animasi) --- -->
    <div class="relative z-10 min-h-screen flex flex-col">

        <!-- Header/Navbar (Elegan dan Fixed) -->
        <header class="bg-white/90 backdrop-blur-sm shadow-md sticky top-0 z-40">
            <div class="container mx-auto px-4 py-3 flex justify-between items-center">
                <h1 class="text-3xl font-bold text-primary tracking-wider">Flaisy Snack</h1>
                <nav class="hidden md:flex space-x-6 text-lg font-medium">
                    <a href="#home" class="text-gray-600 hover:text-primary transition duration-150">Beranda</a>
                    <a href="#products" class="text-gray-600 hover:text-primary transition duration-150">Produk</a>
                    <a href="#order" class="text-gray-600 hover:text-primary transition duration-150">Pesan</a>
                    <a href="#reviews" class="text-gray-600 hover:text-primary transition duration-150">Ulasan</a>
                </nav>
                <a href="#order" class="bg-primary text-white px-4 py-2 rounded-lg font-semibold shadow-lg hover:shadow-xl transition duration-300 transform hover:scale-105 hidden md:block">
                    Pesan Sekarang!
                </a>
            </div>
        </header>

        <!-- Hero Section -->
        <main class="flex-grow">
            <section id="home" class="py-16 md:py-24 bg-soft-bg relative overflow-hidden">
                <div class="container mx-auto px-4 text-center">
                    <!-- Headline Utama -->
                    <h2 class="text-5xl md:text-7xl font-extrabold text-primary mb-4 leading-tight">
                        Pedas Nikmat yang <span class="block md:inline">bikin ketagihan!</span>
                    </h2>
                    <!-- Tagline -->
                    <p class="text-xl md:text-3xl font-medium text-gray-700 mb-10 italic">
                        Cemilan renyah, pedas bikin nangih!
                    </p>

                    <a href="#order" class="inline-block bg-secondary text-gray-900 text-xl font-bold py-4 px-10 rounded-full shadow-2xl border-4 border-primary/50 hover:bg-secondary/80 transition duration-300 transform hover:scale-110">
                        Coba Sekarang Juga!
                    </a>
                </div>
            </section>

            <!-- Product Section -->
            <section id="products" class="py-16 bg-white shadow-inner">
                <div class="container mx-auto px-4">
                    <h2 class="text-4xl font-bold text-center text-primary mb-12">Pilihan Pedas Terbaik Kami</h2>

                    <div class="grid md:grid-cols-2 gap-10 max-w-4xl mx-auto">
                        <!-- Produk 1: Basreng Pedas -->
                        <div class="product-card bg-soft-bg p-6 rounded-2xl shadow-xl border-t-4 border-primary transform hover:scale-[1.02] transition duration-300">
                            <img src="https://o-cdf.oramiland.com/unsafe/cnc-magazine.oramiland.com/parenting/original_images/Screen_Shot_2024-05-24_at_18.47.39.png"
                                 alt="Basreng Pedas Flaisy Snack"
                                 class="w-full h-64 object-cover rounded-lg mb-4 shadow-md transition duration-300 hover:opacity-90">
                            <h3 class="text-3xl font-bold text-gray-900 mb-2 flex justify-between items-center">
                                Basreng Pedas
                                <span class="text-xl text-primary font-extrabold bg-secondary px-3 py-1 rounded-full shadow-inner">Rp 5.000</span>
                            </h3>
                            <p class="text-gray-600 mb-4 italic">Bakso goreng renyah kriuk-kriuk dengan bumbu cabai rahasia.</p>
                            <p class="text-base text-gray-700">
                                Dibuat dari bakso ikan pilihan yang digoreng hingga garing sempurna, Basreng Pedas kami dibalut dengan bumbu pedas manis yang meledak di mulut. Teksturnya yang renyah di luar dan sedikit kenyal di dalam akan membuat Anda tidak bisa berhenti mengunyah! Wajib dicoba untuk pecinta gurih pedas!
                            </p>
                        </div>

                        <!-- Produk 2: Makaroni Pedas -->
                        <div class="product-card bg-soft-bg p-6 rounded-2xl shadow-xl border-t-4 border-primary transform hover:scale-[1.02] transition duration-300">
                            <img src="https://down-id.img.susercontent.com/file/85682004a253e70891e328dd4e103c46"
                                 alt="Makaroni Pedas Flaisy Snack"
                                 class="w-full h-64 object-cover rounded-lg mb-4 shadow-md transition duration-300 hover:opacity-90">
                            <h3 class="text-3xl font-bold text-gray-900 mb-2 flex justify-between items-center">
                                Makaroni Pedas
                                <span class="text-xl text-primary font-extrabold bg-secondary px-3 py-1 rounded-full shadow-inner">Rp 5.000</span>
                            </h3>
                            <p class="text-gray-600 mb-4 italic">Makaroni bantat krispi dengan bubuk cabai super premium.</p>
                            <p class="text-base text-gray-700">
                                Makaroni Pedas kami digoreng dengan metode khusus sehingga menghasilkan tekstur 'bantat' yang super renyah dan tidak keras. Ditaburi bubuk cabai yang gurih dan pedas nendang, cemilan ini sempurna untuk menemani waktu santai atau saat kerja kelompok. Sensasi pedasnya bikin ketagihan dari gigitan pertama!
                            </p>
                        </div>
                    </div>
                </div>
            </section>

            <!-- Order Section (Formulir Pemesanan) -->
            <section id="order" class="py-16 md:py-20 bg-primary/10">
                <div class="container mx-auto px-4 max-w-3xl">
                    <h2 class="text-4xl font-bold text-center text-primary mb-10">Pesan Sekarang & Rasakan Sensasinya!</h2>

                    <form id="orderForm" class="bg-white p-6 md:p-8 rounded-2xl shadow-2xl border-b-8 border-primary">
                        <!-- Data Pembeli -->
                        <h3 class="text-2xl font-semibold mb-4 text-gray-700 border-b pb-2">Informasi Pengiriman</h3>
                        <div class="grid grid-cols-1 md:grid-cols-2 gap-4 mb-6">
                            <div>
                                <label for="name" class="block text-sm font-medium text-gray-600 mb-1">Nama Lengkap</label>
                                <input type="text" id="name" required class="w-full p-3 border border-gray-300 rounded-lg focus:ring-primary focus:border-primary transition duration-150">
                            </div>
                            <div>
                                <label for="phone" class="block text-sm font-medium text-gray-600 mb-1">Nomor Telepon (WA Aktif)</label>
                                <input type="tel" id="phone" required placeholder="Contoh: 089512345678" class="w-full p-3 border border-gray-300 rounded-lg focus:ring-primary focus:border-primary transition duration-150">
                            </div>
                        </div>
                        <div class="mb-8">
                            <label for="address" class="block text-sm font-medium text-gray-600 mb-1">Alamat Lengkap (Jalan, Nomor Rumah, Kecamatan, Kota)</label>
                            <textarea id="address" rows="3" required class="w-full p-3 border border-gray-300 rounded-lg focus:ring-primary focus:border-primary transition duration-150"></textarea>
                        </div>

                        <!-- Daftar Pesanan -->
                        <h3 class="text-2xl font-semibold mb-4 text-gray-700 border-b pb-2">Detail Pesanan</h3>
                        <div class="grid grid-cols-3 gap-4 items-center mb-6 p-4 bg-soft-bg rounded-lg">
                            <p class="font-bold text-gray-800 col-span-2">Produk</p>
                            <p class="font-bold text-gray-800 text-right">Jumlah (Pak)</p>
                            <!-- Basreng Pedas -->
                            <p class="col-span-2 text-lg text-gray-700">Basreng Pedas (Rp 5.000/pack)</p>
                            <input type="number" id="qty-basreng" value="0" min="0" data-price="5000" onchange="calculateTotal()" class="w-full p-2 border border-gray-300 rounded-lg text-right focus:ring-primary focus:border-primary transition duration-150">

                            <!-- Makaroni Pedas -->
                            <p class="col-span-2 text-lg text-gray-700">Makaroni Pedas (Rp 5.000/pack)</p>
                            <input type="number" id="qty-makaroni" value="0" min="0" data-price="5000" onchange="calculateTotal()" class="w-full p-2 border border-gray-300 rounded-lg text-right focus:ring-primary focus:border-primary transition duration-150">
                        </div>

                        <!-- Total Biaya -->
                        <div class="flex justify-between items-center bg-primary/20 p-4 rounded-lg mb-8">
                            <span class="text-xl font-bold text-gray-800">TOTAL BIAYA:</span>
                            <span id="total-cost" class="text-2xl font-
