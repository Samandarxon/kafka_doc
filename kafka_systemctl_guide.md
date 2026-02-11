# Apache Kafka Ubuntu-da o'rnatish va systemctl bilan boshqarish qo'llanmasi

## 1️⃣ Tayyorlik
- Ubuntu 20.04/22.04 yoki undan yangi.
- Java 17+ (OpenJDK) o'rnatilgan bo'lishi kerak.

Java tekshirish:
```bash
java -version
```

Agar Java yo'q bo'lsa, o'rnatish:
```bash
sudo apt update
sudo apt install openjdk-17-jdk -y
```

---

## 2️⃣ Kafka yuklab olish
```bash
cd /opt
sudo wget https://dlcdn.apache.org/kafka/3.5.1/kafka_2.13-3.5.1.tgz
sudo tar -xzf kafka_2.13-3.5.1.tgz
sudo mv kafka_2.13-3.5.1 kafka
sudo useradd -r -m -d /opt/kafka kafka
sudo chown -R kafka:kafka /opt/kafka
```

---

## 3️⃣ Kafka konfiguratsiyasi
```bash
cd /opt/kafka/config
nano server.properties
```
Minimal o'zgartirishlar:
```
broker.id=1
listeners=PLAINTEXT://:9092
log.dirs=/opt/kafka/data
num.network.threads=3
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
```

Data katalogi yaratish:
```bash
sudo mkdir -p /opt/kafka/data
sudo chown -R kafka:kafka /opt/kafka/data
```

---

## 4️⃣ Systemd unit faylini yaratish
```bash
sudo nano /etc/systemd/system/kafka.service
```
Fayl ichiga yozish:
```ini
[Unit]
Description=Apache Kafka Server
After=network.target zookeeper.service
Requires=zookeeper.service

[Service]
Type=simple
User=kafka
Group=kafka
Environment="KAFKA_HEAP_OPTS=-Xmx1G -Xms1G"
ExecStart=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/server.properties
ExecStop=/opt/kafka/bin/kafka-server-stop.sh
Restart=on-failure
RestartSec=10
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

---

## 5️⃣ Systemd’ni yangilash
```bash
sudo systemctl daemon-reload
```

---

## 6️⃣ Kafka’ni ishga tushirish
```bash
sudo systemctl start kafka
```

---

## 7️⃣ Avtomatik ishga tushirishga qo'yish
```bash
sudo systemctl enable kafka
```

---

## 8️⃣ Statusni tekshirish
```bash
sudo systemctl status kafka
```
Chiqariladigan natija:
```
● kafka.service - Apache Kafka Server
     Loaded: loaded (/etc/systemd/system/kafka.service; enabled)
     Active: active (running) since ...
     Main PID: 3800586 (java)
```
- `active (running)` → Kafka muvaffaqiyatli ishga tushgan.

---

## 9️⃣ Qo‘shimcha foydali buyruqlar
- Kafka’ni qayta ishga tushirish:
```bash
sudo systemctl restart kafka
```
- Kafka’ni to‘xtatish:
```bash
sudo systemctl stop kafka
```
- Logs’ni ko‘rish:
```bash
journalctl -u kafka -f
```

---

✅ Shu bilan Kafka Ubuntu tizimida noldan ishga tushirish va systemctl bilan boshqarish jarayoni tayyor.

