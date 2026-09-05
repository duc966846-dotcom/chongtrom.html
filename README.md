<!DOCTYPE html>
<html lang="vi">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>IoT Chống Trộm Xe Máy</title>
  <!-- Thư viện Tailwind CSS -->
  <script src="https://cdn.tailwindcss.com"></script>
  <!-- Thư viện MQTT.js -->
  <script src="https://unpkg.com/mqtt/dist/mqtt.min.js"></script>
</head>
<body class="bg-slate-100 font-sans min-h-screen pb-10">

  <!-- 1. Header -->
  <header class="bg-blue-600 text-white text-center py-4 shadow-md">
    <h1 class="text-xl md:text-2xl font-bold flex items-center justify-center gap-2">
      🚨 IoT CHỐNG TRỘM XE MÁY
    </h1>
    <p class="text-xs md:text-sm text-blue-100 mt-1">ESP32 • PIR • MPU6050 • Wi-Fi / MQTT</p>
  </header>

  <!-- Container chính -->
  <main class="max-w-2xl mx-auto px-4 mt-6 space-y-5">

    <!-- 2. Thẻ Trạng Thái Hệ Thống -->
    <section class="bg-white rounded-2xl p-6 shadow-sm border border-gray-100">
      <div class="flex justify-between items-start">
        <div>
          <p class="text-xs font-semibold text-gray-400 uppercase tracking-wider">TRẠNG THÁI HỆ THỐNG</p>
          <h2 id="systemStatus" class="text-2xl font-extrabold text-gray-800 mt-1">TẮT BẢO VỆ</h2>
        </div>
        <span id="badgeStatus" class="px-3 py-1 rounded-full text-xs font-bold bg-red-100 text-red-600">OFF</span>
      </div>

      <div class="flex gap-4 mt-6">
        <button id="btnBat" onclick="setProtection(true)" class="flex-1 bg-emerald-500 hover:bg-emerald-600 active:scale-95 transition text-white font-bold py-3 px-4 rounded-xl flex items-center justify-center gap-2 shadow-sm">
          <span class="w-3 h-3 rounded-full bg-white opacity-80"></span> BẬT BẢO VỆ
        </button>
        <button id="btnTat" onclick="setProtection(false)" class="flex-1 bg-red-500 hover:bg-red-600 active:scale-95 transition text-white font-bold py-3 px-4 rounded-xl flex items-center justify-center gap-2 shadow-sm">
          <span class="w-3 h-3 rounded-full bg-white opacity-80"></span> TẮT BẢO VỆ
        </button>
      </div>
    </section>

    <!-- 3. Thẻ Giám Sát Cảm Biến -->
    <section class="bg-white rounded-2xl p-6 shadow-sm border border-gray-100">
      <h3 class="text-base font-bold text-gray-800 flex items-center gap-2 mb-4">
        📡 Giám sát cảm biến
      </h3>

      <div class="grid grid-cols-1 sm:grid-cols-2 gap-4">
        <!-- Cảm biến PIR -->
        <div class="border border-gray-100 rounded-xl p-4 bg-gray-50/50">
          <p class="text-xs font-semibold text-gray-500">👤 PIR</p>
          <p id="pirStatus" class="text-base font-bold text-gray-800 mt-1">Không có chuyển động</p>
        </div>

        <!-- Cảm biến MPU6050 -->
        <div class="border border-gray-100 rounded-xl p-4 bg-gray-50/50">
          <p class="text-xs font-semibold text-gray-500">📐 MPU6050</p>
          <p id="mpuStatus" class="text-base font-bold text-gray-800 mt-1">Ổn định</p>
        </div>

        <!-- Đèn LED -->
        <div class="border border-gray-100 rounded-xl p-4 bg-gray-50/50">
          <p class="text-xs font-semibold text-gray-500">💡 LED</p>
          <p id="ledStatus" class="text-base font-bold text-gray-800 mt-1">TẮT</p>
        </div>

        <!-- Còi Buzzer -->
        <div class="border border-gray-100 rounded-xl p-4 bg-gray-50/50">
          <p class="text-xs font-semibold text-gray-500">🔊 Buzzer</p>
          <p id="buzzerStatus" class="text-base font-bold text-gray-800 mt-1">TẮT</p>
        </div>
      </div>
    </section>

    <!-- 4. Thẻ Kết Nối MQTT -->
    <section class="bg-white rounded-2xl p-6 shadow-sm border border-gray-100">
      <h3 class="text-base font-bold text-gray-800 flex items-center gap-2 mb-2">
        📊 Kết nối MQTT
      </h3>
      <p id="mqttConnInfo" class="text-xs text-gray-500 mb-3">
        Đang kết nối tới MQTT Broker...
      </p>
      <div class="text-xs text-gray-500 space-y-1 font-mono bg-gray-50 p-3 rounded-lg border border-gray-100">
        <p>Topic cảnh báo: <span class="text-gray-800 font-semibold">xe_may/canh_bao</span></p>
        <p>Topic trạng thái: <span class="text-gray-800 font-semibold">xe_may/status</span></p>
      </div>
    </section>

  </main>

  <!-- Script Xử Lý Logic & Mô Phỏng -->
  <script>
    let isProtected = false;
    let simInterval = null;
    let mqttClient = null;

    // Cấu hình kết nối MQTT Broker
    const brokerUrl = 'wss://broker.emqx.io:8084/mqtt';
    const topicStatus = 'xe_may/status';
    const topicCanhBao = 'xe_may/canh_bao';

    // Thử kết nối MQTT Broker công cộng
    try {
      mqttClient = mqtt.connect(brokerUrl);

      mqttClient.on('connect', () => {
        document.getElementById('mqttConnInfo').innerText = 'Đã kết nối MQTT Broker (broker.emqx.io)';
        document.getElementById('mqttConnInfo').classList.add('text-emerald-600', 'font-semibold');
        mqttClient.subscribe(topicStatus);
        mqttClient.subscribe(topicCanhBao);
      });

      mqttClient.on('error', () => {
        document.getElementById('mqttConnInfo').innerText = 'Chưa kết nối MQTT — giao diện mô phỏng cục bộ.';
      });

      // Nhận tin nhắn từ MQTT
      mqttClient.on('message', (topic, payload) => {
        const msg = payload.toString();
        if (topic === topicStatus) {
          if (msg === 'ON' && !isProtected) setProtection(true, false);
          if (msg === 'OFF' && isProtected) setProtection(false, false);
        }
      });
    } catch (e) {
      document.getElementById('mqttConnInfo').innerText = 'Chưa kết nối MQTT — giao diện mô phỏng cục bộ.';
    }

    // Bật/Tắt Chế Độ Bảo Vệ
    function setProtection(enable, publish = true) {
      isProtected = enable;

      const sysText = document.getElementById('systemStatus');
      const badge = document.getElementById('badgeStatus');

      if (isProtected) {
        sysText.innerText = 'ĐANG BẢO VỆ';
        sysText.className = 'text-2xl font-extrabold text-emerald-600 mt-1';
        badge.innerText = 'ON';
        badge.className = 'px-3 py-1 rounded-full text-xs font-bold bg-emerald-100 text-emerald-600';

        if (publish && mqttClient && mqttClient.connected) {
          mqttClient.publish(topicStatus, 'ON');
        }
        startSimulation();
      } else {
        sysText.innerText = 'TẮT BẢO VỆ';
        sysText.className = 'text-2xl font-extrabold text-gray-800 mt-1';
        badge.innerText = 'OFF';
        badge.className = 'px-3 py-1 rounded-full text-xs font-bold bg-red-100 text-red-600';

        if (publish && mqttClient && mqttClient.connected) {
          mqttClient.publish(topicStatus, 'OFF');
        }
        stopSimulation();
      }
    }

    // Bộ Giả Lập Cảm Biến
    function startSimulation() {
      clearInterval(simInterval);
      simInterval = setInterval(() => {
        if (!isProtected) return;

        const rand = Math.random();
        if (rand < 0.3) {
          triggerAlarm('PIR', 'Có chuyển động!');
        } else if (rand > 0.7) {
          triggerAlarm('MPU', 'Phát hiện rung lắc!');
        } else {
          resetSensors();
        }
      }, 4000);
    }

    function stopSimulation() {
      clearInterval(simInterval);
      resetSensors();
    }

    function triggerAlarm(source, message) {
      const pirText = document.getElementById('pirStatus');
      const mpuText = document.getElementById('mpuStatus');
      const ledText = document.getElementById('ledStatus');
      const buzzerText = document.getElementById('buzzerStatus');

      if (source === 'PIR') {
        pirText.innerText = message;
        pirText.className = 'text-base font-bold text-red-600 animate-pulse';
      } else {
        mpuText.innerText = message;
        mpuText.className = 'text-base font-bold text-red-600 animate-pulse';
      }

      ledText.innerText = 'BẬT (SÁNG ĐỎ)';
      ledText.className = 'text-base font-bold text-red-600';
      buzzerText.innerText = 'BẬT (REO)';
      buzzerText.className = 'text-base font-bold text-red-600';

      if (mqttClient && mqttClient.connected) {
        mqttClient.publish(topicCanhBao, JSON.stringify({ event: source, message: message }));
      }
    }

    function resetSensors() {
      document.getElementById('pirStatus').innerText = 'Không có chuyển động';
      document.getElementById('pirStatus').className = 'text-base font-bold text-gray-800';
      document.getElementById('mpuStatus').innerText = 'Ổn định';
      document.getElementById('mpuStatus').className = 'text-base font-bold text-gray-800';
      document.getElementById('ledStatus').innerText = 'TẮT';
      document.getElementById('ledStatus').className = 'text-base font-bold text-gray-800';
      document.getElementById('buzzerStatus').innerText = 'TẮT';
      document.getElementById('buzzerStatus').className = 'text-base font-bold text-gray-800';
    }
  </script>
</body>
</html>
