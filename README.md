<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ULTRA_STALKER_3D | ENERGOPLOK_5</title>
    <style>
        @import url('https://googleapis.com');
        
        body { margin: 0; background: #000; overflow: hidden; font-family: 'Orbitron', sans-serif; }
        
        /* ГЛАВНОЕ МЕНЮ */
        #ui-layer {
            position: absolute; width: 100vw; height: 100vh;
            background: radial-gradient(circle, #200 0%, #000 100%);
            display: flex; flex-direction: column; justify-content: center;
            align-items: center; z-index: 100; transition: 0.5s;
        }

        h1 { 
            color: #f00; font-size: clamp(30px, 8vw, 70px); 
            text-shadow: 0 0 20px #f00; margin-bottom: 40px; 
            letter-spacing: 15px; text-transform: uppercase;
            animation: glitch 0.2s infinite;
        }

        .btn-start {
            padding: 20px 80px; background: transparent; border: 3px solid #f00;
            color: #f00; font-size: 24px; cursor: pointer;
            box-shadow: 0 0 15px #f00; transition: 0.3s;
            text-transform: uppercase; font-weight: 900;
        }

        .btn-start:hover { background: #f00; color: #000; box-shadow: 0 0 60px #f00; transform: scale(1.1); }

        #hud {
            position: absolute; top: 30px; left: 30px; color: #f00;
            pointer-events: none; display: none; text-shadow: 0 0 green;
            font-size: 14px; z-index: 50; line-height: 1.6;
        }

        @keyframes glitch {
            0% { transform: skew(0deg); }
            50% { transform: skew(2deg); opacity: 0.8; }
            100% { transform: skew(0deg); }
        }

        canvas { display: block; width: 100%; height: 100%; }
    </style>
</head>
<body>

    <div id="ui-layer">
        <h1>NEON_STALKER</h1>
        <button class="btn-start" onclick="initMission()">[ EXECUTE_MISSION ]</button>
        <div style="color: #400; margin-top: 30px; letter-spacing: 3px; font-size: 12px;">
            WASD - MOVE | MOUSE - LOOK | ESC - PAUSE
        </div>
    </div>

    <div id="hud">
        CORE: ENERGOPLOK_5_ACTIVE <br>
        FPS: <span id="fps-val">...</span> <br>
        ENTITY_DIST: <span id="dist-val">SCANNING</span>
    </div>

    <!-- Подключаем движок Three.js -->
    <script src="https://cloudflare.com"></script>

    <script>
        let scene, camera, renderer, monster, flashlight, clock;
        let keys = {};
        let isStarted = false;
        let lastTime = 0;

        function initMission() {
            document.getElementById('ui-layer').style.display = 'none';
            document.getElementById('hud').style.display = 'block';
            isStarted = true;
            
            setup3D();
            loop();
            
            // Блокировка мышки как в Unity
            document.body.requestPointerLock = document.body.requestPointerLock || document.body.mozRequestPointerLock;
            document.body.requestPointerLock();
        }

        function setup3D() {
            scene = new THREE.Scene();
            scene.fog = new THREE.FogExp2(0x000000, 0.12);

            camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
            camera.position.set(0, 1.7, 15); // Безопасный спавн

            renderer = new THREE.WebGLRenderer({ antialias: true, powerPreference: "high-performance" });
            renderer.setSize(window.innerWidth, window.innerHeight);
            renderer.shadowMap.enabled = true;
            document.body.appendChild(renderer.domElement);

            // Пол
            const floor = new THREE.Mesh(
                new THREE.PlaneGeometry(200, 200),
                new THREE.MeshStandardMaterial({ color: 0x0a0a0a, roughness: 0.1 })
            );
            floor.rotation.x = -Math.PI / 2;
            floor.receiveShadow = true;
            scene.add(floor);

            // Колонны (Лабиринт)
            for(let i=0; i<70; i++) {
                const col = new THREE.Mesh(
                    new THREE.BoxGeometry(2, 10, 2),
                    new THREE.MeshStandardMaterial({ color: 0x110000, emissive: 0x220000 })
                );
                col.position.set(Math.random()*120-60, 5, Math.random()*120-60);
                col.castShadow = true;
                scene.add(col);
            }

            // Фонарик (SpotLight)
            flashlight = new THREE.SpotLight(0xffffff, 4, 40, Math.PI/6, 0.5, 1);
            flashlight.castShadow = true;
            camera.add(flashlight);
            flashlight.target.position.set(0, 0, -1);
            camera.add(flashlight.target);
            scene.add(camera);

            // Монстр (Entity)
            const mGroup = new THREE.Group();
            const body = new THREE.Mesh(new THREE.BoxGeometry(0.6, 4, 0.6), new THREE.MeshBasicMaterial({color: 0x000000}));
            const eyes = new THREE.Mesh(new THREE.BoxGeometry(0.4, 0.1, 0.1), new THREE.MeshBasicMaterial({color: 0xff0000}));
            eyes.position.set(0, 1.6, 0.35);
            mGroup.add(body, eyes);
            monster = mGroup;
            monster.position.set(30, 2, -30);
            scene.add(monster);

            clock = new THREE.Clock();

            // Обработка ввода
            window.addEventListener('keydown', e => keys[e.code] = true);
            window.addEventListener('keyup', e => keys[e.code] = false);

            document.addEventListener('mousemove', e => {
                if (document.pointerLockElement) {
                    camera.rotation.y -= e.movementX * 0.002;
                    camera.rotation.x -= e.movementY * 0.002;
                    camera.rotation.x = Math.max(-1.5, Math.min(1.5, camera.rotation.x));
                }
            });
            camera.rotation.order = 'YXZ';
        }

        function loop(now) {
            if(!isStarted) return;
            requestAnimationFrame(loop);
            
            const dt = clock.getDelta();
            
            // Расчет FPS для твоего реактора
            if (now - lastTime >= 1000) {
                document.getElementById('fps-val').innerText = Math.round(1/dt);
                lastTime = now;
            }

            // Плавное медленное движение
            const speed = 3.5 * dt;
            if(keys['KeyW']) camera.translateZ(-speed);
            if(keys['KeyS']) camera.translateZ(speed);
            if(keys['KeyA']) camera.translateX(-speed);
            if(keys['KeyD']) camera.translateX(speed);
            camera.position.y = 1.7;

            // Интеллект монстра
            monster.lookAt(camera.position);
            monster.position.x -= (monster.position.x - camera.position.x) * 0.007;
            monster.position.z -= (monster.position.z - camera.position.z) * 0.007;

            // Дистанция и глитчи
            const dist = camera.position.distanceTo(monster.position);
            document.getElementById('dist-val').innerText = dist.toFixed(1) + "m";
            
            if(dist < 5) {
                flashlight.intensity = 2 + Math.random() * 6; // Свет мерцает
            } else {
                flashlight.intensity = 4;
            }
   //Привет
            if(dist < 0.8) {
                alert("STALKER_DETECTION_CRITICAL: MISSION_FAILED");
                location.reload();
            }

            renderer.render(scene, camera);
        }

        // Авто-ресайз окна
        window.addEventListener('resize', () => {
            if(isStarted) {
                camera.aspect = window.innerWidth / window.innerHeight;
                camera.updateProjectionMatrix();
                renderer.setSize(window.innerWidth, window.innerHeight);
            }
        });
    </script>
</body>
</html>
