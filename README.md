<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Christmas Magic (CN-Fast)</title>
    <style>
        body { margin: 0; overflow: hidden; background-color: #050505; font-family: 'Arial', sans-serif; touch-action: none; }
        #canvas-container { width: 100vw; height: 100vh; position: absolute; top: 0; left: 0; z-index: 1; }
        #video-input { display: none; transform: scaleX(-1); } 
        
        /* UI 界面 */
        #ui-layer {
            position: absolute; top: 20px; left: 20px; z-index: 10;
            color: white; pointer-events: none; width: 90%;
        }
        .control-group {
            pointer-events: auto; background: rgba(255, 255, 255, 0.15);
            backdrop-filter: blur(10px); padding: 12px; border-radius: 12px;
            border: 1px solid rgba(255,255,255,0.2); margin-bottom: 10px;
        }
        button, label {
            background: linear-gradient(135deg, #ff4e50 0%, #f9d423 100%);
            border: none; padding: 8px 15px; border-radius: 6px;
            color: #000; font-weight: bold; cursor: pointer; font-size: 14px; display: inline-block;
        }
        input[type="file"] { display: none; } /* 隐藏原生丑陋的文件输入 */
        .status { font-size: 12px; opacity: 0.9; margin-top: 5px; color: #eee; }
        
        /* 加载动画 */
        #loader {
            position: fixed; top: 0; left: 0; width: 100%; height: 100%;
            background: #000; z-index: 999; display: flex; justify-content: center; align-items: center;
            color: white; flex-direction: column; transition: opacity 0.5s; text-align: center;
        }
        .spinner { width: 40px; height: 40px; border: 4px solid #333; border-top: 4px solid #fff; border-radius: 50%; animation: spin 1s linear infinite; margin-bottom: 20px; }
        @keyframes spin { 0% { transform: rotate(0deg); } 100% { transform: rotate(360deg); } }
    </style>
    
    <script src="https://lib.baomitu.com/vConsole/3.15.1/vconsole.min.js"></script>
    <script>
        // 初始化调试面板，如果手机报错，点绿色按钮看 Log
        var vConsole = new VConsole();
        console.log("页面初始化开始...");
    </script>
</head>
<body>

    <div id="loader">
        <div class="spinner"></div>
        <div id="loading-text">正在从国内镜像加载引擎...<br>请稍候 (首次加载约10秒)</div>
    </div>

    <video id="video-input" playsinline webkit-playsinline></video>
    <div id="canvas-container"></div>

    <div id="ui-layer">
        <div class="control-group">
            <div style="font-weight:bold; margin-bottom:5px;">🎄 圣诞粒子互动</div>
            <div class="status" id="gesture-status">等待摄像头授权...</div>
        </div>
        <div class="control-group" style="display:flex; gap:10px;">
            <label for="img-upload">📷 上传照片</label>
            <input type="file" id="img-upload" accept="image/*" multiple>
            <button id="fullscreen-btn">📺 全屏</button>
        </div>
    </div>

    <script async src="https://npm.elemecdn.com/es-module-shims@1.8.0/dist/es-module-shims.js"></script>
    <script type="importmap">
        {
            "imports": {
                "three": "https://npm.elemecdn.com/three@0.160.0/build/three.module.js",
                "three/addons/": "https://npm.elemecdn.com/three@0.160.0/examples/jsm/"
            }
        }
    </script>

    <script src="https://npm.elemecdn.com/@mediapipe/camera_utils/camera_utils.js" crossorigin="anonymous"></script>
    <script src="https://npm.elemecdn.com/@mediapipe/control_utils/control_utils.js" crossorigin="anonymous"></script>
    <script src="https://npm.elemecdn.com/@mediapipe/hands/hands.js" crossorigin="anonymous"></script>
    <script src="https://lib.baomitu.com/gsap/3.12.4/gsap.min.js"></script>

    <script type="module">
        import * as THREE from 'three';
        import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
        import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';
        import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';

        console.log("Three.js 模块加载成功");

        let camera, scene, renderer, composer;
        let particleSystem, uniforms;
        let treeTopLight;
        let userImages = [];
        const videoElement = document.getElementById('video-input');
        const statusElement = document.getElementById('gesture-status');
        const loadingText = document.getElementById('loading-text');

        // 检测手机端
        const isMobile = /Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(navigator.userAgent);
        const PARTICLE_COUNT = isMobile ? 1800 : 4000; // 手机端减少粒子防止发热

        const state = {
            isFist: false, isPinching: false, pinchCooldown: false,
            mixRatio: 0, rotationSpeed: 0.2
        };
        
        const ICONS = ['🎁', '🧦', '🔔', '👔', '🔺', '🧊', '🌲', '🎅', '●'];

        function init() {
            try {
                const container = document.getElementById('canvas-container');
                scene = new THREE.Scene();
                scene.fog = new THREE.FogExp2(0x050505, 0.002);

                camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 1000);
                camera.position.set(0, 0, isMobile ? 50 : 40); // 手机端相机拉远一点

                renderer = new THREE.WebGLRenderer({ antialias: false, alpha: true, powerPreference: "high-performance" });
                renderer.setSize(window.innerWidth, window.innerHeight);
                renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
                container.appendChild(renderer.domElement);

                // Bloom 后处理
                const renderScene = new RenderPass(scene, camera);
                const bloomPass = new UnrealBloomPass(new THREE.Vector2(window.innerWidth, window.innerHeight), 1.5, 0.4, 0.85);
                bloomPass.strength = 1.0; 
                bloomPass.radius = 0.5;
                composer = new EffectComposer(renderer);
                composer.addPass(renderScene);
                composer.addPass(bloomPass);

                // 灯光
                scene.add(new THREE.AmbientLight(0x404040));
                treeTopLight = new THREE.PointLight(0xffdd00, 2, 50);
                treeTopLight.position.set(0, 18, 0);
                scene.add(treeTopLight);

                createParticles();

                window.addEventListener('resize', onWindowResize);
                
                // 全屏兼容处理
                document.getElementById('fullscreen-btn').addEventListener('click', () => {
                   const doc = document.documentElement;
                   if (doc.requestFullscreen) doc.requestFullscreen();
                   else if (doc.webkitRequestFullscreen) doc.webkitRequestFullscreen();
                });

                document.getElementById('img-upload').addEventListener('change', handleImageUpload);
                
                animate();
                console.log("3D 场景初始化完成");
            } catch (e) {
                console.error("Init Error:", e);
                alert("3D引擎启动失败: " + e.message);
            }
        }

        function createParticles() {
            const geometry = new THREE.BufferGeometry();
            const positionsTree = [];
            const positionsScatter = [];
            const sizes = [];
            const colors = [];
            const textureIndices = [];
            
            const colorPalette = [0xff0000, 0x00ff00, 0xffd700, 0xffffff, 0x00bfff];

            for (let i = 0; i < PARTICLE_COUNT; i++) {
                // 树形
                const angle = i * 0.1;
                const height = 30 - (i / PARTICLE_COUNT) * 40;
                const radius = (30 - height) * 0.4 * Math.random();
                positionsTree.push(
                    radius * Math.cos(angle * 5),
                    height - 5,
                    radius * Math.sin(angle * 5)
                );

                // 散开形
                const rScatter = 45 * Math.cbrt(Math.random());
                const theta = Math.random() * 2 * Math.PI;
                const phi = Math.acos(2 * Math.random() - 1);
                positionsScatter.push(
                    rScatter * Math.sin(phi) * Math.cos(theta),
                    rScatter * Math.sin(phi) * Math.sin(theta),
                    rScatter * Math.cos(phi)
                );

                sizes.push(Math.random() * 1.5 + 0.5);
                const col = new THREE.Color(colorPalette[Math.floor(Math.random() * colorPalette.length)]);
                colors.push(col.r, col.g, col.b);
                textureIndices.push(Math.floor(Math.random() * ICONS.length));
            }

            geometry.setAttribute('position', new THREE.Float32BufferAttribute(positionsScatter, 3)); 
            geometry.setAttribute('posTree', new THREE.Float32BufferAttribute(positionsTree, 3));
            geometry.setAttribute('posScatter', new THREE.Float32BufferAttribute(positionsScatter, 3));
            geometry.setAttribute('size', new THREE.Float32BufferAttribute(sizes, 1));
            geometry.setAttribute('customColor', new THREE.Float32BufferAttribute(colors, 3));
            geometry.setAttribute('texIndex', new THREE.Float32BufferAttribute(textureIndices, 1));

            uniforms = {
                time: { value: 0 },
                mixRatio: { value: 0.0 },
                pointTexture: { value: createTextureAtlas() },
                atlasSize: { value: ICONS.length },
                globalScale: { value: 1.0 }
            };

            const material = new THREE.ShaderMaterial({
                uniforms: uniforms,
                vertexShader: `
                    attribute float size;
                    attribute vec3 customColor;
                    attribute vec3 posTree;
                    attribute vec3 posScatter;
                    attribute float texIndex;
                    varying vec3 vColor;
                    varying float vTexIndex;
                    uniform float time;
                    uniform float mixRatio;
                    uniform float globalScale;
                    void main() {
                        vColor = customColor;
                        vTexIndex = texIndex;
                        vec3 targetPos = mix(posScatter, posTree, mixRatio);
                        float noise = sin(time * 3.0 + targetPos.y * 0.1) * 0.3; 
                        if(mixRatio > 0.8) targetPos.x += noise; // 只在树形态时晃动
                        vec4 mvPosition = modelViewMatrix * vec4(targetPos * globalScale, 1.0);
                        gl_PointSize = size * (300.0 / -mvPosition.z);
                        gl_Position = projectionMatrix * mvPosition;
                    }
                `,
                fragmentShader: `
                    uniform sampler2D pointTexture;
                    uniform float atlasSize;
                    varying vec3 vColor;
                    varying float vTexIndex;
                    void main() {
                        float spriteStep = 1.0 / atlasSize;
                        vec2 uv = gl_PointCoord;
                        vec2 atlasUV = vec2(uv.x * spriteStep + vTexIndex * spriteStep, uv.y);
                        vec4 texColor = texture2D(pointTexture, atlasUV);
                        if (texColor.a < 0.5) discard;
                        gl_FragColor = vec4(texColor.rgb * vColor, texColor.a);
                    }
                `,
                blending: THREE.AdditiveBlending,
                depthTest: false,
                transparent: true
            });

            particleSystem = new THREE.Points(geometry, material);
            scene.add(particleSystem);
        }

        function createTextureAtlas() {
            const size = 128;
            const canvas = document.createElement('canvas');
            canvas.width = size * ICONS.length;
            canvas.height = size;
            const ctx = canvas.getContext('2d');
            ICONS.forEach((icon, i) => {
                ctx.font = '80px Arial';
                ctx.textAlign = 'center';
                ctx.textBaseline = 'middle';
                ctx.fillStyle = 'white';
                ctx.fillText(icon, i * size + size / 2, size / 2);
            });
            return new THREE.CanvasTexture(canvas);
        }

        // --- 核心修改：MediaPipe 使用国内镜像源 ---
        async function setupMediaPipe() {
            loadingText.innerText = "正在初始化 AI 视觉模型...";
            
            try {
                const hands = new Hands({locateFile: (file) => {
                    // 关键点：强制使用 Eleme CDN 加载 wasm 文件
                    return `https://npm.elemecdn.com/@mediapipe/hands/${file}`;
                }});
                
                hands.setOptions({
                    maxNumHands: 1, // 手机端限制1只手，提高性能
                    modelComplexity: 1,
                    minDetectionConfidence: 0.5,
                    minTrackingConfidence: 0.5
                });

                hands.onResults(onHandsResults);

                const cameraUtils = new Camera(videoElement, {
                    onFrame: async () => {
                        await hands.send({image: videoElement});
                    },
                    width: isMobile ? 320 : 640, // 手机端降低分辨率处理
                    height: isMobile ? 240 : 480
                });

                await cameraUtils.start();
                
                // 成功后移除 Loading
                const loader = document.getElementById('loader');
                loader.style.opacity = 0;
                setTimeout(() => loader.remove(), 500);
                statusElement.innerText = "摄像头已就绪，请举起手 👋";
                console.log("MediaPipe 启动成功");

            } catch (err) {
                console.error("Camera/MediaPipe Error:", err);
                loadingText.innerHTML = "❌ 启动失败<br>" + err.message + "<br>请确保在 HTTPS 环境下打开，并允许摄像头权限";
            }
        }

        function onHandsResults(results) {
            if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
                const lm = results.multiHandLandmarks[0];
                
                // 简单的手势逻辑
                const thumbTip = lm[4];
                const indexTip = lm[8];
                const wrist = lm[0];
                const middleTip = lm[12];

                // 捏合检测
                const pinchDist = Math.hypot(thumbTip.x - indexTip.x, thumbTip.y - indexTip.y);
                const isPinch = pinchDist < 0.05;

                // 握拳检测 (指尖 y > 关节 y, 注意坐标系)
                // 简化版：掌心到指尖距离缩短
                const handSize = Math.hypot(wrist.x - middleTip.x, wrist.y - middleTip.y);
                const isFist = handSize < 0.15; // 阈值需要调试

                // 交互反馈
                updateInteraction(isFist, isPinch, handSize);
                
                statusElement.innerText = `手势: ${isFist ? '✊ 聚合' : '🖐 散开'} | 捏合: ${isPinch ? '👌 是' : '否'}`;
            } else {
                updateInteraction(false, false, 0.3);
                statusElement.innerText = "未检测到手部";
            }
        }

        function updateInteraction(isFist, isPinch, handScale) {
            // 缩放
            let targetScale = 1.0 + (handScale * (isMobile? 3 : 2)); 
            gsap.to(uniforms.globalScale, { value: targetScale, duration: 0.5 });

            // 聚合/散开
            if (isFist) {
                gsap.to(state, { mixRatio: 1, duration: 1.5 });
                state.rotationSpeed = 0.05;
            } else {
                gsap.to(state, { mixRatio: 0, duration: 1 });
                state.rotationSpeed = 0.2;
            }
            uniforms.mixRatio.value = state.mixRatio;

            // 抓取图片
            if (isPinch && !state.isPinching && !state.pinchCooldown) {
                state.isPinching = true;
                triggerImageGrabEffect();
                state.pinchCooldown = true;
                setTimeout(() => { state.pinchCooldown = false; }, 1500);
            } else if (!isPinch) {
                state.isPinching = false;
            }
        }

        function triggerImageGrabEffect() {
            if (userImages.length === 0) {
                // 如果没有图片，闪烁一下灯光作为反馈
                gsap.to(treeTopLight, { intensity: 8, duration: 0.1, yoyo: true, repeat: 1 });
                return;
            }
            const imgTex = userImages[Math.floor(Math.random() * userImages.length)];
            const mesh = new THREE.Mesh(
                new THREE.PlaneGeometry(5, 5),
                new THREE.MeshBasicMaterial({ map: imgTex, transparent: true, side: THREE.DoubleSide })
            );
            
            // 随机位置出现
            mesh.position.set((Math.random()-0.5)*15, (Math.random()-0.5)*20, (Math.random()-0.5)*15);
            scene.add(mesh);
            mesh.lookAt(camera.position);

            // 动画
            const tl = gsap.timeline({ onComplete: () => scene.remove(mesh) });
            tl.from(mesh.scale, { x: 0, y: 0, duration: 0.5, ease: "back.out" });
            tl.to(mesh.position, { y: mesh.position.y + 5, duration: 2 }, "<");
            tl.to(mesh.material, { opacity: 0, duration: 0.5, delay: 1 });
        }

        function handleImageUpload(e) {
            const files = e.target.files;
            if (!files.length) return;
            const loader = new THREE.TextureLoader();
            for (let i = 0; i < files.length; i++) {
                const reader = new FileReader();
                reader.onload = (ev) => {
                    const tex = loader.load(ev.target.result);
                    tex.colorSpace = THREE.SRGBColorSpace;
                    userImages.push(tex);
                };
                reader.readAsDataURL(files[i]);
            }
            statusElement.innerText = `已添加 ${files.length} 张照片`;
        }

        function animate() {
            requestAnimationFrame(animate);
            uniforms.time.value += 0.01;
            particleSystem.rotation.y += state.rotationSpeed * 0.05;
            composer.render();
        }

        function onWindowResize() {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
            composer.setSize(window.innerWidth, window.innerHeight);
        }

        // 启动
        init();
        setupMediaPipe();

    </script>
</body>
</html>
