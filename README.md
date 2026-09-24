<DOCTYPE html>
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
    <!-- Importação externa segura via CDN estável -->
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
        const container = document.getElementById('canvas-container');
        const scene = new THREE.Scene();

        // Câmera com FOV e profundidade otimizados
        const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 1000);
        camera.position.set(0, 2, 12);
        camera.lookAt(0, 0, 0);

        // AJUSTE MOBILE: Inicialização forçando alto desempenho em GPUs móveis
        const renderer = new THREE.WebGLRenderer({ antialias: true, powerPreference: "high-performance" });
        renderer.setSize(window.innerWidth, window.innerHeight);
        
        // AJUSTE MOBILE: Limita o multiplicador de pixels em telas de alta densidade (evita sobrecarga e tela preta)
        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
        
        // Fundo militar translúcido visível para atestar o funcionamento do canvas
        renderer.setClearColor(0x0c1a0c, 1); 
        container.appendChild(renderer.domElement);

        // Iluminação geral tridimensional
        const ambientLight = new THREE.AmbientLight(0xffffff, 0.8);
        scene.add(ambientLight);
        
        const dirLight = new THREE.DirectionalLight(0xffffff, 1.0);
        dirLight.position.set(5, 10, 7);
        scene.add(dirLight);

        // Estrutura Central (Reator)
        const coreGeo = new THREE.CylinderGeometry(2, 2, 5, 16, 1, true);
        const coreMat = new THREE.MeshBasicMaterial({ 
            color: 0x00ff66, 
            wireframe: true, 
            transparent: true, 
            opacity: 0.3 
        });
        const reactorCore = new THREE.Mesh(coreGeo, coreMat);
        scene.add(reactorCore);

        // Anel do Reator
        const ringGeo = new THREE.RingGeometry(2.3, 2.5, 32);
        const ringMat = new THREE.MeshBasicMaterial({ color: 0x00ff66, side: THREE.DoubleSide, transparent: true, opacity: 0.7 });
        const ring1 = new THREE.Mesh(ringGeo, ringMat);
        ring1.rotation.x = Math.PI / 2;
        scene.add(ring1);

        // Gerenciamento Estável de Moléculas 3D Reais
        const particleCount = 45; // Quantidade otimizada para navegadores mobile
        const molecules = [];
        
        const matCO2 = new THREE.MeshBasicMaterial({ color: 0xff3333 });
        const matO2 = new THREE.MeshBasicMaterial({ color: 0x3399ff });
        const matC = new THREE.MeshBasicMaterial({ color: 0xaaaaaa });
        const sphereGeo = new THREE.SphereGeometry(0.15, 6, 6); // Malhas ultra leves de carregamento garantido

        function initMolecule(mesh) {
            mesh.position.x = (Math.random() - 0.5) * 3.5;
            mesh.position.y = Math.random() * 4 + 3;
            mesh.position.z = (Math.random() - 0.5) * 3.5;
            mesh.material = matCO2;
            mesh.userData = { type: 0, speed: 0.02 + Math.random() * 0.03 };
        }

        for (let i = 0; i < particleCount; i++) {
            const molMesh = new THREE.Mesh(sphereGeo, matCO2);
            initMolecule(molMesh);
            molMesh.position.y = Math.random() * 7 - 2; // Distribuição inicial randômica na tela
            scene.add(molMesh);
            molecules.push(molMesh);
        }

        // Dados do Painel
        let co2Lvl = 450;
        let o2Lvl = 21.0;
        let cGrams = 0.00;

        const co2El = document.getElementById('co2-val');
        const o2El = document.getElementById('o2-val');
        const cEl = document.getElementById('c-val');

        // Loop de Renderização e Simulação Físico-Química
        function animate() {
            requestAnimationFrame(animate);

            reactorCore.rotation.y += 0.005;
            ring1.position.y = Math.sin(Date.now() * 0.002) * 0.4;

            molecules.forEach(mol => {
                mol.position.y -= mol.userData.speed;

                // Transmutação molecular na barreira elétrica do núcleo
                if (mol.userData.type === 0 && mol.position.y <= 0.6 && mol.position.y >= -0.6) {
                    if (Math.random() > 0.4) {
                        mol.userData.type = 1; // Oxigênio (Azul)
                        mol.material = matO2;
                        o2Lvl = Math.min(25.0, o2Lvl + 0.008);
                        if (co2Lvl > 100) co2Lvl -= 1;
                    } else {
                        mol.userData.type = 2; // Carbono Sólido (Cinza)
                        mol.material = matC;
                        cGrams += 0.02;
                    }
                }

                // Reciclagem da molécula ao sair do campo inferior
                if (mol.position.y < -5) {
                    initMolecule(mol);
                }
            });

            // Atualização contínua de Telemetria Textual
            co2El.innerText = Math.floor(co2Lvl) + " PPM";
            o2El.innerText = o2Lvl.toFixed(1) + "%";
            cEl.innerText = cGrams.toFixed(2) + "g";

            renderer.render(scene, camera);
        }

        // Clique do Botão Injetar
        document.getElementById('trigger-pulse').addEventListener('click', () => {
            co2Lvl += 60;
            molecules.forEach(mol => {
                if (mol.position.y < 0) {
                    initMolecule(mol);
                }
            });
        });

        // Responsividade para alteração de orientação (Retrato / Paisagem no Celular)
        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        });

        animate();
    </script>
</body>
</html>
