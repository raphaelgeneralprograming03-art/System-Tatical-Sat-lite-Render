<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Systems Tactical Satellite Render Engine</title>
    <style>
        body {
            margin: 0;
            padding: 0;
            overflow: hidden;
            background-color: #050505;
            font-family: monospace;
        }
        #hud-overlay {
            position: absolute;
            top: 20px;
            left: 20px;
            color: #39ff14;
            text-shadow: 0 0 5px rgba(57,255,20,0.8);
            pointer-events: none;
            font-size: 12px;
            line-height: 1.6;
        }
        #loading {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            color: white;
            font-size: 18px;
            font-weight: bold;
        }
    </style>
    <!-- Carrega a biblioteca Three.js via CDN estável para o repositório -->
    <script src="https://cloudflare.com"></script>
</head>
<body>

    <div id="loading">COMPILANDO SHADERS DO TERRENO PROCEDURAL...</div>
    
    <div id="hud-overlay">
        SYS_STATUS: LINK ENGAGED<br>
        RESOLUTION: 4K HIGH-RES PROCEDURAL MATRIX<br>
        SATELLITE ALTITUDE: 850m<br>
        MHD PROPULSION: RE-ROUTING POWER<br>
        S.A.M.S CELL: LOCK-ON STANDBY
    </div>

    <script>
        // --- 1. CONFIGURAÇÃO DO CENÁRIO GLOBAL (THREE.JS) ---
        const cena = new THREE.Scene();
        cena.fog = new THREE.FogExp2(0x8a765d, 0.005); // Névoa de poeira desértica atmosférica

        const camera = new THREE.OrthographicCamera(
            window.innerWidth / -20, window.innerWidth / 20,
            window.innerHeight / 20, window.innerHeight / -20,
            1, 1000
        );
        // Posiciona a câmera exatamente no topo (Visão Nadir de Satélite Militar)
        camera.position.set(0, 150, 0);
        camera.lookAt(0, 0, 0);

        const renderizador = new THREE.WebGLRenderer({ antialias: true, powerPreference: "high-performance" });
        renderizador.setSize(window.innerWidth, window.innerHeight);
        renderizador.shadowMap.enabled = true;
        renderizador.shadowMap.type = THREE.PCFSoftShadowMap;
        document.body.appendChild(renderizador.domElement);

        // --- 2. ILUMINAÇÃO DE SATÉLITE REALISTA ---
        const luzSolar = new THREE.DirectionalLight(0xfff5e6, 1.5); // Luz solar forte de deserto
        luzSolar.position.set(60, 100, 40);
        luzSolar.castShadow = true;
        luzSolar.shadow.mapSize.width = 4096; // Resolução ultra-alta para sombras nítidas dos veículos
        luzSolar.shadow.mapSize.height = 4096;
        luzSolar.shadow.camera.near = 0.5;
        luzSolar.shadow.camera.far = 500;
        const d = 100;
        luzSolar.shadow.camera.left = -d; luzSolar.shadow.camera.right = d;
        luzSolar.shadow.camera.top = d; luzSolar.shadow.camera.bottom = -d;
        luzSolar.shadow.bias = -0.0005;
        cena.add(luzSolar);

        const luzAmbiente = new THREE.AmbientLight(0x443a2e, 0.6); // Luz difusa da atmosfera de poeira
        cena.add(luzAmbiente);

        // --- 3. ALGORITMO PROCEDURAL DE TERRENO (GOOGLE MAPS REALISM) ---
        // Criação de ruído matemático diretamente via Fragment Shader para performance 4K
        const malhaTerreno = new THREE.PlaneGeometry(300, 300, 120, 120);
        
        // Simula elevações, cânions e crateras modificando os vértices geometricamente
        const posicoes = malhaTerreno.attributes.position;
        for (let i = 0; i < posicoes.count; i++) {
            let x = posicoes.getX(i);
            let y = posicoes.getY(i);
            // Combinação de ondas senoidais pseudo-aleatórias para relevo de dunas e trincheiras
            let z = Math.sin(x * 0.05) * Math.cos(y * 0.05) * 4;
            z += Math.sin(x * 0.1) * 1.5; 
            z += Math.cos(y * 0.2) * 0.5; // Micro-textura do solo
            
            // Cavidades para trincheiras militares artificiais no cenário
            if (x > -30 && x < -10 && y > -20 && y < 40) z -= 3;
            if (x > 20 && x < 50 && y > -40 && y < -10) z -= 2.5;

            posicoes.setZ(i, z);
        }
        malhaTerreno.computeVertexNormals();

        // Canvas dinâmico para pintar a textura de solo árido e estradas de terra de satélite
        const canvasTextura = document.createElement('canvas');
        canvasTextura.width = 2048; canvasTextura.height = 2048;
        const ctxTxt = canvasTextura.getContext('2d');
        ctxTxt.fillStyle = '#bfa583'; ctxTxt.fillRect(0,0,2048,2048);
        // Ruído de areia fina carbonizada
        for(let i=0; i<100000; i++) {
            ctxTxt.fillStyle = Math.random() > 0.5 ? '#a68f6f' : '#cbbe9f';
            ctxTxt.fillRect(Math.random()*2048, Math.random()*2048, 2, 2);
        }
        // Desenho de estradas táticas desbotadas pelo vento
        ctxTxt.strokeStyle = '#937d5f'; ctxTxt.lineWidth = 40; ctxTxt.beginPath();
        ctxTxt.moveTo(0, 300); ctxTxt.lineTo(2048, 1800); ctxTxt.stroke();
        ctxTxt.moveTo(500, 0); ctxTxt.lineTo(1500, 2048); ctxTxt.stroke();

        const texturaSolo = new THREE.CanvasTexture(canvasTextura);
        const materialTerreno = new THREE.MeshStandardMaterial({
            map: texturaSolo,
            roughness: 0.9,
            metalness: 0.1,
            flatShading: true
        });

        const terreno = new THREE.Mesh(malhaTerreno, materialTerreno);
        terreno.rotation.x = -Math.PI / 2; // Deita o plano para servir de solo
        terreno.receiveShadow = true;
        cena.add(terreno);

        // --- 4. ALGORITMOS DE VETORIZAÇÃO E MODELAGEM DE VEÍCULOS ---
        const conteinerVeiculos = [];

        // Construtor procedural do Tanque Abrams / T-90
        function criarTanqueProcedural(cor, eAliado) {
            const grupo = new THREE.Group();
            
            // Chassi Principal
            const geoChassi = new THREE.BoxGeometry(6, 1.5, 9);
            const matChassi = new THREE.MeshStandardMaterial({ color: cor, roughness: 0.5 });
            const chassi = new THREE.Mesh(geoChassi, matChassi);
            chassi.position.y = 0.75;
            chassi.castShadow = true; chassi.receiveShadow = true;
            grupo.add(chassi);

            // Esteiras Magnéticas laterais (MHD / Rodas)
            const geoEsteira = new THREE.BoxGeometry(1, 1.2, 9.5);
            const matEsteira = new THREE.MeshStandardMaterial({ color: 0x1a1a1a, roughness: 0.9 });
            const estEsq = new THREE.Mesh(geoEsteira, matEsteira);
            estEsq.position.set(-3.2, 0.6, 0);
            estEsq.castShadow = true;
            const estDir = estEsq.clone(); estDir.position.x = 3.2;
            grupo.add(estEsq); grupo.add(estDir);

            // Torreta Giratória
            const geoTorreta = new THREE.BoxGeometry(4, 1, 5);
            const torreta = new THREE.Mesh(geoTorreta, matChassi);
            torreta.position.set(0, 1.8, -0.5);
            torreta.castShadow = true;
            grupo.add(torreta);

            // Canhão Principal de Almas Longas
            const geoCanhao = new THREE.CylinderGeometry(0.2, 0.2, 6);
            geoCanhao.rotateX(Math.PI / 2); // Aponta para frente
            const canhao = new THREE.Mesh(geoCanhao, new THREE.MeshStandardMaterial({ color: 0x222222 }));
            canhao.position.set(0, 1.8, 4);
            canhao.castShadow = true;
            grupo.add(canhao);

            grupo.userData = { eAliado: eAliado, tipo: 'tank', hp: 100 };
            return grupo;
        }

        // Construtor procedural do Caça F-35 Furtivo
        function criarCaçaProcedural() {
            const grupo = new THREE.Group();
            const matCaça = new THREE.MeshStandardMaterial({ color: 0x3a444d, metalness: 0.5, roughness: 0.3 });

            // Fuselagem central aerodinâmica
            const geoCorpo = new THREE.ConeGeometry(1.5, 12, 4);
            geoCorpo.rotateX(Math.PI / 2);
            const corpo = new THREE.Mesh(geoCorpo, matCaça);
            corpo.castShadow = true;
            grupo.add(corpo);

            // Asa Delta Furtiva Esquerda
            const geoAsaEsq = new THREE.BufferGeometry();
            const vertices = new Float32Array([
                0, 0, 2,     // Centro fuselagem
                -7, 0, -3,   // Ponta da asa
                0, 0, -4     // Cauda da asa
            ]);
            geoAsaEsq.setAttribute('position', new THREE.BufferAttribute(vertices, 3));
            geoAsaEsq.computeVertexNormals();
            const asaEsq = new THREE.Mesh(geoAsaEsq, matCaça);
            asaEsq.castShadow = true;
            grupo.add(asaEsq);

            // Asa Direita (Espelhada)
            const geoAsaDir = geoAsaEsq.clone();
            geoAsaDir.scale(-1, 1, 1);
            const asaDir = new THREE.Mesh(geoAsaDir, matCaça);
            asaDir.castShadow = true;
            grupo.add(asaDir);

            grupo.userData = { tipo: 'fighter', vel: 1.8 };
            return grupo;
        }

        // Construtor procedural do Drone MQ-9 Reaper
        function criarDroneProcedural() {
            const grupo = new THREE.Group();
            const matDrone = new THREE.MeshStandardMaterial({ color: 0x7f8c8d });

            const fuselagem = new THREE.Mesh(new THREE.CylinderGeometry(0.4, 0.2, 6), matDrone);
            fuselagem.rotateX(Math.PI/2);
            fuselagem.castShadow = true;
            grupo.add(fuselagem);

            // Envergadura massiva fina de asas planadoras de satélite
            const asas = new THREE.Mesh(new THREE.BoxGeometry(14, 0.1, 0.8), matDrone);
            asas.position.set(0, 0, 1);
            asas.castShadow = true;
            grupo.add(asas);

            grupo.userData = { tipo: 'drone', vel: 0.4 };
            return grupo;
        }

        // --- 5. INSTANCIAÇÃO DINÂMICA DO CAMPO DE BATALHA ---
