<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Arcade Ticket Scanner</title>
    <script src="https://cdn.jsdelivr.net/npm/@ericblade/quagga2/dist/quagga.min.js"></script>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }

        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
            padding: 20px;
        }

        .container {
            max-width: 600px;
            margin: 0 auto;
            background: white;
            border-radius: 20px;
            box-shadow: 0 20px 60px rgba(0,0,0,0.3);
            overflow: hidden;
        }

        .header {
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            padding: 30px;
            text-align: center;
        }

        h1 {
            font-size: 28px;
            margin-bottom: 10px;
        }

        .total-display {
            background: rgba(255,255,255,0.2);
            border-radius: 15px;
            padding: 20px;
            margin-top: 20px;
        }

        .total-label {
            font-size: 14px;
            opacity: 0.9;
            margin-bottom: 5px;
        }

        .total-value {
            font-size: 48px;
            font-weight: bold;
        }

        .scanner-section {
            padding: 30px;
        }

        #video-container {
            position: relative;
            width: 100%;
            max-width: 100%;
            aspect-ratio: 4/3;
            background: #000;
            border-radius: 15px;
            overflow: hidden;
            margin-bottom: 20px;
        }

        #video {
            width: 100%;
            height: 100%;
            object-fit: cover;
        }

        .scan-line {
            position: absolute;
            top: 50%;
            left: 10%;
            right: 10%;
            height: 2px;
            background: #00ff00;
            box-shadow: 0 0 10px #00ff00;
            animation: scan 2s ease-in-out infinite;
        }

        @keyframes scan {
            0%, 100% { transform: translateY(-50px); opacity: 0.5; }
            50% { transform: translateY(50px); opacity: 1; }
        }

        .status {
            text-align: center;
            padding: 15px;
            border-radius: 10px;
            margin-bottom: 20px;
            font-weight: 500;
        }

        .status.scanning {
            background: #e3f2fd;
            color: #1976d2;
        }

        .status.success {
            background: #e8f5e9;
            color: #388e3c;
        }

        .status.error {
            background: #ffebee;
            color: #d32f2f;
        }

        .button {
            width: 100%;
            padding: 15px;
            border: none;
            border-radius: 10px;
            font-size: 16px;
            font-weight: 600;
            cursor: pointer;
            transition: all 0.3s;
            margin-bottom: 10px;
        }

        .button-primary {
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
        }

        .button-primary:hover {
            transform: translateY(-2px);
            box-shadow: 0 5px 15px rgba(102, 126, 234, 0.4);
        }

        .button-secondary {
            background: #f5f5f5;
            color: #333;
        }

        .button-secondary:hover {
            background: #e0e0e0;
        }

        .history {
            margin-top: 30px;
            padding-top: 20px;
            border-top: 2px solid #f0f0f0;
        }

        .history-title {
            font-size: 18px;
            font-weight: 600;
            margin-bottom: 15px;
            color: #333;
        }

        .history-item {
            display: flex;
            justify-content: space-between;
            padding: 12px;
            background: #f9f9f9;
            border-radius: 8px;
            margin-bottom: 8px;
            animation: slideIn 0.3s ease;
        }

        @keyframes slideIn {
            from {
                opacity: 0;
                transform: translateX(-20px);
            }
            to {
                opacity: 1;
                transform: translateX(0);
            }
        }

        .history-tickets {
            font-weight: 600;
            color: #667eea;
            font-size: 18px;
        }

        .history-time {
            color: #666;
            font-size: 12px;
        }

        .no-history {
            text-align: center;
            color: #999;
            padding: 20px;
            font-style: italic;
        }

        .hidden {
            display: none;
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>🎟️ Arcade Ticket Scanner</h1>
            <div class="total-display">
                <div class="total-label">Total Tickets</div>
                <div class="total-value" id="totalTickets">0</div>
            </div>
        </div>

        <div class="scanner-section">
            <div id="status" class="status scanning hidden">
                Ready to scan...
            </div>

            <div id="video-container">
                <video id="video" playsinline></video>
                <div class="scan-line"></div>
            </div>

            <button id="startButton" class="button button-primary">Start Scanning</button>
            <button id="stopButton" class="button button-secondary hidden">Stop Scanning</button>
            <button id="resetButton" class="button button-secondary">Reset Total</button>

            <div class="history">
                <div class="history-title">Scan History</div>
                <div id="historyList">
                    <div class="no-history">No scans yet</div>
                </div>
            </div>
        </div>
    </div>

    <script>
        let isScanning = false;
        let totalTickets = 0;
        let scanHistory = [];
        let lastScannedBarcode = null;
        let lastScanTime = 0;

        const video = document.getElementById('video');
        const startButton = document.getElementById('startButton');
        const stopButton = document.getElementById('stopButton');
        const resetButton = document.getElementById('resetButton');
        const statusDiv = document.getElementById('status');
        const totalTicketsDiv = document.getElementById('totalTickets');
        const historyList = document.getElementById('historyList');

        function showStatus(message, type) {
            statusDiv.textContent = message;
            statusDiv.className = `status ${type}`;
            statusDiv.classList.remove('hidden');
        }

        function extractTicketCount(barcode) {
            // Format: 7234-XXXXXXXX-0XXX-X
            // Extract the ticket count from positions 12-15 (the 0XXX part)
            if (barcode.length >= 16 && barcode.startsWith('7234')) {
                const ticketPart = barcode.substring(12, 16);
                // Remove the leading 0 and the check digit at the end
                const tickets = parseInt(ticketPart.substring(1, 4));
                return tickets;
            }
            return null;
        }

        function updateTotal(tickets) {
            totalTickets += tickets;
            totalTicketsDiv.textContent = totalTickets.toLocaleString();
            
            // Add to history
            const scan = {
                tickets: tickets,
                time: new Date().toLocaleTimeString(),
                barcode: lastScannedBarcode
            };
            scanHistory.unshift(scan);
            
            updateHistoryDisplay();
        }

        function updateHistoryDisplay() {
            if (scanHistory.length === 0) {
                historyList.innerHTML = '<div class="no-history">No scans yet</div>';
                return;
            }

            historyList.innerHTML = scanHistory.map(scan => `
                <div class="history-item">
                    <div>
                        <div class="history-tickets">+${scan.tickets} tickets</div>
                        <div class="history-time">${scan.time}</div>
                    </div>
                    <div style="text-align: right; font-size: 11px; color: #999;">
                        ${scan.barcode}
                    </div>
                </div>
            `).join('');
        }

        async function startScanning() {
            try {
                showStatus('Initializing camera...', 'scanning');
                isScanning = true;
                startButton.classList.add('hidden');
                stopButton.classList.remove('hidden');

                Quagga.init({
                    inputStream: {
                        name: "Live",
                        type: "LiveStream",
                        target: document.querySelector('#video-container'),
                        constraints: {
                            facingMode: "environment", // Use back camera
                            aspectRatio: { min: 1, max: 2 }
                        }
                    },
                    decoder: {
                        readers: ["code_128_reader", "ean_reader", "ean_8_reader", "code_39_reader", "upc_reader"]
                    },
                    locate: true
                }, function(err) {
                    if (err) {
                        console.error(err);
                        showStatus('Error starting camera: ' + err.message, 'error');
                        stopScanning();
                        return;
                    }
                    console.log("Initialization finished. Ready to start");
                    Quagga.start();
                    showStatus('Scanning... Point camera at barcode', 'scanning');
                });

                Quagga.onDetected(function(result) {
                    const barcode = result.codeResult.code;
                    const now = Date.now();
                    
                    // Prevent duplicate scans (ignore same barcode within 2 seconds)
                    if (barcode === lastScannedBarcode && (now - lastScanTime) < 2000) {
                        return;
                    }

                    const tickets = extractTicketCount(barcode);
                    
                    if (tickets !== null) {
                        lastScannedBarcode = barcode;
                        lastScanTime = now;
                        
                        showStatus(`✓ Scanned: ${tickets} tickets!`, 'success');
                        updateTotal(tickets);
                        
                        // Play success feedback (vibrate if available)
                        if (navigator.vibrate) {
                            navigator.vibrate(200);
                        }

                        // Reset status after 2 seconds
                        setTimeout(() => {
                            if (isScanning) {
                                showStatus('Scanning... Point camera at barcode', 'scanning');
                            }
                        }, 2000);
                    } else {
                        // Don't show error for every non-matching barcode, as it might scan other things
                        console.log('Scanned barcode not in expected format:', barcode);
                    }
                });

            } catch (err) {
                console.error(err);
                showStatus('Error starting camera: ' + err.message, 'error');
                stopScanning();
            }
        }

        function stopScanning() {
            if (isScanning) {
                Quagga.stop();
                isScanning = false;
                startButton.classList.remove('hidden');
                stopButton.classList.add('hidden');
                statusDiv.classList.add('hidden');
            }
        }

        function resetTotal() {
            if (confirm('Are you sure you want to reset the total and clear history?')) {
                totalTickets = 0;
                scanHistory = [];
                lastScannedBarcode = null;
                lastScanTime = 0;
                totalTicketsDiv.textContent = '0';
                updateHistoryDisplay();
                showStatus('Total reset!', 'success');
                setTimeout(() => {
                    statusDiv.classList.add('hidden');
                }, 2000);
            }
        }

        startButton.addEventListener('click', startScanning);
        stopButton.addEventListener('click', stopScanning);
        resetButton.addEventListener('click', resetTotal);

        // Load saved data on page load
        window.addEventListener('load', () => {
            const saved = localStorage.getItem('arcadeTicketData');
            if (saved) {
                const data = JSON.parse(saved);
                totalTickets = data.total || 0;
                scanHistory = data.history || [];
                totalTicketsDiv.textContent = totalTickets.toLocaleString();
                updateHistoryDisplay();
            }
        });

        // Save data before page unload
        window.addEventListener('beforeunload', () => {
            localStorage.setItem('arcadeTicketData', JSON.stringify({
                total: totalTickets,
                history: scanHistory
            }));
        });
    </script>
</body>
</html>
