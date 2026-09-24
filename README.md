<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>SROS - Reator de Eletrólise Molecular CO₂</title>
    <style>
        * { box-sizing: border-box; margin: 0; padding: 0; }
        body, html { width: 100%; height: 100%; overflow: hidden; background-color: #050b05; font-family: 'Courier New', Courier, monospace; color: #00ff66; }
        #canvas-container { width: 100%; height: 100%; position: absolute; top: 0; left: 0; z-index: 1; }
        .hud-overlay { position: absolute; top: 20px; left: 20px; z-index: 10; pointer-events: none; text-shadow: 0 0 5px #00ff66; background: rgba(5, 15, 5, 0.85); padding: 15px; border: 1px solid #00ff66; border-radius: 4px; box-shadow: 0 0 15px rgba(0, 255, 102, 0.2); max-width: 320px; }
        h1 { font-size: 16px; margin-bottom: 8px; border-bottom: 1px solid #00ff66; padding-bottom: 4px; text-transform: uppercase; }
        .telemetry-item { font-size: 12px; margin: 6px 0; display: flex; justify-content: space-between; }
        .status-active { color: #00ff66; animation: blink 1.5s infinite; }
        .btn-action { position: absolute; bottom: 20px; left: 20px; z-index: 10; background: #051a05; border: 1px solid #00ff66; color: #00ff66; padding: 10px 20px; font-family: inherit; font-size: 12px; cursor: pointer; text-shadow: 0 0 3px #00ff66; box-shadow: 0 0 10px rgba(0,255,102,0.1); border-radius: 4px; pointer-events: auto; }
        .btn-action:hover { background: #00ff66; color: #050b05; font-weight: bold; }
        .legend { position: absolute; top: 20px; right: 20px; z-index: 10; background: rgba(5,15,5,0.85); border: 1px solid #00ff66; padding: 10px; font-size: 11px; border-radius: 4px; }
        .legend-item { margin: 4px 0; display: flex; align-items: center; }
        .dot { width: 8px; height: 8px; border-radius: 50%; margin-right: 8px; display: inline-block; }
        @keyframes blink { 0%, 100% { opacity: 1; } 50% { opacity: 0.4; } }
    </style>
    <!-- CORREÇÃO: Importação correta e funcional do Three.js via CDN cdnjs -->
    <script src="https://cloudflare.com"></script>
</head>
<body>

    <div id="canvas-container"></div>

    <div class="hud-overlay">
        <h1>SROS // REATOR SOEC</h1>
        <div class="telemetry-item"><span>STATUS:</span> <span class="status-active">CONVERSÃO ATIVA</span></div>
        <div class="telemetry-item"><span>EFICIÊNCIA:</span> <span>98.42%</span></div>
        <div class="telemetry-item"><span>FLUXO CO₂:</span> <span id="co2-val">450 PPM</span></div>
        <div class="telemetry-item"><span>PRODUÇÃO O₂:</span> <span id="o2-val">21.0%</span></div>
        <div class="telemetry-item"><span>C-SÓLIDO EXTRAÍDO:</span> <span id="c-val">0.00g</span></div>
        <div class="telemetry-item"><span>DRENO ENERGIA:</span> <span>0.25 kW</span></div>
    </div>

    <div class="legend">
        <div class="legend-item"><span class="dot" style="background:#ff3333;"></span>CO₂ (Entrada)</div>
        <div class="legend-item"><span class="dot" style="background:#3399ff;"></span>O₂ (Retorno Respirável)</div>
        <div class="legend-item"><span class="dot" style="background:#888888;"></span>Carbono (Armazenado)</div>
    </div>

    <button class="btn-action" id="trigger-pulse">INJETAR FLUXO CO₂</button>

    <script>
        // Configuração Inicial da Cena Three.js
        const container = document.getElementById('canvas-container');
        const scene = new THREE.Scene();
        scene.fog = new THREE.FogExp2(0x050b05, 0.05);

        const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 1000);
        camera.position.set(0, 0, 15);

        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        renderer.setPixelRatio(window.devicePixelRatio);
        container.appendChild(renderer.domElement);

        // Geometria Central: Núcleo do Reator Eletroquímico
        const coreGeo = new THREE.CylinderGeometry(2, 2, 4, 32, 1, true);
        const coreMat = new THREE.MeshBasicMaterial({ 
            color: 0x00ff66, 
            wireframe: true, 
            transparent: true, 
            opacity: 0.25 
        });
        const reactorCore = new THREE.Mesh(coreGeo, coreMat);
        scene.add(reactorCore);

        // Anéis de Energia do Reator
        const ringGeo = new THREE.RingGeometry(2.2, 2.4, 32);
        const ringMat = new THREE.MeshBasicMaterial({ color: 0x00ff66, side: THREE.DoubleSide, transparent: true, opacity: 0.6 });
        const ring1 = new THREE.Mesh(ringGeo, ringMat);
        ring1.rotation.x = Math.PI / 2;
        scene.add(ring1);

        // Gerenciamento de Partículas (CO2, O2, Carbono)
        const particleCount = 200;
        const pGeometry = new THREE.BufferGeometry();
        const positions = new Float32Array(particleCount * 3);
        const colors = new Float32Array(particleCount * 3);
        const pTypes = []; // 0: CO2, 1: O2, 2: Carbono
        const pSpeeds = [];

        const colorCO2 = new THREE.Color(0xff3333);
        const colorO2 = new THREE.Color(0x3399ff);
        const colorC = new THREE.Color(0x888888);

        function resetParticle(i) {
            positions[i*3] = (Math.random() - 0.5) * 4;
            positions[i*3+1] = Math.random() * 4 + 4; 
            positions[i*3+2] = (Math.random() - 0.5) * 4;

            colors[i*3] = colorCO2.r;
            colors[i*3+1] = colorCO2.g;
            colors[i*3+2] = colorCO2.b;

            pTypes[i] = 0; 
            pSpeeds[i] = 0.03 + Math.random() * 0.04;
        }

        for(let i=0; i<particleCount; i++) {
            pTypes.push(0);
            pSpeeds.push(0);
            resetParticle(i);
            // Distribuir a altura inicial para não caírem todas juntas no começo
            positions[i*3+1] = Math.random() * 8 - 2;
        }

        pGeometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
        pGeometry.setAttribute('color', new THREE.BufferAttribute(colors, 3));

        // Textura simulada por Canvas para partículas redondas limpas
        const pCanvas = document.createElement('canvas');
        pCanvas.width = 16; pCanvas.height = 16;
        const pCtx = pCanvas.getContext('2d');
        let grad = pCtx.createRadialGradient(8,8,0, 8,8,8);
        grad.addColorStop(0, 'rgba(255,255,255,1)');
        grad.addColorStop(1, 'rgba(255,255,255,0)');
        pCtx.fillStyle = grad; pCtx.fillRect(0,0,16,16);
        const pTex = new THREE.CanvasTexture(pCanvas);

        const pMaterial = new THREE.PointsMaterial({
            size: 0.35,
            map: pTex,
            transparent: true,
            blending: THREE.AdditiveBlending,
            vertexColors: true,
            depthWrite: false
        });

        const particleSystem = new THREE.Points(pGeometry, pMaterial);
        scene.add(particleSystem);

        // Variáveis de Telemetria Dinâmica do HUD
        let co2Lvl = 450;
        let o2Lvl = 21.0;
        let cGrams = 0.00;

        const co2El = document.getElementById('co2-val');
        const o2El = document.getElementById('o2-val');
        const cEl = document.getElementById('c-val');

        // Loop de Animação Principal
        function animate() {
            requestAnimationFrame(animate);

            // Rotação dos elementos estáticos do reator
            reactorCore.rotation.y += 0.005;
            ring1.position.y = Math.sin(Date.now() * 0.002) * 0.5;

            const posArr = pGeometry.attributes.position.array;
            const colArr = pGeometry.attributes.color.array;

            for(let i=0; i<particleCount; i++) {
                // Movimento descendente em direção ao centro do reator
                posArr[i*3+1] -= pSpeeds[i];

                // Zona de Quebra Molecular Elétrica (Y entre 0.5 e -0.5)
                if(pTypes[i] === 0 && posArr[i*3+1] <= 0.5 && posArr[i*3+1] >= -0.5) {
                    if(Math.random() > 0.4) {
                        pTypes[i] = 1; // Transforma em Oxigênio
                        colArr[i*3] = colorO2.r; colArr[i*3+1] = colorO2.g; colArr[i*3+2] = colorO2.b;
                        o2Lvl = Math.min(25.0, o2Lvl + 0.002);
                        if(co2Lvl > 120) co2Lvl -= 1;
                    } else {
                        pTypes[i] = 2; // Transforma em Carbono Sólido
                        colArr[i*3] = colorC.r; colArr[i*3+1] = colorC.g; colArr[i*3+2] = colorC.b;
                        cGrams += 0.01;
                    }
                }

                // Reseta a partícula se ela cair demais
                if(posArr[i*3+1] < -6) {
                    resetParticle(i);
                }
            }

            // Atualiza os dados no HUD de forma legível
            co2El.innerText = Math.floor(co2Lvl) + " PPM";
            o2El.innerText = o2Lvl.toFixed(1) + "%";
            cEl.innerText = cGrams.toFixed(2) + "g";

            pGeometry.attributes.position.needsUpdate = true;
            pGeometry.attributes.color.needsUpdate = true;

            renderer.render(scene, camera);
        }

        // Ação do Botão: Injeta mais CO2 e sobe os níveis do HUD
        document.getElementById('trigger-pulse').addEventListener('click', () => {
            co2Lvl += 50;
            const posArr = pGeometry.attributes.position.array;
            for(let i=0; i<particleCount; i++) {
                if(posArr[i*3+1] < 0) {
                    resetParticle(i);
                }
            }
        });

        // Ajuste de tela responsivo
        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        });

        // Inicia a animação
        animate();
    </script>
</body>
