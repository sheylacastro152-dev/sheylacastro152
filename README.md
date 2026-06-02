<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Escáner QR Todo en Uno</title>
    
    <script src="https://unpkg.com/html5-qrcode" type="text/javascript"></script>

    <style>
        :root {
            --primary-color: #4f46e5;
            --primary-hover: #4338ca;
            --bg-color: #f3f4f6;
            --card-bg: #ffffff;
            --text-color: #1f2937;
            --border-color: #e5e7eb;
        }

        body {
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
            background-color: var(--bg-color);
            color: var(--text-color);
            margin: 0;
            padding: 20px;
            display: flex;
            justify-content: center;
        }

        .container {
            max-width: 600px;
            width: 100%;
            background: var(--card-bg);
            padding: 24px;
            border-radius: 12px;
            box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
        }

        h1, h2 {
            text-align: center;
            margin-bottom: 8px;
        }

        .subtitle {
            text-align: center;
            color: #6b7280;
            margin-bottom: 24px;
        }

        .btn-primary {
            display: block;
            width: 100%;
            background-color: var(--primary-color);
            color: white;
            border: none;
            padding: 14px;
            font-size: 16px;
            font-weight: bold;
            border-radius: 8px;
            cursor: pointer;
            transition: background 0.2s ease;
        }

        .btn-primary:hover {
            background-color: var(--primary-hover);
        }

        #reader-container {
            margin-top: 20px;
            border-radius: 8px;
            overflow: hidden;
            border: 2px solid var(--border-color);
        }

        #reader {
            width: 100%;
        }

        .hidden {
            display: none;
        }

        hr {
            margin: 30px 0;
            border: 0;
            border-top: 1px solid var(--border-color);
        }

        .table-wrapper {
            overflow-x: auto;
        }

        table {
            width: 100%;
            border-collapse: collapse;
            margin-top: 12px;
            font-size: 14px;
        }

        th, td {
            padding: 12px;
            text-align: left;
            border-bottom: 1px solid var(--border-color);
        }

        th {
            background-color: #f9fafb;
            color: #4b5563;
            font-weight: 600;
        }

        td a {
            color: var(--primary-color);
            text-decoration: none;
            word-break: break-all;
        }

        td a:hover {
            text-decoration: underline;
        }
    </style>
</head>
<body>

    <div class="container">
        <h1>Escáner de Códigos QR</h1>
        <p class="subtitle">Escanea y guarda tu historial localmente</p>
        
        <button id="btn-scan" class="btn-primary">Iniciar Cámara</button>
        
        <div id="reader-container" class="hidden">
            <div id="reader"></div>
        </div>

        <hr>

        <h2>Historial de Lecturas</h2>
        <div class="table-wrapper">
            <table id="history-table">
                <thead>
                    <tr>
                        <th>Fecha y Hora</th>
                        <th>Contenido / Enlace</th>
                    </tr>
                </thead>
                <tbody id="history-body">
                    </tbody>
            </table>
        </div>
    </div>

    <script>
        document.addEventListener("DOMContentLoaded", () => {
            const btnScan = document.getElementById("btn-scan");
            const readerContainer = document.getElementById("reader-container");
            const historyBody = document.getElementById("history-body");
            
            let html5QrCode;
            let isScanning = false;

            // Cargar historial guardado al iniciar la página
            loadHistory();

            btnScan.addEventListener("click", () => {
                if (!isScanning) {
                    startScanner();
                } else {
                    stopScanner();
                }
            });

            function startScanner() {
                readerContainer.classList.remove("hidden");
                btnScan.textContent = "Detener Cámara";
                btnScan.style.backgroundColor = "#ef4444"; // Cambia a color rojo de detención
                isScanning = true;

                html5QrCode = new Html5Qrcode("reader");
                
                const config = { fps: 10, qrbox: { width: 250, height: 250 } };

                // 'environment' prioriza la cámara trasera en dispositivos móviles
                html5QrCode.start(
                    { facingMode: "environment" }, 
                    config, 
                    onScanSuccess, 
                    onScanFailure
                ).catch(err => {
                    console.error("Error al iniciar la cámara:", err);
                    alert("No se pudo acceder a la cámara. Asegúrate de dar los permisos necesarios y usar HTTPS.");
                    stopScanner();
                });
            }

            function stopScanner() {
                if (html5QrCode) {
                    html5QrCode.stop().then(() => {
                        readerContainer.classList.add("hidden");
                        btnScan.textContent = "Iniciar Cámara";
                        btnScan.style.backgroundColor = "#4f46e5"; // Vuelve al color azul original
                        isScanning = false;
                    }).catch(err => console.error("Error al detener la cámara:", err));
                }
            }

            function onScanSuccess(decodedText, decodedResult) {
                // Detiene el escáner inmediatamente tras leer un código
                stopScanner();

                // Obtiene la fecha y hora actual formateada localmente
                const ahora = new Date();
                const fechaHora = ahora.toLocaleString();

                // Almacena los datos en el historial
                saveToHistory(fechaHora, decodedText);
            }

            function onScanFailure(error) {
                // Este callback se ejecuta continuamente mientras busca un QR.
                // Se deja vacío para evitar ralentizar el dispositivo con logs constantes.
            }

            function saveToHistory(fecha, texto) {
                let historial = JSON.parse(localStorage.getItem("qr_history")) || [];
                
                // Inserta el nuevo registro al principio de la lista
                historial.unshift({ fecha, texto });
                
                localStorage.setItem("qr_history", JSON.stringify(historial));
                renderTable(historial);
            }

            function loadHistory() {
                let historial = JSON.parse(localStorage.getItem("qr_history")) || [];
                renderTable(historial);
            }

            function renderTable(historial) {
                historyBody.innerHTML = "";
                
                if (historial.length === 0) {
                    historyBody.innerHTML = `<tr><td colspan="2" style="text-align:center; color:#9ca3af;">No hay registros aún</td></tr>`;
                    return;
                }

                historial.forEach(item => {
                    const row = document.createElement("tr");
                    
                    const cellFecha = document.createElement("td");
                    cellFecha.textContent = item.fecha;
                    
                    const cellTexto = document.createElement("td");
                    // Si el texto detectado es un enlace web, lo vuelve clickeable automáticamente
                    if (item.texto.startsWith("http://") || item.texto.startsWith("https://")) {
                        const link = document.createElement("a");
                        link.href = item.texto;
                        link.target = "_blank";
                        link.textContent = item.texto;
                        cellTexto.appendChild(link);
                    } else {
                        cellTexto.textContent = item.texto;
                    }
                    
                    row.appendChild(cellFecha);
                    row.appendChild(cellTexto);
                    historyBody.appendChild(row);
                });
            }
        });
    </script>
</body>
</html>
