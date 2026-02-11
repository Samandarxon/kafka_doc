```markdown
# Apache Kafka O'rnatish va Systemd orqali Boshqarish Qo'llanmasi

## 1. Talablar
- Ubuntu 20.04 / 22.04 yoki boshqa Linux distributiv
- Java 17+ o'rnatilgan bo'lishi kerak
- Administrator (sudo) huquqi

## 2. Kafka yuklab olish
1. Rasmiy sayt orqali Kafka’ni yuklab oling:
```bash
wget https://downloads.apache.org/kafka/4.1.1/kafka_2.13-4.1.1.tgz
```
2. Arxivni chiqarish:
```bash
tar -xzf kafka_2.13-4.1.1.tgz -C /opt
```
3. Kafka papkasini oson boshqarish uchun nomini o'zgartiring:
```bash
sudo mv /opt/kafka_2.13-4.1.1 /opt/kafka
```

## 3. Kafka konfiguratsiyasi
1. Kafka konfiguratsiya fayllari `/opt/kafka/config/` papkada joylashgan.
2. `server.properties` faylini kerak bo'lsa tahrir qiling, masalan:
```properties
broker.id=1
listeners=PLAINTEXT://:9092,CONTROLLER://:9093
dataDir=/opt/kafka/data
num.partitions=1
default.replication.factor=1
controller.quorum.voters=1@localhost:9093
```
3. Kafka uchun ma'lumotlar papkasini yarating:
```bash
sudo mkdir -p /opt/kafka/data
sudo chown -R $USER:$USER /opt/kafka/data
```

## 4. KRaft mode uchun metadata formatlash
1. Avvalo UUID yaratish:
```bash
/opt/kafka/bin/kafka-storage.sh random-uuid
```
Bu sizga UUID beradi, masalan: `vLV-b0sgSZG6Tx2sFe26sQ`

2. Metadata papkasini formatlash:
```bash
/opt/kafka/bin/kafka-storage.sh format -t vLV-b0sgSZG6Tx2sFe26sQ -c /opt/kafka/config/server.properties
```

## 5. Kafka serverini ishga tushurish
```bash
/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/server.properties
```
- Agar server muvaffaqiyatli ishlasa, logda `Started NetworkTrafficServer` kabi xabarlarni ko'rasiz.
- Agar xatolik bo'lsa (`Address already in use`), 9092 va 9093 portlarini tekshiring va boshqa jarayonlar ishlayotgan bo'lsa to'xtating:
```bash
sudo lsof -i :9092
sudo lsof -i :9093
sudo kill -9 <PID>
```

## 6. Kafka’ni systemd xizmatiga qo'shish
1. Yangi systemd unit faylini yaratish:
```bash
sudo nano /etc/systemd/system/kafka.service
```
Faylga quyidagilarni yozing:
```ini
[Unit]
Description=Apache Kafka Server
After=network.target

[Service]
Type=simple
User=$USER
ExecStart=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/server.properties
ExecStop=/opt/kafka/bin/kafka-server-stop.sh
Restart=on-abnormal

[Install]
WantedBy=multi-user.target
```
2. systemd’ni yangilang:
```bash
sudo systemctl daemon-reload
```
3. Kafka xizmatini ishga tushiring:
```bash
sudo systemctl start kafka
```
4. Kafka xizmatini avtomatik ishga tushirishga sozlang:
```bash
sudo systemctl enable kafka
```
5. Kafka holatini tekshirish:
```bash
sudo systemctl status kafka
```
- Agar `active (running)` bo'lsa, Kafka muvaffaqiyatli ishga tushgan.

## 7. Kafka bilan ishlashni boshlash
1. Topic yaratish:
```bash
/opt/kafka/bin/kafka-topics.sh --create --topic test-topic --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1
```
2. Xabar yuborish:
```bash
/opt/kafka/bin/kafka-console-producer.sh --topic test-topic --bootstrap-server localhost:9092
```
3. Xabarlarni o'qish:
```bash
/opt/kafka/bin/kafka-console-consumer.sh --topic test-topic --from-beginning --bootstrap-server localhost:9092
```

---
Ushbu qo'llanma orqali siz Kafka’ni o’rnatib, KRaft mode’da ishga tushirishingiz, systemd orqali boshqarishingiz va test topic yaratib xabarlar yuborishingiz mumkin.
```

