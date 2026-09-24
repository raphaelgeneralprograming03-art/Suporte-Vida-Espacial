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
        canvas { display: block; width: 100%; height: 100%; }
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
</head>
<body>

    <div id="canvas-container">
        <!-- Substituído para renderização 2D nativa e universal -->
        <canvas id="simuladorCanvas"></canvas>
    </div>

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
        const canvas = document.getElementById('simuladorCanvas');
        const ctx = canvas.getContext('2d');

        // Ajusta o tamanho do canvas nativo
        function redimensionar() {
            canvas.width = window.innerWidth;
            canvas.height = window.innerHeight;
        }
        window.addEventListener('resize', redimensionar);
        redimensionar();

        // Parâmetros do Reator Central
        let anguloRotacao = 0;

        // Configuração das Moléculas
        const particleCount = 40;
        const molecules = [];

        function criarMolecula(inicializarAleatorio = false) {
            return {
                x: (Math.random() * 0.4 + 0.3) * canvas.width, // Centralizado horizontalmente
                y: inicializarAleatorio ? Math.random() * canvas.height : -10,
                speed: 1.5 + Math.random() * 2,
                type: 0, // 0: CO2 (Vermelho), 1: O2 (Azul), 2: C (Cinza)
                radius: 4 + Math.random() * 2
            };
        }

        // Inicializa a lista de moléculas
        for (let i = 0; i < particleCount; i++) {
            molecules.push(criarMolecula(true));
        }

        // Dados do HUD
        let co2Lvl = 450;
        let o2Lvl = 21.0;
        let cGrams = 0.00;

        const co2El = document.getElementById('co2-val');
        const o2El = document.getElementById('o2-val');
        const cEl = document.getElementById('c-val');

        // Loop de Renderização 2D Altamente Compatível
        function draw() {
            // Limpa a tela com fundo verde militar escuro
            ctx.fillStyle = '#0c1a0c';
            ctx.fillRect(0, 0, canvas.width, canvas.height);

            const centerX = canvas.width / 2;
            const centerY = canvas.height / 2;
            const reatorLargura = Math.min(canvas.width * 0.35, 140);
            const reatorAltura = Math.min(canvas.height * 0.4, 260);

            // 1. DESENHAR O REATOR CENTRAL (Grade cibernética simulando 3D)
            ctx.strokeStyle = 'rgba(0, 255, 102, 0.4)';
            ctx.lineWidth = 1.5;
            
            // Desenha linhas verticais rotativas da grade do cilindro
            anguloRotacao += 0.01;
            for(let i = 0; i < 6; i++) {
                let offset = Math.sin(anguloRotacao + (i * Math.PI / 3)) * (reatorLargura / 2);
                ctx.beginPath();
                ctx.moveTo(centerX + offset, centerY - reatorAltura / 2);
                ctx.lineTo(centerX + offset, centerY + reatorAltura / 2);
                ctx.stroke();
            }

            // Desenha os anéis horizontais do Reator
            ctx.beginPath();
            ctx.arc(centerX, centerY - reatorAltura / 2, reatorLargura / 2, 0, Math.PI * 2);
            ctx.arc(centerX, centerY, reatorLargura / 2, 0, Math.PI * 2);
            ctx.arc(centerX, centerY + reatorAltura / 2, reatorLargura / 2, 0, Math.PI * 2);
            ctx.stroke();

            // 2. PROCESSAR E DESENHAR AS MOLÉCULAS
            molecules.forEach((mol, index) => {
                mol.y += mol.speed;

                // Zona de reação molecular (centro do reator)
                if (mol.type === 0 && mol.y >= (centerY - 30) && mol.y <= (centerY + 30)) {
                    if (Math.random() > 0.4) {
                        mol.type = 1; // Transforma em Oxigênio (Azul)
                        o2Lvl = Math.min(25.0, o2Lvl + 0.08);
                        if (co2Lvl > 100) co2Lvl -= 1;
                    } else {
                        mol.type = 2; // Transforma em Carbono (Cinza)
                        cGrams += 0.04;
                    }
                }

                // Define as cores com base no tipo químico atual
                if (mol.type === 0) ctx.fillStyle = '#ff3333'; // CO2
                else if (mol.type === 1) ctx.fillStyle = '#3399ff'; // O2
                else ctx.fillStyle = '#888888'; // Carbono

                // Desenha a molécula na tela
                ctx.beginPath();
                ctx.arc(mol.x, mol.y, mol.radius, 0, Math.PI * 2);
                ctx.fill();

                // Recicla a molécula se ela passar da base da tela
                if (mol.y > canvas.height + 10) {
                    molecules[index] = criarMolecula(false);
                }
            });

            // 3. ATUALIZAR HUD DE TEXTO
            co2El.innerText = Math.floor(co2Lvl) + " PPM";
            o2El.innerText = o2Lvl.toFixed(1) + "%";
            cEl.innerText = cGrams.toFixed(2) + "g";

            requestAnimationFrame(draw);
        }

        // Ação do Botão Injetar
        document.getElementById('trigger-pulse').addEventListener('click', () => {
            co2Lvl += 60;
            // Força moléculas do topo a resetarem para CO2
            for (let i = 0; i < 15; i++) {
                let m = criarMolecula(false);
                m.y = Math.random() * -100;
                molecules.push(m);
                if(molecules.length > particleCount) molecules.shift();
            }
        });

        // Inicia a animação imediatamente
        draw();
    </script>
</body>
</html>
