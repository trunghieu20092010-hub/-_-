<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Chamber of Clouds</title>
    <style>
        body, html { 
            margin: 0; 
            padding: 0; 
            width: 100%; 
            height: 100%; 
            background: linear-gradient(to bottom, #010a15, #0a2540, #1b4965); /* Gradient nền bầu trời đêm chuyển bình minh */
            overflow: hidden; 
            font-family: 'Segoe UI', sans-serif; 
            touch-action: none;
        }
        #canvas-container { width: 100%; height: 100%; position: absolute; top:0; left:0; z-index: 1; }
        
        /* Hiệu ứng sương mù/khói nhẹ phủ trên bề mặt màn hình */
        #vignette {
            position: absolute; width: 100%; height: 100%; top:0; left:0;
            background: radial-gradient(circle, transparent 40%, rgba(1, 10, 21, 0.7) 100%);
            z-index: 2; pointer-events: none;
        }
    </style>
</head>
<body>

    <div id="canvas-container"></div>
    <div id="vignette"></div>

    <script type="importmap">
        { "imports": { "three": "https://unpkg.com/three@0.160.0/build/three.module.js" } }
    </script>

    <script type="module">
        import * as THREE from 'three';

        // 1. Khởi tạo Không gian và Camera góc rộng
        const scene = new THREE.Scene();
        scene.fog = new THREE.FogExp2(0x0a2540, 0.015); // Sương mù môi trường tạo độ sâu trường ảnh

        const camera = new THREE.PerspectiveCamera(65, window.innerWidth / window.innerHeight, 0.1, 1000);
        const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2)); // Tối ưu nét chữ/hình cho điện thoại xịn
        renderer.setSize(window.innerWidth, window.innerHeight);
        document.getElementById('canvas-container').appendChild(renderer.domElement);

        // 2. Thiết lập ánh sáng phức hợp (Ánh sáng giúp mây có khối 3D)
        const ambientLight = new THREE.AmbientLight(0x1b4965, 1.2); // Ánh sáng môi trường xanh lam
        scene.add(ambientLight);

        const directionalLight = new THREE.DirectionalLight(0xffffff, 2.0); // Ánh sáng chính chiếu rọi mây trắng
        directionalLight.position.set(0, 4, 5);
        scene.add(directionalLight);

        const blueLight = new THREE.PointLight(0x00b4d8, 5, 50); // Điểm sáng Neon Xanh cyan tạo điểm nhấn tinh xảo
        blueLight.position.set(0, 0, -10);
        scene.add(blueLight);

        // 3. Thuật toán tự tạo Texture Đám mây mềm (Soft Cloud Texture) bằng Canvas gốc
        function generateCloudTexture() {
            const canvas = document.createElement('canvas');
            canvas.width = 256; canvas.height = 256;
            const ctx = canvas.getContext('2d');
            
            // Tạo một khối nhòe mờ tâm tỏa tròn (Radial Gradient)
            const gradient = ctx.createRadialGradient(128, 128, 10, 128, 128, 128);
            gradient.addColorStop(0, 'rgba(255, 255, 255, 1)');
            gradient.addColorStop(0.2, 'rgba(212, 241, 244, 0.6)'); // Ánh xanh pastel nhẹ
            gradient.addColorStop(0.5, 'rgba(144, 224, 239, 0.2)');
            gradient.addColorStop(1, 'rgba(0, 0, 0, 0)');
            
            ctx.fillStyle = gradient;
            ctx.fillRect(0, 0, 256, 256);
            return new THREE.CanvasTexture(canvas);
        }

        const cloudTexture = generateCloudTexture();
        const clouds = [];

        // 4. Tạo cụm mây phức hợp phân tầng (Xếp chồng cấu trúc hình học)
        // Số lượng hạt mây lớn (350 hạt) kết hợp với nhau tạo ra các dải mây ngẫu nhiên không trùng lặp
        const cloudCount = 350;
        for (let i = 0; i < cloudCount; i++) {
            const size = Math.random() * 25 + 15; // Kích thước mây đa dạng
            const geometry = new THREE.PlaneGeometry(size, size);
            
            // Vật liệu xử lý hiệu ứng hòa trộn ánh sáng
            const material = new THREE.MeshLambertMaterial({
                map: cloudTexture,
                transparent: true,
                opacity: Math.random() * 0.4 + 0.15, // Độ đậm nhạt ngẫu nhiên
                blending: THREE.NormalBlending,
                depthWrite: false
            });
            
            const cloudParticle = new THREE.Mesh(geometry, material);
            
            // Phân bổ mây tỏa rộng khắp không gian 3D dạng hầm (Tunnel)
            cloudParticle.position.set(
                (Math.random() - 0.5) * 80,
                (Math.random() - 0.5) * 60,
                (Math.random() - 0.5) * 150 - 50
            );
            
            // Xoay góc ngẫu nhiên để các đám mây lồng vào nhau tự nhiên nhất
            cloudParticle.rotation.z = Math.random() * Math.PI * 2;
            
            scene.add(cloudParticle);
            clouds.push({
                mesh: cloudParticle,
                rotSpeed: (Math.random() - 0.5) * 0.002, // Tốc độ tự xoay quanh tâm cực nhẹ
                driftSpeed: Math.random() * 0.05 + 0.05   // Tốc độ mây trôi về phía camera
            });
        }

        camera.position.z = 30;

        // 5. Tương tác mượt mà với cảm ứng di chuyển chuột/vuốt màn hình điện thoại
        let targetX = 0, targetY = 0;
        const handleMove = (x, y) => {
            targetX = (x - window.innerWidth / 2) / 80;
            targetY = (y - window.innerHeight / 2) / 80;
        };

        document.addEventListener('mousemove', (e) => handleMove(e.clientX, e.clientY));
        document.addEventListener('touchmove', (e) => handleMove(e.touches[0].clientX, e.touches[0].clientY), { passive: true });

        // 6. Vòng lặp Render & Xử lý Chuyển động thời gian thực (Real-time Physics)
        function animate() {
            requestAnimationFrame(animate);

            // Camera di chuyển quán tính mượt mà (Lerp effect) tạo góc nhìn 3D cực thực
            camera.position.x += (targetX - camera.position.x) * 0.03;
            camera.position.y += (-targetY - camera.position.y) * 0.03;
            camera.lookAt(new THREE.Vector3(0, 0, -30));

            // Vòng lặp chuyển động của từng hạt mây
            clouds.forEach(cloud => {
                cloud.mesh.rotation.z += cloud.rotSpeed; // Mây tự cuộn
                cloud.mesh.position.z += cloud.driftSpeed; // Mây trôi về phía người xem

                // Nếu dải mây vượt quá tầm nhìn camera, đẩy nó về khoảng không xa xăm phía trước để tạo vòng lặp vô tận
                if (cloud.mesh.position.z > 40) {
                    cloud.mesh.position.z = -100;
                    cloud.mesh.position.x = (Math.random() - 0.5) * 80;
                    cloud.mesh.position.y = (Math.random() - 0.5) * 60;
                }
            });

            renderer.render(scene, camera);
        }

        animate();

        // Tự động căn chỉnh khi xoay ngang/dọc điện thoại hoặc đổi kích thước cửa sổ
        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        });
    </script>
</body>
</html>
