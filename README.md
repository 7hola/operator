<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <title>Christmas Particle Magic</title>
    <style>
        body { margin: 0; overflow: hidden; background-color: #050505; font-family: 'Arial', sans-serif; }
        #canvas-container { width: 100vw; height: 100vh; position: absolute; top: 0; left: 0; z-index: 1; }
        #video-input { display: none; transform: scaleX(-1); } /* 隐藏原始视频，但在逻辑中使用 */
        
        /* UI 界面 */
        #ui-layer {
            position: absolute;
            top: 20px;
            left: 20px;
            z-index: 10;
            color: white;
            pointer-events: none; /* 让鼠标事件穿透到 Canvas */
        }
        .control-group {
            pointer-events: auto;
            background: rgba(255, 255, 255, 0.1);
            backdrop-filter: blur(10px);
            padding: 15px;
            border-radius: 12px;
            border: 1px solid rgba(255,255,255,0.2);
            margin-bottom: 10px;
            display: flex;
            flex-direction: column;
            gap: 10px;
        }
        button, input[type="file"]::file-selector-button {
            background: linear-gradient(135deg, #ff4e50 0%, #f9d423 100%);
            border: none;
            padding: 10px 20px;
            border-radius: 8px;
            color: #000;
            font-weight: bold;
            cursor: pointer;
            transition: transform 0.2s;
        }
        button:hover { transform: scale(1.05); }
        .status { font-size: 12px; opacity: 0.8; margin-top: 5px; }
        
        /* 加载动画 */
        #loader {
            position: fixed; top: 0; left: 0; width: 100%; height: 100%;
            background: #000; z-index: 999; display: flex; justify-content: center; align-items: center;
            color: white; flex-direction: column; transition: opacity 0.5s;
        }
        .spinner { width: 40px; height: 40px; border: 4px solid #333; border-top: 4px solid #fff; border-radius: 50%; animation: spin 1s linear infinite; margin-bottom: 20px; }
        @keyframes spin { 0% { transform: rotate(0deg); } 100% { transform: rotate(360deg); } }
    </style>
</head>
<body>

    <div id="loader">
        <div class="spinner"></div>
        <div id="loading-text">正在初始化视觉引擎与摄像头...</div>
    </div>

    <video id="video-input" playsinline></video>
    <div id="canvas-container"></div>

    <div id="ui-layer">
        <div class="control-group">
            <h3>🎄 交互控制台</h3>
            <p style="font-size: 12px; margin:0;">🖐 张开手：散开旋转 | ✊ 握拳：聚合圣诞树 | 👌 捏合：抓取照片</p>
            <div class="status" id="gesture-status">当前手势: 等待检测...</div>
        </div>
        <div class="control-group">
            <label for="img-upload" style="font-size:14px; margin-bottom:5px;">添加照片到圣诞树 (支持多选)</label>
            <input type="file" id="img-upload" accept="image/*" multiple>
        </div>
        <div class="control-group">
             <button id="fullscreen-btn">全屏沉浸模式</button>
        </div>
    </div>

    <script async src="https://unpkg.com/es-module-shims@1.6.3/dist/es-module-shims.js"></script>
    <script type="importmap">
        {
            "imports": {
                "three": "https://unpkg.com/three@0.160.0/build/three.module.js",
                "three/addons/": "https://unpkg.com/three@0.160.0/examples/jsm/"
            }
        }
    </script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/control_utils/control_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js" crossorigin="anonymous"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/gsap/3.12.4/gsap.min.js"></script>

    <script type="module">
        import * as THREE from 'three';
        import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
        import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';
        import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';

        // --- 全局变量 ---
        let camera, scene, renderer, composer;
        let particleSystem, uniforms;
        let treeTopLight;
        let userImages = []; // 存储用户上传的图片纹理
        const videoElement = document.getElementById('video-input');
        const statusElement = document.getElementById('gesture-status');
        
        // 状态管理
        const state = {
            isFist: false,
            isPinching: false,
            pinchCooldown: false,
            mixRatio: 0, // 0 = Scatter, 1 = Tree
            rotationSpeed: 0.2
        };

        // 粒子配置
        const PARTICLE_COUNT = 4000;
        const ICONS = ['🎁', '🧦', '🔔', '👔', '🔺', '🧊', '🌲', '🎅', '●']; 
        
        // --- 1. 初始化 Three.js 场景 ---
        function init() {
            const container = document.getElementById('canvas-container');

            scene = new THREE.Scene();
            scene.fog = new THREE.FogExp
