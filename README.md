
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
            letter-spacing: 1px;
            background: rgba(0, 0, 0, 0.8);
            padding: 20px 30px;
            border: 1px solid #39ff14;
            border-radius: 4px;
        }
    </style>
    <!-- Carrega a biblioteca Three.js via CDN estável -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
</head>
<body>

    <div id="loading">COMPILANDO SHADERS DO TERRENO PROCEDURAL...</div>
    
    <div id="hud-overlay">
        SYS_STATUS: LINK ENGAGED<br>
        RESOLUTION: 4K HIGH-RES PROCEDURAL MATRIX<br>
        SATELLITE ALTITUDE: 850m<br>
        MHD PROPULSION: RE-ROUTING POWER<br>
        S.A.M.S CELL: LOCK-ON STANDBY<br>
        <span id="hud-coordinates">TARGET LAT: 33.8921° N | LON: 44.3119° E</span>
    </div>

    <script>
        // --- 1. CONFIGURAÇÃO DO CENÁRIO GLOBAL (THREE.JS) ---
        const cena = new THREE.Scene();
        cena.fog = new THREE.FogExp2(0x8a765d, 0.003); // Névoa de poeira desértica atmosférica

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
        luzSolar.shadow.mapSize.width = 2048; // Alta resolução para sombras nítidas
        luzSolar.shadow.mapSize.height = 2048;
        luzSolar.shadow.camera.near = 0.5;
        luzSolar.shadow.camera.far = 500;
        const d = 100;
        luzSolar.shadow.camera.left = -d; luzSolar.shadow.camera.right = d;
        luzSolar.shadow.camera.top = d; luzSolar.shadow.camera.bottom = -d;
        luzSolar.shadow.bias = -0.0005;
        cena.add(luzSolar);

        const luzAmbiente = new THREE.AmbientLight(0x443a2e, 0.6); // Luz difusa da atmosfera de poeira
        cena.add(luzAmbiente);

        // --- 3. ALGORITMO PROCEDURAL DE TERRENO ---
        const malhaTerreno = new THREE.PlaneGeometry(300, 300, 120, 120);
        
        // Função matemática para calcular a altura exata em qualquer ponto (x, z)
        function obterAlturaTerreno(wx, wz) {
            let x = wx;
            let y = -wz; // Conversão de sistema cartesiano local do PlaneGeometry
            let z = Math.sin(x * 0.05) * Math.cos(y * 0.05) * 4;
            z += Math.sin(x * 0.1) * 1.5; 
            z += Math.cos(y * 0.2) * 0.5; // Micro-textura do solo
            
            // Cavidades para trincheiras militares artificiais no cenário
            if (x > -30 && x < -10 && y > -20 && y < 40) z -= 3;
            if (x > 20 && x < 50 && y > -40 && y < -10) z -= 2.5;

            return z;
        }

        // Aplicação da deformação no relevo
        const posicoes = malhaTerreno.attributes.position;
        for (let i = 0; i < posicoes.count; i++) {
            let x = posicoes.getX(i);
            let y = posicoes.getY(i);
            let z = obterAlturaTerreno(x, -y);
            posicoes.setZ(i, z);
        }
        malhaTerreno.computeVertexNormals();

        // Canvas dinâmico para pintar a textura do solo
        const canvasTextura = document.createElement('canvas');
        canvasTextura.width = 2048; canvasTextura.height = 2048;
        const ctxTxt = canvasTextura.getContext('2d');
        ctxTxt.fillStyle = '#bfa583'; ctxTxt.fillRect(0,0,2048,2048);
        
        for(let i=0; i<100000; i++) {
            ctxTxt.fillStyle = Math.random() > 0.5 ? '#a68f6f' : '#cbbe9f';
            ctxTxt.fillRect(Math.random()*2048, Math.random()*2048, 2, 2);
        }
        
        // Estradas táticas
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
        
        function criarTanqueProcedural(cor, eAliado) {
            const grupo = new THREE.Group();
            
            // Chassi Principal
            const geoChassi = new THREE.BoxGeometry(3, 1.2, 4.5);
            const matChassi = new THREE.MeshStandardMaterial({ color: cor, roughness: 0.5 });
            const chassi = new THREE.Mesh(geoChassi, matChassi);
            chassi.position.y = 0.6;
            chassi.castShadow = true; chassi.receiveShadow = true;
            grupo.add(chassi);

            // Esteiras laterais
            const geoEsteira = new THREE.BoxGeometry(0.6, 0.8, 4.8);
            const matEsteira = new THREE.MeshStandardMaterial({ color: 0x1a1a1a, roughness: 0.9 });
            const estEsq = new THREE.Mesh(geoEsteira, matEsteira);
            estEsq.position.set(-1.6, 0.4, 0);
            estEsq.castShadow = true;
            const estDir = estEsq.clone(); estDir.position.x = 1.6;
            grupo.add(estEsq); grupo.add(estDir);

            // Torreta Giratória
            const geoTorreta = new THREE.BoxGeometry(2, 0.8, 2.5);
            const torreta = new THREE.Mesh(geoTorreta, matChassi);
            torreta.position.set(0, 1.4, -0.2);
            torreta.castShadow = true;
            grupo.add(torreta);

            // Canhão
            const geoCanhao = new THREE.CylinderGeometry(0.12, 0.12, 3);
            geoCanhao.rotateX(Math.PI / 2);
            const canhao = new THREE.Mesh(geoCanhao, new THREE.MeshStandardMaterial({ color: 0x222222 }));
            canhao.position.set(0, 1.4, 2);
            canhao.castShadow = true;
            grupo.add(canhao);

            grupo.userData = { eAliado: eAliado, tipo: 'tank', v: 0.08, direcao: Math.random() * Math.PI * 2 };
            return grupo;
        }

        function criarCaçaProcedural() {
            const grupo = new THREE.Group();
            const matCaça = new THREE.MeshStandardMaterial({ color: 0x3a444d, metalness: 0.5, roughness: 0.3 });

            const geoCorpo = new THREE.ConeGeometry(1, 8, 4);
            geoCorpo.rotateX(Math.PI / 2);
            const corpo = new THREE.Mesh(geoCorpo, matCaça);
            corpo.castShadow = true;
            grupo.add(corpo);

            const geoAsaEsq = new THREE.BufferGeometry();
            const vertices = new Float32Array([
                0, 0, 1,
                -4, 0, -2,
                0, 0, -3
            ]);
            geoAsaEsq.setAttribute('position', new THREE.BufferAttribute(vertices, 3));
            geoAsaEsq.computeVertexNormals();
            const asaEsq = new THREE.Mesh(geoAsaEsq, matCaça);
            asaEsq.castShadow = true;
            grupo.add(asaEsq);

            const geoAsaDir = geoAsaEsq.clone();
            geoAsaDir.scale(-1, 1, 1);
            const asaDir = new THREE.Mesh(geoAsaDir, matCaça);
            asaDir.castShadow = true;
            grupo.add(asaDir);

            grupo.userData = { tipo: 'fighter', vel: 0.8 };
            return grupo;
        }

        function criarDroneProcedural() {
            const grupo = new THREE.Group();
            const matDrone = new THREE.MeshStandardMaterial({ color: 0x7f8c8d });

            const fuselagem = new THREE.Mesh(new THREE.CylinderGeometry(0.3, 0.15, 4), matDrone);
            fuselagem.rotateX(Math.PI/2);
            fuselagem.castShadow = true;
            grupo.add(fuselagem);

            const asas = new THREE.Mesh(new THREE.BoxGeometry(9, 0.08, 0.6), matDrone);
            asas.position.set(0, 0, 0.5);
            asas.castShadow = true;
            grupo.add(asas);

            grupo.userData = { tipo: 'drone', angulo: 0, raio: 35, vel: 0.01 };
            return grupo;
        }

        // --- 5. INSTANCIAÇÃO DINÂMICA DO CAMPO DE BATALHA ---
        const conteinerVeiculos = [];

        // Instanciação de Tanques Aliados (Verde/Caqui)
        for(let i = 0; i < 4; i++) {
            const t = criarTanqueProcedural(0x4b5320, true);
            t.position.set(-40 + i * 15, 0, -20 + (Math.random() * 20));
            cena.add(t);
            conteinerVeiculos.push(t);
        }

        // Instanciação de Tanques Inimigos (Camuflagem Escura)
        for(let i = 0; i < 4; i++) {
            const t = criarTanqueProcedural(0x2a2a2a, false);
            t.position.set(20 + i * 15, 0, 10 + (Math.random() * 20));
            cena.add(t);
            conteinerVeiculos.push(t);
        }

        // Instanciação de Esquadrilha de Caças
        const caca1 = criarCaçaProcedural();
        caca1.position.set(-100, 35, -50);
        caca1.rotation.y = Math.PI / 4;
        cena.add(caca1);
        conteinerVeiculos.push(caca1);

        // Instanciação de Drone de Reconhecimento
        const drone = criarDroneProcedural();
        drone.position.set(0, 25, 0);
        cena.add(drone);
        conteinerVeiculos.push(drone);

        // Oculta mensagem de carregamento quando o cenário estiver pronto
        document.getElementById('loading').style.display = 'none';

        // --- 6. CICLO DE ANIMAÇÃO E FÍSICA ---
        let tempo = 0;

        function animar() {
            requestAnimationFrame(animar);
            tempo += 0.01;

            conteinerVeiculos.forEach(obj => {
                const data = obj.userData;

                // Animação dos Tanques
                if (data.tipo === 'tank') {
                    obj.position.x += Math.sin(data.direcao) * data.v;
                    obj.position.z += Math.cos(data.direcao) * data.v;
                    obj.rotation.y = data.direcao;

                    // Ajusta a altura Y do tanque para acompanhar o relevo exato
                    const h = obterAlturaTerreno(obj.position.x, obj.position.z);
                    obj.position.y = h;

                    // Curva aleatória dentro dos limites do cenário
                    if (Math.abs(obj.position.x) > 80 || Math.abs(obj.position.z) > 80 || Math.random() < 0.005) {
                        data.direcao += (Math.random() - 0.5) * Math.PI;
                    }
                }

                // Animação dos Caças
                if (data.tipo === 'fighter') {
                    obj.position.x += Math.sin(obj.rotation.y) * data.vel;
                    obj.position.z += Math.cos(obj.rotation.y) * data.vel;

                    // Loop de patrulha aérea
                    if (obj.position.x > 120 || obj.position.z > 120) {
                        obj.position.set(-120, 35, -120 + Math.random() * 50);
                    }
                }

                // Animação do Drone (Órbita Circular)
                if (data.tipo === 'drone') {
                    data.angulo += data.vel;
                    obj.position.x = Math.cos(data.angulo) * data.raio;
                    obj.position.z = Math.sin(data.angulo) * data.raio;
                    obj.rotation.y = -data.angulo;
                }
            });

            // Atualização dinâmica do HUD tático
            if (Math.floor(tempo * 100) % 30 === 0) {
                const lat = (33.8921 + Math.sin(tempo) * 0.005).toFixed(4);
                const lon = (44.3119 + Math.cos(tempo) * 0.005).toFixed(4);
                document.getElementById('hud-coordinates').innerText = `TARGET LAT: ${lat}° N | LON: ${lon}° E`;
            }

            renderizador.render(cena, camera);
        }

        // --- 7. ADAPTAÇÃO DINÂMICA DE TELA ---
        window.addEventListener('resize', () => {
            camera.left = window.innerWidth / -20;
            camera.right = window.innerWidth / 20;
            camera.top = window.innerHeight / 20;
            camera.bottom = window.innerHeight / -20;
            camera.updateProjectionMatrix();

            renderizador.setSize(window.innerWidth, window.innerHeight);
        });

        // Inicia a simulação
        animar();
    </script>
</body>
</html>
