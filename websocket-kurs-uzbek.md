# WebSocket + Golang — Noldan Professional Darajagacha

## To'liq Kurs O'zbek Tilida

> **Talablar:** Go, goroutine, channel, REST API, Redis, Docker bilimi bor deb faraz qilinadi.
> Barcha kodlar production darajasida, to'liq ishlaydigan holda yozilgan.

---

## CURRICULUM (O'quv Rejasi)

```
PHASE 1 — Asos (Foundation)
  1-Dars: WebSocket protokoli — HTTP dan farqi, handshake, frame tuzilishi
  2-Dars: Eng oddiy echo server (gorilla/websocket)
  3-Dars: JSON xabarlar va xabar turlari

PHASE 2 — Ko'p Clientlar Bilan Ishlash
  4-Dars: Multiple clients — broadcast pattern
  5-Dars: Hub pattern — markaziy boshqaruv
  6-Dars: Read Pump / Write Pump pattern

PHASE 3 — Room System
  7-Dars: Room (xona) arxitekturasi
  8-Dars: Room ga qo'shilish/chiqish, xona ichida xabar almashish
  9-Dars: Private messaging (shaxsiy xabar)

PHASE 4 — Authentication va Xavfsizlik
  10-Dars: JWT bilan WebSocket authentication
  11-Dars: Origin tekshirish, rate limiting, xavfsizlik

PHASE 5 — Production Muammolari
  12-Dars: Heartbeat (Ping/Pong), dead connection detection
  13-Dars: Slow client muammosi va yechimi
  14-Dars: Goroutine leak, memory leak oldini olish
  15-Dars: Client reconnect strategiyasi

PHASE 6 — Scaling (Ko'p Server)
  16-Dars: Redis Pub/Sub bilan horizontal scaling
  17-Dars: Multi-server arxitektura

PHASE 7 — Real Loyiha
  18-Dars: Real-time Chat Service — to'liq loyiha
  19-Dars: Real-time Notification Service
  20-Dars: Production deployment va monitoring
```

---

# PHASE 1 — Asos (Foundation)

---

## 1-Dars: WebSocket Protokoli

### HTTP ning Cheklovi

REST API da client va server **request-response** modelida ishlaydi:

```
Client                     Server
  │── GET /messages ──────→│
  │←── 200 OK + data ──────│
  │                         │
  │  (5 soniya kutish)      │
  │                         │
  │── GET /messages ──────→│
  │←── 200 OK + data ──────│
```

**Muammolar:**
- Yangi xabar bormi bilish uchun **har safar so'rov** yuborish kerak (polling)
- Har bir so'rovda HTTP headerlar qayta yuboriladi (~500-1000 bayt ortiqcha)
- Real-time emas — kechikish bor (polling intervali)

**Polling turlari:**

```
1. Short Polling (Qisqa so'rov):
   Har 3 soniyada: GET /messages → "yangilik yo'q" → 3s → GET → ...
   Muammo: Serverga ortiqcha yuk, kechikish

2. Long Polling (Uzoq so'rov):
   GET /messages → Server KUTADI (yangilik bo'lguncha) → javob → yangi GET
   Muammo: Har safar yangi HTTP connection, timeout boshqaruvi murakkab

3. SSE (Server-Sent Events):
   GET /stream → Server bir tomonlama xabar yuboradi
   Muammo: Faqat serverdan clientga (bir tomonlama)
```

### WebSocket — Ikki Tomonlama Aloqa

```
Client                     Server
  │── HTTP Upgrade ───────→│  (1 marta handshake)
  │←── 101 Switching ──────│
  │                         │
  │◄════ DOIMIY ALOQA ════►│  (full-duplex)
  │                         │
  │── "salom" ────────────→│
  │←── "javob" ────────────│
  │←── "yangilik!" ────────│  (server o'zi yuboradi)
  │── "rahmat" ───────────→│
  │                         │
  │── Close ──────────────→│  (ulanish yopiladi)
  │←── Close ──────────────│
```

**WebSocket afzalliklari:**

| Xususiyat | HTTP Polling | WebSocket |
|-----------|-------------|-----------|
| Ulanish | Har safar yangi | Bitta doimiy |
| Yo'nalish | Client → Server | Ikki tomonlama |
| Header overhead | Har safar ~1KB | Faqat 2-14 bayt (frame) |
| Real-time | Yo'q (kechikish) | Ha (millisekundlar) |
| Server push | Mumkin emas | Mumkin |

### WebSocket Handshake

WebSocket ulanishi **HTTP Upgrade** so'rovi bilan boshlanadi:

**Client so'rovi:**
```http
GET /ws HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Origin: http://example.com
```

**Server javobi:**
```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

**Jarayon:**

```
1. Client HTTP GET yuboradi (Upgrade: websocket header bilan)
2. Server 101 Switching Protocols javob beradi
3. TCP ulanish WebSocket protokoliga o'tadi
4. Endi ikki tomon ham erkin xabar almashadi
5. Close frame yuborilganda ulanish yopiladi
```

### WebSocket Frame Tuzilishi

Handshake dan keyin ma'lumotlar **frame** larda uzatiladi:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |           (16/64)             |
|N|V|V|V|       |S|             |   (if payload len==126/127)   |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+-------------------------------+
|                     Masking key (0 or 4 bytes)                |
+-------------------------------+-------------------------------+
|                     Payload Data                              |
+---------------------------------------------------------------+
```

**Muhim maydonlar:**
- **FIN** — oxirgi frame belgisi
- **Opcode** — xabar turi:
  - `0x1` — Text frame
  - `0x2` — Binary frame
  - `0x8` — Close
  - `0x9` — Ping
  - `0xA` — Pong
- **MASK** — client dan server ga yuborilgan frame lar doimo masklangan
- **Payload** — haqiqiy ma'lumot

### Qachon WebSocket Ishlatish Kerak?

**WebSocket KERAK:**
- Chat ilovalar
- Real-time notification
- Online o'yinlar
- Jonli narx kotirovkalari (stock ticker)
- Collaborative editing (Google Docs)
- IoT qurilmalar monitoringi

**WebSocket KERAK EMAS:**
- Oddiy CRUD operatsiyalar
- Fayllarni yuklab olish
- Bir martalik so'rov-javob
- Kamdan-kam yangilanadigan ma'lumotlar

---

## 2-Dars: Eng Oddiy Echo Server

### gorilla/websocket O'rnatish

```bash
mkdir -p ~/websocket-course/01-echo
cd ~/websocket-course/01-echo
go mod init echo-server
go get github.com/gorilla/websocket
```

### Echo Server Kodi

```go
// ~/websocket-course/01-echo/main.go
package main

import (
	"log"
	"net/http"

	"github.com/gorilla/websocket"
)

// upgrader — HTTP ulanishni WebSocket ga o'tkazadi
var upgrader = websocket.Upgrader{
	ReadBufferSize:  1024,
	WriteBufferSize: 1024,
	// CheckOrigin — production da bu funksiyani to'g'ri sozlang!
	// Hozir o'rganish uchun barcha origin larni ruxsat qilamiz
	CheckOrigin: func(r *http.Request) bool {
		return true
	},
}

func handleWebSocket(w http.ResponseWriter, r *http.Request) {
	// 1. HTTP → WebSocket upgrade
	conn, err := upgrader.Upgrade(w, r, nil)
	if err != nil {
		log.Printf("Upgrade xatosi: %v", err)
		return
	}
	defer conn.Close()

	log.Printf("Yangi client ulandi: %s", conn.RemoteAddr())

	// 2. Xabarlarni o'qish loop
	for {
		// messageType: TextMessage (1) yoki BinaryMessage (2)
		messageType, message, err := conn.ReadMessage()
		if err != nil {
			if websocket.IsUnexpectedCloseError(err,
				websocket.CloseGoingAway,
				websocket.CloseNormalClosure) {
				log.Printf("O'qish xatosi: %v", err)
			}
			log.Printf("Client uzildi: %s", conn.RemoteAddr())
			break
		}

		log.Printf("Qabul qilindi [%s]: %s", conn.RemoteAddr(), message)

		// 3. Xabarni qaytarish (echo)
		if err := conn.WriteMessage(messageType, message); err != nil {
			log.Printf("Yozish xatosi: %v", err)
			break
		}
	}
}

func main() {
	http.HandleFunc("/ws", handleWebSocket)

	log.Println("Echo server ishga tushdi: ws://localhost:8080/ws")
	if err := http.ListenAndServe(":8080", nil); err != nil {
		log.Fatal("Server xatosi:", err)
	}
}
```

### Test Uchun HTML Client

```go
// ~/websocket-course/01-echo/main.go ga qo'shing (main funksiyasidan oldin)

const htmlPage = `<!DOCTYPE html>
<html>
<head><title>WebSocket Echo Test</title></head>
<body>
<h2>WebSocket Echo Test</h2>
<div>
    <input type="text" id="msg" placeholder="Xabar yozing..." size="40">
    <button onclick="send()">Yuborish</button>
</div>
<pre id="log"></pre>
<script>
    var ws = new WebSocket("ws://localhost:8080/ws");
    var logEl = document.getElementById("log");

    ws.onopen = function() {
        logEl.textContent += ">>> Ulandi\n";
    };
    ws.onmessage = function(e) {
        logEl.textContent += "<<< " + e.data + "\n";
    };
    ws.onclose = function() {
        logEl.textContent += ">>> Uzildi\n";
    };
    function send() {
        var input = document.getElementById("msg");
        ws.send(input.value);
        logEl.textContent += ">>> " + input.value + "\n";
        input.value = "";
    }
</script>
</body>
</html>`
```

```go
// main() funksiyasida /ws dan oldin qo'shing:
func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "text/html")
		w.Write([]byte(htmlPage))
	})
	http.HandleFunc("/ws", handleWebSocket)
	// ...
}
```

### Ishga Tushirish va Test

```bash
go run main.go

# Brauzerda oching: http://localhost:8080
# Yoki wscat bilan test:
# npm install -g wscat
# wscat -c ws://localhost:8080/ws
```

### Nima Uchun Shunday Yozilgan?

| Element | Sabab |
|---------|-------|
| `upgrader` global | Har safar yangi yaratish shart emas, thread-safe |
| `defer conn.Close()` | Funksiya qanday tugamasin, ulanish yopiladi |
| `IsUnexpectedCloseError` | Oddiy uzilishni xatodan ajratish |
| `for` loop | Doimiy xabar o'qish — WebSocket sessiyasi uzluksiz |
| `ReadBufferSize/WriteBufferSize` | Xotira boshqaruvi — har bir client uchun 1KB+1KB |

### Keng Tarqalgan Xatolar

1. **`CheckOrigin` ni e'tiborsiz qoldirish** — Standartda gorilla/websocket boshqa origin larni rad etadi. Production da faqat ruxsat etilgan domainlarni yozing.
2. **`conn.Close()` ni unutish** — Goroutine va file descriptor leak bo'ladi.
3. **Concurrent write** — Bitta WebSocket ulanishga bir vaqtda ikki goroutine dan yozish MUMKIN EMAS. Bu race condition va panic ga olib keladi.

---

## 3-Dars: JSON Xabarlar va Xabar Turlari

### Xabar Protokoli

Real ilovada xabarlar oddiy matn emas, tuzilmali bo'ladi:

```go
// ~/websocket-course/02-json/messages.go
package main

import "time"

// MessageType — xabar turlari
type MessageType string

const (
	MsgTypeChat         MessageType = "chat"
	MsgTypeJoin         MessageType = "join"
	MsgTypeLeave        MessageType = "leave"
	MsgTypeTyping       MessageType = "typing"
	MsgTypeError        MessageType = "error"
	MsgTypeAck          MessageType = "ack"
	MsgTypeUserList     MessageType = "user_list"
)

// IncomingMessage — clientdan kelgan xabar
type IncomingMessage struct {
	Type    MessageType `json:"type"`
	Content string      `json:"content,omitempty"`
	To      string      `json:"to,omitempty"`       // private message uchun
	Room    string      `json:"room,omitempty"`
}

// OutgoingMessage — serverdan clientga yuborilgan xabar
type OutgoingMessage struct {
	Type      MessageType `json:"type"`
	Content   string      `json:"content,omitempty"`
	From      string      `json:"from,omitempty"`
	Room      string      `json:"room,omitempty"`
	Timestamp time.Time   `json:"timestamp"`
	Users     []string    `json:"users,omitempty"` // user_list uchun
}
```

### JSON WebSocket Server

```go
// ~/websocket-course/02-json/main.go
package main

import (
	"encoding/json"
	"log"
	"net/http"
	"time"

	"github.com/gorilla/websocket"
)

var upgrader = websocket.Upgrader{
	ReadBufferSize:  1024,
	WriteBufferSize: 1024,
	CheckOrigin:     func(r *http.Request) bool { return true },
}

func handleWebSocket(w http.ResponseWriter, r *http.Request) {
	conn, err := upgrader.Upgrade(w, r, nil)
	if err != nil {
		log.Printf("Upgrade xatosi: %v", err)
		return
	}
	defer conn.Close()

	// Username ni query param dan olish
	username := r.URL.Query().Get("username")
	if username == "" {
		username = "anonim"
	}

	log.Printf("[%s] ulandi", username)

	for {
		// JSON xabarni o'qish
		var msg IncomingMessage
		if err := conn.ReadJSON(&msg); err != nil {
			if websocket.IsUnexpectedCloseError(err,
				websocket.CloseGoingAway,
				websocket.CloseNormalClosure) {
				log.Printf("ReadJSON xatosi: %v", err)
			}
			break
		}

		log.Printf("[%s] %s: %s", username, msg.Type, msg.Content)

		// Xabar turiga qarab javob berish
		var response OutgoingMessage

		switch msg.Type {
		case MsgTypeChat:
			response = OutgoingMessage{
				Type:      MsgTypeChat,
				Content:   msg.Content,
				From:      username,
				Timestamp: time.Now(),
			}

		case MsgTypeTyping:
			response = OutgoingMessage{
				Type:      MsgTypeTyping,
				From:      username,
				Timestamp: time.Now(),
			}

		default:
			response = OutgoingMessage{
				Type:      MsgTypeError,
				Content:   "Noma'lum xabar turi: " + string(msg.Type),
				Timestamp: time.Now(),
			}
		}

		// JSON javob yuborish
		if err := conn.WriteJSON(response); err != nil {
			log.Printf("WriteJSON xatosi: %v", err)
			break
		}
	}

	log.Printf("[%s] uzildi", username)
}

func main() {
	http.HandleFunc("/ws", handleWebSocket)
	log.Println("JSON server: ws://localhost:8080/ws?username=Ali")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### ReadJSON/WriteJSON vs ReadMessage/WriteMessage

```go
// Usul 1: ReadJSON/WriteJSON (qulay)
var msg IncomingMessage
conn.ReadJSON(&msg)    // Avtomatik JSON unmarshal
conn.WriteJSON(resp)   // Avtomatik JSON marshal

// Usul 2: ReadMessage/WriteMessage (ko'proq kontrol)
_, raw, _ := conn.ReadMessage()
json.Unmarshal(raw, &msg)

data, _ := json.Marshal(resp)
conn.WriteMessage(websocket.TextMessage, data)

// Usul 3: NextWriter (eng samarali — katta xabarlar uchun)
w, _ := conn.NextWriter(websocket.TextMessage)
json.NewEncoder(w).Encode(resp)  // Oraliq buffer yo'q
w.Close()
```

**Tavsiya:** Oddiy holatlar uchun `ReadJSON/WriteJSON`. Katta xabarlar yoki streaming uchun `NextWriter`.

### Xabar Validatsiyasi

```go
// Xabar hajmi cheklash
conn.SetReadLimit(4096) // Max 4KB xabar

// Validatsiya funksiyasi
func validateMessage(msg *IncomingMessage) error {
	if msg.Type == "" {
		return fmt.Errorf("xabar turi ko'rsatilmagan")
	}
	if msg.Type == MsgTypeChat && msg.Content == "" {
		return fmt.Errorf("chat xabarida matn bo'lishi kerak")
	}
	if len(msg.Content) > 2000 {
		return fmt.Errorf("xabar juda uzun (max 2000 belgi)")
	}
	return nil
}
```

---

# PHASE 2 — Ko'p Clientlar Bilan Ishlash

---

## 4-Dars: Multiple Clients — Broadcast Pattern

### Muammo

Echo serverda har bir client faqat o'ziga javob oladi. Real chat da xabar **barcha** clientlarga yuborilishi kerak.

```
Hozirgi holat:              Kerakli holat:
Client A → Server → A       Client A → Server → A, B, C
Client B → Server → B       Client B → Server → A, B, C
Client C → Server → C       Client C → Server → A, B, C
```

### Oddiy Yondashuv (Noto'g'ri!)

```go
// XATO YONDASHUV — Race condition!
var clients = make(map[*websocket.Conn]bool)

func handleWS(w http.ResponseWriter, r *http.Request) {
    conn, _ := upgrader.Upgrade(w, r, nil)
    clients[conn] = true // Race condition!

    for {
        _, msg, _ := conn.ReadMessage()
        for client := range clients { // Race condition!
            client.WriteMessage(1, msg) // Concurrent write!
        }
    }
}
```

**Muammolar:**
1. `map` Go da thread-safe emas — bir nechta goroutine bir vaqtda yozsa panic
2. `conn.WriteMessage` concurrent chaqirilsa crash

### To'g'ri Yondashuv — sync.Mutex Bilan

```go
// ~/websocket-course/03-broadcast/main.go
package main

import (
	"encoding/json"
	"log"
	"net/http"
	"sync"
	"time"

	"github.com/gorilla/websocket"
)

type MessageType string

const (
	MsgChat  MessageType = "chat"
	MsgJoin  MessageType = "join"
	MsgLeave MessageType = "leave"
)

type IncomingMessage struct {
	Type    MessageType `json:"type"`
	Content string      `json:"content"`
}

type OutgoingMessage struct {
	Type      MessageType `json:"type"`
	Content   string      `json:"content,omitempty"`
	From      string      `json:"from"`
	Timestamp time.Time   `json:"timestamp"`
}

// Client — bitta WebSocket ulanish
type Client struct {
	conn     *websocket.Conn
	username string
	mu       sync.Mutex // Yozish uchun lock
}

// Send — client ga xabar yuborish (thread-safe)
func (c *Client) Send(msg OutgoingMessage) error {
	c.mu.Lock()
	defer c.mu.Unlock()
	return c.conn.WriteJSON(msg)
}

// ClientManager — barcha clientlarni boshqarish
type ClientManager struct {
	clients map[*Client]bool
	mu      sync.RWMutex
}

func NewClientManager() *ClientManager {
	return &ClientManager{
		clients: make(map[*Client]bool),
	}
}

func (cm *ClientManager) Add(client *Client) {
	cm.mu.Lock()
	defer cm.mu.Unlock()
	cm.clients[client] = true
}

func (cm *ClientManager) Remove(client *Client) {
	cm.mu.Lock()
	defer cm.mu.Unlock()
	delete(cm.clients, client)
	client.conn.Close()
}

// Broadcast — barcha clientlarga xabar yuborish
func (cm *ClientManager) Broadcast(msg OutgoingMessage, exclude *Client) {
	cm.mu.RLock()
	defer cm.mu.RUnlock()

	for client := range cm.clients {
		if client == exclude {
			continue // O'ziga yubormaslik (ixtiyoriy)
		}
		if err := client.Send(msg); err != nil {
			log.Printf("Yuborishda xato [%s]: %v", client.username, err)
			// Keyinroq o'chiramiz (RLock ichida o'chirish mumkin emas)
			go cm.Remove(client)
		}
	}
}

var (
	upgrader = websocket.Upgrader{
		ReadBufferSize:  1024,
		WriteBufferSize: 1024,
		CheckOrigin:     func(r *http.Request) bool { return true },
	}
	manager = NewClientManager()
)

func handleWebSocket(w http.ResponseWriter, r *http.Request) {
	conn, err := upgrader.Upgrade(w, r, nil)
	if err != nil {
		log.Printf("Upgrade xatosi: %v", err)
		return
	}

	username := r.URL.Query().Get("username")
	if username == "" {
		username = "anonim"
	}

	client := &Client{
		conn:     conn,
		username: username,
	}

	// Clientni ro'yxatga qo'shish
	manager.Add(client)
	log.Printf("[+] %s ulandi (jami: %d)", username, len(manager.clients))

	// Boshqalarga "qo'shildi" xabari
	manager.Broadcast(OutgoingMessage{
		Type:      MsgJoin,
		Content:   username + " chatga qo'shildi",
		From:      "system",
		Timestamp: time.Now(),
	}, nil)

	// Ulanish yopilganda
	defer func() {
		manager.Remove(client)
		log.Printf("[-] %s uzildi", username)
		manager.Broadcast(OutgoingMessage{
			Type:      MsgLeave,
			Content:   username + " chatdan chiqdi",
			From:      "system",
			Timestamp: time.Now(),
		}, nil)
	}()

	// Xabarlarni o'qish
	for {
		var msg IncomingMessage
		if err := conn.ReadJSON(&msg); err != nil {
			if websocket.IsUnexpectedCloseError(err,
				websocket.CloseGoingAway,
				websocket.CloseNormalClosure) {
				log.Printf("O'qish xatosi [%s]: %v", username, err)
			}
			break
		}

		if msg.Type == MsgChat && msg.Content != "" {
			manager.Broadcast(OutgoingMessage{
				Type:      MsgChat,
				Content:   msg.Content,
				From:      username,
				Timestamp: time.Now(),
			}, nil) // nil = hammaga, client = o'zidan tashqari
		}
	}
}

func main() {
	http.HandleFunc("/ws", handleWebSocket)
	log.Println("Broadcast server: ws://localhost:8080/ws?username=Ali")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### Muammo: Mutex Yondashuvi Cheklovlari

```
1. Broadcast vaqtida RLock ushlanadi
   → Agar bitta client sekin bo'lsa, BARCHASI kutadi

2. Broadcast ichida Remove chaqirish mumkin emas
   → go cm.Remove(client) — goroutine leak xavfi

3. WriteJSON sinxron — bitta client 100ms kutsa,
   broadcast 100 client x 100ms = 10 soniya!
```

**Yechim:** Phase 2 ning keyingi darsida — **Hub Pattern**.

---

## 5-Dars: Hub Pattern — Markaziy Boshqaruv

### Hub Pattern Nima?

Hub — barcha clientlarni va xabarlarni boshqaradigan **bitta goroutine**.

```
                    ┌─────────────────┐
Client A ──read───→ │                 │ ──write──→ Client A
Client B ──read───→ │    HUB          │ ──write──→ Client B
Client C ──read───→ │  (1 goroutine)  │ ──write──→ Client C
                    │                 │
                    │  register chan  │
                    │  unregister chan│
                    │  broadcast chan │
                    └─────────────────┘
```

**Afzalliklari:**
- **Mutex kerak emas** — faqat bitta goroutine map ni boshqaradi
- **Channel orqali muloqot** — Go idiomatik yondashuv
- **Sekin client boshqalarga ta'sir qilmaydi** — har bir client o'z write goroutine ga ega

### Hub Implementatsiyasi

```go
// ~/websocket-course/04-hub/hub.go
package main

import (
	"log"
	"time"
)

// Hub — markaziy xabar boshqaruvchisi
type Hub struct {
	// Ro'yxatdan o'tgan clientlar
	clients map[*Client]bool

	// Clientlardan kelgan xabarlar
	broadcast chan OutgoingMessage

	// Yangi client ro'yxatdan o'tishi
	register chan *Client

	// Client chiqishi
	unregister chan *Client
}

func NewHub() *Hub {
	return &Hub{
		clients:    make(map[*Client]bool),
		broadcast:  make(chan OutgoingMessage, 256),
		register:   make(chan *Client),
		unregister: make(chan *Client),
	}
}

// Run — Hub ning asosiy loop i (bitta goroutine da ishlaydi)
func (h *Hub) Run() {
	for {
		select {
		case client := <-h.register:
			h.clients[client] = true
			log.Printf("[Hub] +%s (jami: %d)", client.username, len(h.clients))

			// Boshqalarga xabar berish
			h.broadcastMessage(OutgoingMessage{
				Type:      MsgJoin,
				Content:   client.username + " chatga qo'shildi",
				From:      "system",
				Timestamp: time.Now(),
			}, nil)

		case client := <-h.unregister:
			if _, ok := h.clients[client]; ok {
				delete(h.clients, client)
				close(client.send) // Write pump ni to'xtatadi
				log.Printf("[Hub] -%s (jami: %d)", client.username, len(h.clients))

				h.broadcastMessage(OutgoingMessage{
					Type:      MsgLeave,
					Content:   client.username + " chatdan chiqdi",
					From:      "system",
					Timestamp: time.Now(),
				}, nil)
			}

		case msg := <-h.broadcast:
			h.broadcastMessage(msg, nil)
		}
	}
}

func (h *Hub) broadcastMessage(msg OutgoingMessage, exclude *Client) {
	for client := range h.clients {
		if client == exclude {
			continue
		}
		select {
		case client.send <- msg:
			// Xabar yuborildi
		default:
			// Client ning send buferi to'la — sekin client
			// Uni o'chiramiz
			close(client.send)
			delete(h.clients, client)
			log.Printf("[Hub] Sekin client o'chirildi: %s", client.username)
		}
	}
}
```

### Client — Read Pump / Write Pump

```go
// ~/websocket-course/04-hub/client.go
package main

import (
	"encoding/json"
	"log"
	"time"

	"github.com/gorilla/websocket"
)

const (
	// Xabar yozish uchun max kutish vaqti
	writeWait = 10 * time.Second

	// Ping oralig'i
	pongWait = 60 * time.Second

	// Ping yuborish oralig'i (pongWait dan kichik bo'lishi kerak)
	pingPeriod = (pongWait * 9) / 10

	// Max xabar hajmi
	maxMessageSize = 4096

	// Har bir client uchun yuborish buferi
	sendBufferSize = 256
)

type MessageType string

const (
	MsgChat  MessageType = "chat"
	MsgJoin  MessageType = "join"
	MsgLeave MessageType = "leave"
)

type IncomingMessage struct {
	Type    MessageType `json:"type"`
	Content string      `json:"content"`
}

type OutgoingMessage struct {
	Type      MessageType `json:"type"`
	Content   string      `json:"content,omitempty"`
	From      string      `json:"from"`
	Timestamp time.Time   `json:"timestamp"`
}

// Client — bitta WebSocket ulanish
type Client struct {
	hub      *Hub
	conn     *websocket.Conn
	send     chan OutgoingMessage // Yuborish buferi
	username string
}

// readPump — clientdan xabar o'qish (alohida goroutine)
// Har bir client uchun bitta readPump goroutine
func (c *Client) readPump() {
	defer func() {
		c.hub.unregister <- c
		c.conn.Close()
	}()

	c.conn.SetReadLimit(maxMessageSize)
	c.conn.SetReadDeadline(time.Now().Add(pongWait))
	c.conn.SetPongHandler(func(string) error {
		c.conn.SetReadDeadline(time.Now().Add(pongWait))
		return nil
	})

	for {
		_, raw, err := c.conn.ReadMessage()
		if err != nil {
			if websocket.IsUnexpectedCloseError(err,
				websocket.CloseGoingAway,
				websocket.CloseNormalClosure,
				websocket.CloseNoStatusReceived) {
				log.Printf("Read xatosi [%s]: %v", c.username, err)
			}
			break
		}

		var msg IncomingMessage
		if err := json.Unmarshal(raw, &msg); err != nil {
			log.Printf("JSON parse xatosi [%s]: %v", c.username, err)
			continue
		}

		if msg.Type == MsgChat && msg.Content != "" {
			c.hub.broadcast <- OutgoingMessage{
				Type:      MsgChat,
				Content:   msg.Content,
				From:      c.username,
				Timestamp: time.Now(),
			}
		}
	}
}

// writePump — clientga xabar yuborish (alohida goroutine)
// Har bir client uchun bitta writePump goroutine
func (c *Client) writePump() {
	ticker := time.NewTicker(pingPeriod)
	defer func() {
		ticker.Stop()
		c.conn.Close()
	}()

	for {
		select {
		case msg, ok := <-c.send:
			c.conn.SetWriteDeadline(time.Now().Add(writeWait))

			if !ok {
				// Hub send channelni yopgan — client o'chirilgan
				c.conn.WriteMessage(websocket.CloseMessage, []byte{})
				return
			}

			// JSON yozish
			if err := c.conn.WriteJSON(msg); err != nil {
				log.Printf("Write xatosi [%s]: %v", c.username, err)
				return
			}

		case <-ticker.C:
			// Ping yuborish — client hali ulanganmi tekshirish
			c.conn.SetWriteDeadline(time.Now().Add(writeWait))
			if err := c.conn.WriteMessage(websocket.PingMessage, nil); err != nil {
				return
			}
		}
	}
}
```

### Main Fayl

```go
// ~/websocket-course/04-hub/main.go
package main

import (
	"log"
	"net/http"

	"github.com/gorilla/websocket"
)

var upgrader = websocket.Upgrader{
	ReadBufferSize:  1024,
	WriteBufferSize: 1024,
	CheckOrigin:     func(r *http.Request) bool { return true },
}

func serveWS(hub *Hub, w http.ResponseWriter, r *http.Request) {
	conn, err := upgrader.Upgrade(w, r, nil)
	if err != nil {
		log.Printf("Upgrade xatosi: %v", err)
		return
	}

	username := r.URL.Query().Get("username")
	if username == "" {
		username = "anonim"
	}

	client := &Client{
		hub:      hub,
		conn:     conn,
		send:     make(chan OutgoingMessage, sendBufferSize),
		username: username,
	}

	// Hub ga ro'yxatdan o'tish
	client.hub.register <- client

	// Har bir client uchun 2 ta goroutine
	go client.writePump()
	go client.readPump()
}

func main() {
	hub := NewHub()
	go hub.Run() // Hub ni alohida goroutine da ishga tushirish

	http.HandleFunc("/ws", func(w http.ResponseWriter, r *http.Request) {
		serveWS(hub, w, r)
	})

	log.Println("Hub server: ws://localhost:8080/ws?username=Ali")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### Hub Pattern Diagrammasi

```
Client A                    Hub (1 goroutine)              Client A
┌──────────┐               ┌──────────────┐               ┌──────────┐
│readPump()│──broadcast──→ │              │ ──send chan──→ │writePump()│
│(goroutin)│               │  select {    │               │(goroutine)│
└──────────┘               │    register  │               └──────────┘
                           │    unregister│
Client B                   │    broadcast │               Client B
┌──────────┐               │  }           │               ┌──────────┐
│readPump()│──broadcast──→ │              │ ──send chan──→ │writePump()│
│(goroutin)│               │  clients map │               │(goroutine)│
└──────────┘               │  (faqat Hub  │               └──────────┘
                           │   o'zgartiradi)
Client C                   │              │               Client C
┌──────────┐               │              │               ┌──────────┐
│readPump()│──broadcast──→ │              │ ──send chan──→ │writePump()│
│(goroutin)│               └──────────────┘               │(goroutine)│
└──────────┘                                              └──────────┘

Goroutine lari:
  1 Hub + N readPump + N writePump = 1 + 2N goroutine
  1000 client = 2001 goroutine
```

### Nima Uchun Bu Pattern?

| Savol | Javob |
|-------|-------|
| Nega Hub bitta goroutine? | `clients` map ga faqat bitta goroutine yozadi — mutex kerak emas |
| Nega `send` channel? | `writePump` faqat channeldan o'qiydi — concurrent write yo'q |
| Nega `select default`? | Send channel to'la bo'lsa, sekin clientni o'chirish (block qilmaslik) |
| Nega ping/pong? | Dead connection larni aniqlash (cable uzilsa, TCP bilmaydi) |

---

## 6-Dars: Read Pump / Write Pump Chuqurroq

### Read Pump Vazifasi

```go
func (c *Client) readPump() {
    // 1. Cleanup (defer) — goroutine tugaganda
    defer func() {
        c.hub.unregister <- c  // Hub dan chiqish
        c.conn.Close()          // WebSocket yopish
    }()

    // 2. Limitlar o'rnatish
    c.conn.SetReadLimit(maxMessageSize)            // Max xabar hajmi
    c.conn.SetReadDeadline(time.Now().Add(pongWait)) // Timeout

    // 3. Pong handler — client "tirik" ekanligi tasdig'i
    c.conn.SetPongHandler(func(string) error {
        c.conn.SetReadDeadline(time.Now().Add(pongWait))
        return nil
    })

    // 4. Cheksiz o'qish loop
    for {
        _, raw, err := c.conn.ReadMessage()
        if err != nil {
            break  // Xato = ulanish uzilgan
        }
        // Xabarni qayta ishlash...
    }
}
```

**ReadPump qoidalari:**
- Har bir client uchun **bitta** readPump goroutine
- **Faqat o'qiydi** — hech qachon WriteMessage chaqirmaydi
- Goroutine tugashi = client uzildi

### Write Pump Vazifasi

```go
func (c *Client) writePump() {
    ticker := time.NewTicker(pingPeriod)
    defer func() {
        ticker.Stop()
        c.conn.Close()
    }()

    for {
        select {
        case msg, ok := <-c.send:
            if !ok {
                // Channel yopilgan — client o'chirilgan
                c.conn.WriteMessage(websocket.CloseMessage, []byte{})
                return
            }

            c.conn.SetWriteDeadline(time.Now().Add(writeWait))
            c.conn.WriteJSON(msg)

        case <-ticker.C:
            // Ping yuborish
            c.conn.SetWriteDeadline(time.Now().Add(writeWait))
            c.conn.WriteMessage(websocket.PingMessage, nil)
        }
    }
}
```

**WritePump qoidalari:**
- Har bir client uchun **bitta** writePump goroutine
- **Faqat yozadi** — hech qachon ReadMessage chaqirmaydi
- `send` channel orqali xabar oladi
- Ping/Pong boshqaradi

### Batched Write (Optimizatsiya)

Bir nechta xabar navbatda bo'lsa, barchasini bitta write operatsiyada yuborish:

```go
func (c *Client) writePump() {
	ticker := time.NewTicker(pingPeriod)
	defer func() {
		ticker.Stop()
		c.conn.Close()
	}()

	for {
		select {
		case msg, ok := <-c.send:
			if !ok {
				c.conn.WriteMessage(websocket.CloseMessage, []byte{})
				return
			}

			c.conn.SetWriteDeadline(time.Now().Add(writeWait))

			// NextWriter — bitta write operatsiyada ko'p xabar
			w, err := c.conn.NextWriter(websocket.TextMessage)
			if err != nil {
				return
			}

			// Birinchi xabarni yozish
			data, _ := json.Marshal(msg)
			w.Write(data)

			// Navbatdagi xabarlarni ham yozish (agar bo'lsa)
			n := len(c.send)
			for i := 0; i < n; i++ {
				w.Write([]byte("\n")) // Ajratuvchi
				nextMsg := <-c.send
				data, _ = json.Marshal(nextMsg)
				w.Write(data)
			}

			if err := w.Close(); err != nil {
				return
			}

		case <-ticker.C:
			c.conn.SetWriteDeadline(time.Now().Add(writeWait))
			if err := c.conn.WriteMessage(websocket.PingMessage, nil); err != nil {
				return
			}
		}
	}
}
```

### Goroutine Lifecycle

```
Client ulanganda:
  serveWS() → go readPump()   (goroutine 1)
            → go writePump()  (goroutine 2)

Normal ishlash:
  readPump:  conn.ReadMessage() ← client dan xabar kutadi
  writePump: <-c.send ← Hub dan xabar kutadi

Client uzilganda (3 xil senariy):

1. Client o'zi ulanishni yopdi:
   readPump: ReadMessage() xato → defer: hub.unregister + conn.Close()
   Hub: unregister → close(client.send)
   writePump: <-c.send → ok=false → return

2. Client javob bermaydi (pong kelmaydi):
   readPump: ReadDeadline o'tdi → ReadMessage() xato → defer...
   (1 bilan bir xil)

3. Yozishda xato (network):
   writePump: WriteJSON xato → return → conn.Close()
   readPump: ReadMessage() xato (conn yopilgan) → defer...
```

---

# PHASE 3 — Room System

---

## 7-Dars: Room (Xona) Arxitekturasi

### Nega Room Kerak?

Hub pattern da barcha clientlar bitta "xona"da. Real ilovada xonalar kerak:

```
Global broadcast:
  Ali → "salom" → [Ali, Vali, Soli, Anvar, Dilshod]  ← HAMMAGA

Room system:
  Room "backend":  [Ali, Vali, Soli]
  Room "frontend": [Anvar, Dilshod]
  Room "general":  [Ali, Anvar]

  Ali → "salom" (room: "backend") → [Ali, Vali, Soli]  ← faqat shu xonaga
```

### Arxitektura

```
┌───────────────────────────────────────────────┐
│                    HUB                        │
│                                               │
│  rooms map[string]*Room                       │
│  ┌──────────────┐  ┌──────────────┐          │
│  │ Room:backend │  │Room:frontend │          │
│  │  Ali         │  │  Anvar       │          │
│  │  Vali        │  │  Dilshod     │          │
│  │  Soli        │  │              │          │
│  └──────────────┘  └──────────────┘          │
│  ┌──────────────┐                            │
│  │Room:general  │                            │
│  │  Ali         │   Client bitta vaqtda      │
│  │  Anvar       │   bir nechta room da       │
│  └──────────────┘   bo'lishi mumkin          │
└───────────────────────────────────────────────┘
```

### Xabar Turlari (Kengaytirilgan)

```go
// ~/websocket-course/05-rooms/messages.go
package main

import "time"

type MessageType string

const (
	MsgChat       MessageType = "chat"
	MsgJoin       MessageType = "join"
	MsgLeave      MessageType = "leave"
	MsgRoomJoin   MessageType = "room.join"
	MsgRoomLeave  MessageType = "room.leave"
	MsgRoomList   MessageType = "room.list"
	MsgUserList   MessageType = "user.list"
	MsgPrivate    MessageType = "private"
	MsgError      MessageType = "error"
	MsgTyping     MessageType = "typing"
)

type IncomingMessage struct {
	Type    MessageType `json:"type"`
	Content string      `json:"content,omitempty"`
	Room    string      `json:"room,omitempty"`
	To      string      `json:"to,omitempty"` // private message uchun
}

type OutgoingMessage struct {
	Type      MessageType `json:"type"`
	Content   string      `json:"content,omitempty"`
	From      string      `json:"from,omitempty"`
	Room      string      `json:"room,omitempty"`
	Timestamp time.Time   `json:"timestamp"`
	Rooms     []string    `json:"rooms,omitempty"`
	Users     []string    `json:"users,omitempty"`
}
```

### Room Implementatsiyasi

```go
// ~/websocket-course/05-rooms/room.go
package main

import (
	"log"
	"time"
)

// Room — bitta xona
type Room struct {
	name    string
	clients map[*Client]bool
	hub     *Hub
}

func NewRoom(name string, hub *Hub) *Room {
	return &Room{
		name:    name,
		clients: make(map[*Client]bool),
		hub:     hub,
	}
}

// AddClient — xonaga client qo'shish
func (r *Room) AddClient(client *Client) {
	r.clients[client] = true
	log.Printf("[Room:%s] +%s (jami: %d)", r.name, client.username, len(r.clients))

	// Xonadagilarga xabar berish
	r.Broadcast(OutgoingMessage{
		Type:      MsgRoomJoin,
		Content:   client.username + " xonaga qo'shildi",
		From:      "system",
		Room:      r.name,
		Timestamp: time.Now(),
	}, nil)

	// Yangi clientga xona foydalanuvchilari ro'yxatini yuborish
	users := r.GetUsers()
	client.send <- OutgoingMessage{
		Type:      MsgUserList,
		Room:      r.name,
		Users:     users,
		Timestamp: time.Now(),
	}
}

// RemoveClient — xonadan client olib tashlash
func (r *Room) RemoveClient(client *Client) {
	if _, ok := r.clients[client]; !ok {
		return
	}
	delete(r.clients, client)
	log.Printf("[Room:%s] -%s (jami: %d)", r.name, client.username, len(r.clients))

	r.Broadcast(OutgoingMessage{
		Type:      MsgRoomLeave,
		Content:   client.username + " xonadan chiqdi",
		From:      "system",
		Room:      r.name,
		Timestamp: time.Now(),
	}, nil)
}

// Broadcast — xonadagi barcha clientlarga xabar
func (r *Room) Broadcast(msg OutgoingMessage, exclude *Client) {
	for client := range r.clients {
		if client == exclude {
			continue
		}
		select {
		case client.send <- msg:
		default:
			// Sekin client — o'chirish
			delete(r.clients, client)
			close(client.send)
		}
	}
}

// GetUsers — xonadagi foydalanuvchilar ro'yxati
func (r *Room) GetUsers() []string {
	users := make([]string, 0, len(r.clients))
	for client := range r.clients {
		users = append(users, client.username)
	}
	return users
}

// IsEmpty — xona bo'shmi
func (r *Room) IsEmpty() bool {
	return len(r.clients) == 0
}
```

---

## 8-Dars: Hub — Room Bilan Ishlash

### Hub (Room Qo'llab-quvvatlash Bilan)

```go
// ~/websocket-course/05-rooms/hub.go
package main

import (
	"log"
	"time"
)

// HubMessage — Hub ga yuboriladigan ichki xabar
type HubMessage struct {
	msg    OutgoingMessage
	room   string
	client *Client
}

// Hub — markaziy boshqaruv (room qo'llab-quvvatlash bilan)
type Hub struct {
	rooms      map[string]*Room
	clients    map[*Client]bool
	register   chan *Client
	unregister chan *Client
	broadcast  chan HubMessage
	joinRoom   chan RoomAction
	leaveRoom  chan RoomAction
}

type RoomAction struct {
	client *Client
	room   string
}

func NewHub() *Hub {
	return &Hub{
		rooms:      make(map[string]*Room),
		clients:    make(map[*Client]bool),
		register:   make(chan *Client),
		unregister: make(chan *Client),
		broadcast:  make(chan HubMessage, 256),
		joinRoom:   make(chan RoomAction, 256),
		leaveRoom:  make(chan RoomAction, 256),
	}
}

func (h *Hub) Run() {
	for {
		select {
		case client := <-h.register:
			h.clients[client] = true
			log.Printf("[Hub] Ro'yxatdan o'tdi: %s", client.username)

			// "general" xonasiga avtomatik qo'shish
			h.addClientToRoom(client, "general")

			// Mavjud xonalar ro'yxatini yuborish
			client.send <- OutgoingMessage{
				Type:      MsgRoomList,
				Rooms:     h.getRoomNames(),
				Timestamp: time.Now(),
			}

		case client := <-h.unregister:
			if _, ok := h.clients[client]; ok {
				// Barcha xonalardan chiqarish
				for _, room := range h.rooms {
					room.RemoveClient(client)
				}
				delete(h.clients, client)
				close(client.send)
				log.Printf("[Hub] Chiqdi: %s", client.username)

				// Bo'sh xonalarni tozalash
				h.cleanEmptyRooms()
			}

		case action := <-h.joinRoom:
			h.addClientToRoom(action.client, action.room)

		case action := <-h.leaveRoom:
			if room, ok := h.rooms[action.room]; ok {
				room.RemoveClient(action.client)
				if room.IsEmpty() {
					delete(h.rooms, action.room)
					log.Printf("[Hub] Bo'sh xona o'chirildi: %s", action.room)
				}
			}

		case hubMsg := <-h.broadcast:
			if hubMsg.room != "" {
				// Faqat shu xonaga yuborish
				if room, ok := h.rooms[hubMsg.room]; ok {
					room.Broadcast(hubMsg.msg, hubMsg.client)
				}
			} else {
				// Global broadcast
				for client := range h.clients {
					if client == hubMsg.client {
						continue
					}
					select {
					case client.send <- hubMsg.msg:
					default:
						close(client.send)
						delete(h.clients, client)
					}
				}
			}
		}
	}
}

func (h *Hub) addClientToRoom(client *Client, roomName string) {
	room, ok := h.rooms[roomName]
	if !ok {
		room = NewRoom(roomName, h)
		h.rooms[roomName] = room
		log.Printf("[Hub] Yangi xona yaratildi: %s", roomName)
	}
	room.AddClient(client)
	client.rooms[roomName] = true
}

func (h *Hub) getRoomNames() []string {
	names := make([]string, 0, len(h.rooms))
	for name := range h.rooms {
		names = append(names, name)
	}
	return names
}

func (h *Hub) cleanEmptyRooms() {
	for name, room := range h.rooms {
		if room.IsEmpty() && name != "general" {
			delete(h.rooms, name)
			log.Printf("[Hub] Bo'sh xona tozalandi: %s", name)
		}
	}
}
```

### Client (Room Qo'llab-quvvatlash Bilan)

```go
// ~/websocket-course/05-rooms/client.go
package main

import (
	"encoding/json"
	"log"
	"time"

	"github.com/gorilla/websocket"
)

const (
	writeWait      = 10 * time.Second
	pongWait       = 60 * time.Second
	pingPeriod     = (pongWait * 9) / 10
	maxMessageSize = 4096
	sendBufferSize = 256
)

type Client struct {
	hub      *Hub
	conn     *websocket.Conn
	send     chan OutgoingMessage
	username string
	rooms    map[string]bool // Client qaysi xonalarda
}

func (c *Client) readPump() {
	defer func() {
		c.hub.unregister <- c
		c.conn.Close()
	}()

	c.conn.SetReadLimit(maxMessageSize)
	c.conn.SetReadDeadline(time.Now().Add(pongWait))
	c.conn.SetPongHandler(func(string) error {
		c.conn.SetReadDeadline(time.Now().Add(pongWait))
		return nil
	})

	for {
		_, raw, err := c.conn.ReadMessage()
		if err != nil {
			if websocket.IsUnexpectedCloseError(err,
				websocket.CloseGoingAway,
				websocket.CloseNormalClosure) {
				log.Printf("Read xatosi [%s]: %v", c.username, err)
			}
			break
		}

		var msg IncomingMessage
		if err := json.Unmarshal(raw, &msg); err != nil {
			c.send <- OutgoingMessage{
				Type:      MsgError,
				Content:   "Noto'g'ri JSON format",
				Timestamp: time.Now(),
			}
			continue
		}

		c.handleMessage(msg)
	}
}

func (c *Client) handleMessage(msg IncomingMessage) {
	switch msg.Type {
	case MsgChat:
		if msg.Room == "" {
			msg.Room = "general"
		}
		// Faqat o'zi a'zo bo'lgan xonaga yozishi mumkin
		if !c.rooms[msg.Room] {
			c.send <- OutgoingMessage{
				Type:      MsgError,
				Content:   "Siz bu xonada emassiz: " + msg.Room,
				Timestamp: time.Now(),
			}
			return
		}
		c.hub.broadcast <- HubMessage{
			msg: OutgoingMessage{
				Type:      MsgChat,
				Content:   msg.Content,
				From:      c.username,
				Room:      msg.Room,
				Timestamp: time.Now(),
			},
			room:   msg.Room,
			client: nil, // Hammaga (o'ziga ham)
		}

	case MsgRoomJoin:
		if msg.Room == "" {
			return
		}
		c.hub.joinRoom <- RoomAction{client: c, room: msg.Room}

	case MsgRoomLeave:
		if msg.Room == "" || msg.Room == "general" {
			return // "general" dan chiqish mumkin emas
		}
		c.hub.leaveRoom <- RoomAction{client: c, room: msg.Room}
		delete(c.rooms, msg.Room)

	case MsgPrivate:
		c.handlePrivateMessage(msg)

	case MsgTyping:
		if msg.Room != "" && c.rooms[msg.Room] {
			c.hub.broadcast <- HubMessage{
				msg: OutgoingMessage{
					Type:      MsgTyping,
					From:      c.username,
					Room:      msg.Room,
					Timestamp: time.Now(),
				},
				room:   msg.Room,
				client: c, // O'zidan tashqari
			}
		}
	}
}

func (c *Client) handlePrivateMessage(msg IncomingMessage) {
	if msg.To == "" || msg.Content == "" {
		return
	}

	// Hub dan maqsad clientni topish
	for client := range c.hub.clients {
		if client.username == msg.To {
			client.send <- OutgoingMessage{
				Type:      MsgPrivate,
				Content:   msg.Content,
				From:      c.username,
				Timestamp: time.Now(),
			}
			return
		}
	}

	c.send <- OutgoingMessage{
		Type:      MsgError,
		Content:   "Foydalanuvchi topilmadi: " + msg.To,
		Timestamp: time.Now(),
	}
}

func (c *Client) writePump() {
	ticker := time.NewTicker(pingPeriod)
	defer func() {
		ticker.Stop()
		c.conn.Close()
	}()

	for {
		select {
		case msg, ok := <-c.send:
			c.conn.SetWriteDeadline(time.Now().Add(writeWait))
			if !ok {
				c.conn.WriteMessage(websocket.CloseMessage, []byte{})
				return
			}
			if err := c.conn.WriteJSON(msg); err != nil {
				return
			}

		case <-ticker.C:
			c.conn.SetWriteDeadline(time.Now().Add(writeWait))
			if err := c.conn.WriteMessage(websocket.PingMessage, nil); err != nil {
				return
			}
		}
	}
}
```

### Main

```go
// ~/websocket-course/05-rooms/main.go
package main

import (
	"log"
	"net/http"

	"github.com/gorilla/websocket"
)

var upgrader = websocket.Upgrader{
	ReadBufferSize:  1024,
	WriteBufferSize: 1024,
	CheckOrigin:     func(r *http.Request) bool { return true },
}

func serveWS(hub *Hub, w http.ResponseWriter, r *http.Request) {
	conn, err := upgrader.Upgrade(w, r, nil)
	if err != nil {
		return
	}

	username := r.URL.Query().Get("username")
	if username == "" {
		username = "anonim"
	}

	client := &Client{
		hub:      hub,
		conn:     conn,
		send:     make(chan OutgoingMessage, sendBufferSize),
		username: username,
		rooms:    make(map[string]bool),
	}

	client.hub.register <- client
	go client.writePump()
	go client.readPump()
}

func main() {
	hub := NewHub()
	go hub.Run()

	http.HandleFunc("/ws", func(w http.ResponseWriter, r *http.Request) {
		serveWS(hub, w, r)
	})

	log.Println("Room server: ws://localhost:8080/ws?username=Ali")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### Client Xabar Misollari

```json
// Xonaga qo'shilish
{"type": "room.join", "room": "backend"}

// Xonada xabar yozish
{"type": "chat", "room": "backend", "content": "Salom hammaga!"}

// Xonadan chiqish
{"type": "room.leave", "room": "backend"}

// Shaxsiy xabar
{"type": "private", "to": "Vali", "content": "Qanday ahvol?"}

// Yozayotgan...
{"type": "typing", "room": "backend"}
```

---

# PHASE 4 — Authentication va Xavfsizlik

---

## 9-Dars: JWT Bilan WebSocket Authentication

### Muammo

WebSocket da HTTP headerlardek `Authorization` header yuborish **brauzerdan mumkin emas**.

```javascript
// Bu ISHLAMAYDI brauzerda!
new WebSocket("ws://localhost:8080/ws", {
    headers: { "Authorization": "Bearer token123" }
});
```

### Yechimlar

**1. Query Parameter (Eng keng tarqalgan):**
```javascript
new WebSocket("ws://localhost:8080/ws?token=jwt_token_here");
```

**2. Birinchi xabar sifatida:**
```javascript
ws.onopen = () => {
    ws.send(JSON.stringify({type: "auth", token: "jwt_token_here"}));
};
```

**3. Cookie orqali:**
```javascript
// Cookie avtomatik yuboriladi
document.cookie = "token=jwt_token_here";
new WebSocket("ws://localhost:8080/ws");
```

Biz **1-usul** (query parameter) ishlatamiz — eng oddiy va keng tarqalgan.

### JWT Authentication Implementatsiyasi

```go
// ~/websocket-course/06-auth/auth.go
package main

import (
	"errors"
	"net/http"
	"time"

	"github.com/golang-jwt/jwt/v5"
)

var jwtSecret = []byte("your-secret-key-change-in-production")

// UserClaims — JWT da saqlanadigan ma'lumot
type UserClaims struct {
	UserID   int    `json:"user_id"`
	Username string `json:"username"`
	Role     string `json:"role"` // "admin", "user"
	jwt.RegisteredClaims
}

// GenerateToken — JWT token yaratish
func GenerateToken(userID int, username, role string) (string, error) {
	claims := UserClaims{
		UserID:   userID,
		Username: username,
		Role:     role,
		RegisteredClaims: jwt.RegisteredClaims{
			ExpiresAt: jwt.NewNumericDate(time.Now().Add(24 * time.Hour)),
			IssuedAt:  jwt.NewNumericDate(time.Now()),
		},
	}

	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	return token.SignedString(jwtSecret)
}

// ValidateToken — JWT tokenni tekshirish
func ValidateToken(tokenString string) (*UserClaims, error) {
	token, err := jwt.ParseWithClaims(tokenString, &UserClaims{},
		func(token *jwt.Token) (interface{}, error) {
			if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
				return nil, errors.New("noto'g'ri signing method")
			}
			return jwtSecret, nil
		})
	if err != nil {
		return nil, err
	}

	claims, ok := token.Claims.(*UserClaims)
	if !ok || !token.Valid {
		return nil, errors.New("noto'g'ri token")
	}

	return claims, nil
}

// AuthenticateWS — WebSocket ulanish uchun authentication
func AuthenticateWS(r *http.Request) (*UserClaims, error) {
	// 1. Query parameter dan token olish
	token := r.URL.Query().Get("token")

	// 2. Agar query da bo'lmasa, cookie dan qidiramiz
	if token == "" {
		cookie, err := r.Cookie("token")
		if err == nil {
			token = cookie.Value
		}
	}

	if token == "" {
		return nil, errors.New("token topilmadi")
	}

	return ValidateToken(token)
}
```

### Auth Bilan WebSocket Server

```go
// ~/websocket-course/06-auth/main.go
package main

import (
	"encoding/json"
	"log"
	"net/http"

	"github.com/gorilla/websocket"
)

var upgrader = websocket.Upgrader{
	ReadBufferSize:  1024,
	WriteBufferSize: 1024,
	CheckOrigin:     func(r *http.Request) bool { return true },
}

func serveWS(hub *Hub, w http.ResponseWriter, r *http.Request) {
	// 1. Authentication — UPGRADE dan OLDIN
	claims, err := AuthenticateWS(r)
	if err != nil {
		log.Printf("Auth xatosi: %v", err)
		http.Error(w, "Unauthorized", http.StatusUnauthorized)
		return
	}

	// 2. Upgrade
	conn, err := upgrader.Upgrade(w, r, nil)
	if err != nil {
		return
	}

	// 3. Autentifikatsiyalangan client yaratish
	client := &Client{
		hub:      hub,
		conn:     conn,
		send:     make(chan OutgoingMessage, sendBufferSize),
		userID:   claims.UserID,
		username: claims.Username,
		role:     claims.Role,
		rooms:    make(map[string]bool),
	}

	client.hub.register <- client
	go client.writePump()
	go client.readPump()
}

// Login endpoint — JWT token olish
func handleLogin(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		http.Error(w, "POST only", http.StatusMethodNotAllowed)
		return
	}

	var req struct {
		Username string `json:"username"`
		Password string `json:"password"`
	}
	json.NewDecoder(r.Body).Decode(&req)

	// Production da bazadan tekshiring!
	if req.Password != "secret123" {
		http.Error(w, "Noto'g'ri login", http.StatusUnauthorized)
		return
	}

	token, err := GenerateToken(1, req.Username, "user")
	if err != nil {
		http.Error(w, "Token xatosi", http.StatusInternalServerError)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(map[string]string{
		"token": token,
	})
}

func main() {
	hub := NewHub()
	go hub.Run()

	http.HandleFunc("/login", handleLogin)
	http.HandleFunc("/ws", func(w http.ResponseWriter, r *http.Request) {
		serveWS(hub, w, r)
	})

	log.Println("Auth server ishga tushdi :8080")
	log.Println("1. POST /login → token oling")
	log.Println("2. ws://localhost:8080/ws?token=... → ulaning")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### Test Qilish

```bash
# 1. Token olish
curl -X POST http://localhost:8080/login \
  -H "Content-Type: application/json" \
  -d '{"username":"Ali","password":"secret123"}'

# Javob: {"token":"eyJhbGciOiJIUzI1NiIs..."}

# 2. WebSocket ulanish
wscat -c "ws://localhost:8080/ws?token=eyJhbGciOiJIUzI1NiIs..."

# 3. Tokensiz ulanish (rad etilishi kerak)
wscat -c "ws://localhost:8080/ws"
# Xato: 401 Unauthorized
```

---

## 10-Dars: Origin Tekshirish, Rate Limiting, Xavfsizlik

### Origin Tekshirish (CORS)

```go
var upgrader = websocket.Upgrader{
	ReadBufferSize:  1024,
	WriteBufferSize: 1024,
	CheckOrigin: func(r *http.Request) bool {
		origin := r.Header.Get("Origin")
		allowedOrigins := map[string]bool{
			"https://myapp.com":     true,
			"https://www.myapp.com": true,
			"http://localhost:3000": true, // Development
		}
		return allowedOrigins[origin]
	},
}
```

### Rate Limiting

```go
// rate_limiter.go
package main

import (
	"sync"
	"time"
)

// RateLimiter — IP bo'yicha tezlik cheklash
type RateLimiter struct {
	mu       sync.Mutex
	visitors map[string]*visitor
	rate     int           // Max ulanishlar
	window   time.Duration // Vaqt oynasi
}

type visitor struct {
	count    int
	lastSeen time.Time
}

func NewRateLimiter(rate int, window time.Duration) *RateLimiter {
	rl := &RateLimiter{
		visitors: make(map[string]*visitor),
		rate:     rate,
		window:   window,
	}

	// Eski yozuvlarni tozalash
	go func() {
		for {
			time.Sleep(time.Minute)
			rl.mu.Lock()
			for ip, v := range rl.visitors {
				if time.Since(v.lastSeen) > rl.window {
					delete(rl.visitors, ip)
				}
			}
			rl.mu.Unlock()
		}
	}()

	return rl
}

// Allow — bu IP ga ruxsat bormi
func (rl *RateLimiter) Allow(ip string) bool {
	rl.mu.Lock()
	defer rl.mu.Unlock()

	v, exists := rl.visitors[ip]
	if !exists {
		rl.visitors[ip] = &visitor{count: 1, lastSeen: time.Now()}
		return true
	}

	if time.Since(v.lastSeen) > rl.window {
		v.count = 1
		v.lastSeen = time.Now()
		return true
	}

	if v.count >= rl.rate {
		return false
	}

	v.count++
	v.lastSeen = time.Now()
	return true
}

// Ishlatish:
// limiter := NewRateLimiter(10, time.Minute) // 1 daqiqada max 10 ulanish
//
// func serveWS(...) {
//     ip := r.RemoteAddr
//     if !limiter.Allow(ip) {
//         http.Error(w, "Too many requests", http.StatusTooManyRequests)
//         return
//     }
//     // ... upgrade
// }
```

### Xabar Rate Limiting (Per-Client)

```go
// Client ga qo'shish:
type Client struct {
	// ...
	msgCount    int
	msgResetAt  time.Time
	maxMsgRate  int // Daqiqada max xabar soni
}

func (c *Client) canSendMessage() bool {
	now := time.Now()
	if now.After(c.msgResetAt) {
		c.msgCount = 0
		c.msgResetAt = now.Add(time.Minute)
	}
	c.msgCount++
	return c.msgCount <= c.maxMsgRate
}

// readPump ichida:
// if !c.canSendMessage() {
//     c.send <- OutgoingMessage{Type: MsgError, Content: "Juda ko'p xabar!"}
//     continue
// }
```

### Xavfsizlik Checklist

```
☐ CheckOrigin — faqat ruxsat etilgan domainlar
☐ JWT/Token Authentication — Upgrade dan OLDIN
☐ SetReadLimit — max xabar hajmi (4-64KB)
☐ Rate Limiting — IP bo'yicha ulanish cheklash
☐ Message Rate Limiting — client boshiga xabar cheklash
☐ Input Validation — barcha kiritishni tekshirish
☐ TLS/WSS — production da DOIMO wss:// ishlatish
☐ Token Expiry — muddati o'tgan tokenlarni rad etish
☐ XSS himoya — HTML/JS kodni sanitize qilish
```

---

# PHASE 5 — Production Muammolari

---

## 11-Dars: Heartbeat — Ping/Pong

### Muammo: Dead Connections

```
Client A ──── Internet ──── Server
                 │
            Kabel uzildi
            (TCP bilmaydi!)
                 │
Client A (o'lik)              Server (hali "tirik" deb o'ylaydi)
                              → Xotira isrof
                              → Goroutine isrof
                              → Xabar yuborishga urinadi → xato
```

TCP o'zi ulanish uzilganini **darhol bilmaydi**. Bu soatlab davom etishi mumkin.

### Yechim: Ping/Pong Mexanizmi

```
Server                          Client
  │── Ping ──────────────────→│
  │←── Pong ──────────────────│  (avtomatik javob)
  │                            │
  │  (54 soniya kutish)        │
  │                            │
  │── Ping ──────────────────→│
  │←── Pong ──────────────────│
  │                            │
  │── Ping ──────────────────→│
  │         ╳ (javob yo'q)     │  (client o'lgan)
  │                            │
  │  pongWait o'tdi            │
  │  ReadDeadline expired      │
  │  → ReadMessage() xato     │
  │  → Client o'chiriladi     │
```

### gorilla/websocket da Ping/Pong

```go
const (
	// Client dan javob kutish vaqti
	pongWait = 60 * time.Second

	// Ping yuborish oralig'i (pongWait dan kichik bo'lishi SHART)
	pingPeriod = (pongWait * 9) / 10 // 54 soniya

	// Yozish uchun max kutish
	writeWait = 10 * time.Second
)

// readPump da:
func (c *Client) readPump() {
	defer func() {
		c.hub.unregister <- c
		c.conn.Close()
	}()

	// Dastlabki deadline
	c.conn.SetReadDeadline(time.Now().Add(pongWait))

	// Pong kelganda deadline yangilanadi
	c.conn.SetPongHandler(func(appData string) error {
		c.conn.SetReadDeadline(time.Now().Add(pongWait))
		return nil
	})

	for {
		_, _, err := c.conn.ReadMessage()
		if err != nil {
			break // Deadline o'tdi yoki ulanish uzildi
		}
	}
}

// writePump da:
func (c *Client) writePump() {
	ticker := time.NewTicker(pingPeriod)
	defer ticker.Stop()

	for {
		select {
		case msg, ok := <-c.send:
			// ... xabar yuborish

		case <-ticker.C:
			c.conn.SetWriteDeadline(time.Now().Add(writeWait))
			if err := c.conn.WriteMessage(websocket.PingMessage, nil); err != nil {
				return // Ping yuborib bo'lmadi — client o'lgan
			}
		}
	}
}
```

### Application-Level Heartbeat

Ba'zan Ping/Pong yetarli emas (proxy yoki load balancer Pong javob berishi mumkin). Application-level heartbeat:

```go
// Server har 30 soniyada:
client.send <- OutgoingMessage{
	Type:      "heartbeat",
	Timestamp: time.Now(),
}

// Client javob berishi kerak:
// {"type": "heartbeat_ack"}

// Agar 2 ta ketma-ket heartbeat ga javob kelmasa — uzish
```

---

## 12-Dars: Slow Client Muammosi

### Muammo

```
Server broadcast:
  Client A (tez)  → send channel ← [msg1] → yozildi
  Client B (sekin) → send channel ← [msg1, msg2, msg3, ... msg256] → TO'LDI!
  Client C (tez)  → send channel ← [msg1] → yozildi

Client B sekin o'qiyotgan bo'lsa:
  1. send channel to'ladi
  2. Hub broadcast da BLOCK bo'ladi
  3. BARCHA clientlar kutadi! (agar select default bo'lmasa)
```

### Yechim 1: select default (Hub da)

```go
// Hub.broadcastMessage da:
for client := range h.clients {
	select {
	case client.send <- msg:
		// OK — xabar yuborildi
	default:
		// Channel to'la — bu client sekin
		// Uni o'chiramiz
		close(client.send)
		delete(h.clients, client)
		log.Printf("Sekin client o'chirildi: %s", client.username)
	}
}
```

### Yechim 2: Write Deadline

```go
// writePump da:
c.conn.SetWriteDeadline(time.Now().Add(writeWait)) // 10 soniya
if err := c.conn.WriteJSON(msg); err != nil {
	return // 10 soniyada yozib bo'lmasa — uzish
}
```

### Yechim 3: Buffered Channel + O'chirish Siyosati

```go
// Client yaratishda:
client := &Client{
	send: make(chan OutgoingMessage, 256), // 256 xabargacha buffer
}

// Agar 256 dan oshsa — client sekin deb hisoblanadi va o'chiriladi
```

### Monitoring

```go
// Client send channel to'lishini kuzatish
func (h *Hub) broadcastMessage(msg OutgoingMessage, exclude *Client) {
	for client := range h.clients {
		if client == exclude {
			continue
		}

		// Channel to'lish darajasini kuzatish
		usage := len(client.send) * 100 / cap(client.send)
		if usage > 80 {
			log.Printf("OGOHLANTIRISH: %s send buferi %d%% to'la",
				client.username, usage)
		}

		select {
		case client.send <- msg:
		default:
			close(client.send)
			delete(h.clients, client)
		}
	}
}
```

---

## 13-Dars: Goroutine va Memory Leak Oldini Olish

### Goroutine Leak

```go
// XATO — Goroutine hech qachon to'xtamaydi!
func (c *Client) readPump() {
	for {
		_, _, err := c.conn.ReadMessage()
		if err != nil {
			return // Bu yerda to'xtaydi, lekin...
		}
		// Agar hub.broadcast channel to'la bo'lsa,
		// bu goroutine ABADIY kutadi:
		c.hub.broadcast <- msg // BLOCK! Goroutine leak!
	}
}
```

### Yechim: Context va Buffered Channel

```go
// 1. Buffered channel ishlatish
broadcast: make(chan HubMessage, 256), // 256 xabargacha buffer

// 2. Context bilan timeout
func (c *Client) readPump() {
	for {
		_, raw, err := c.conn.ReadMessage()
		if err != nil {
			return
		}

		// Timeout bilan yuborish
		select {
		case c.hub.broadcast <- HubMessage{msg: outMsg}:
			// OK
		case <-time.After(5 * time.Second):
			log.Printf("Broadcast timeout — Hub haddan tashqari band")
			return
		}
	}
}
```

### Memory Leak

```go
// XATO — conn.Close() chaqirilmaydi
func serveWS(hub *Hub, w http.ResponseWriter, r *http.Request) {
	conn, _ := upgrader.Upgrade(w, r, nil)
	// defer conn.Close() — UNUTILGAN!

	client := &Client{conn: conn, send: make(chan OutgoingMessage, 256)}
	// Agar readPump crash bo'lsa — conn va send channel GC tomonidan tozalanmaydi
}

// TO'G'RI
func (c *Client) readPump() {
	defer func() {
		c.hub.unregister <- c // Hub dan chiqarish
		c.conn.Close()         // WebSocket yopish
	}()
	// ...
}
```

### Goroutine Monitoring

```go
import "runtime"

// Har 30 soniyada goroutine sonini tekshirish
go func() {
	ticker := time.NewTicker(30 * time.Second)
	for range ticker.C {
		log.Printf("Goroutine soni: %d", runtime.NumGoroutine())
		log.Printf("Client soni: %d", len(hub.clients))

		// Kutilgan: 1 (Hub) + 2*N (readPump + writePump) + bir nechta system
		expected := 1 + 2*len(hub.clients) + 5
		actual := runtime.NumGoroutine()
		if actual > expected+100 {
			log.Printf("OGOHLANTIRISH: Goroutine leak! Kutilgan: ~%d, Haqiqiy: %d",
				expected, actual)
		}
	}
}()
```

### Cleanup Checklist

```
Har bir client uzilganda DOIMO:
  ☐ conn.Close() — WebSocket ulanishni yopish
  ☐ close(client.send) — write pump goroutineni to'xtatish
  ☐ Hub clients map dan o'chirish
  ☐ Room lardan o'chirish
  ☐ Timer/Ticker larni to'xtatish
  ☐ Context cancel qilish (agar ishlatilsa)
```

---

## 14-Dars: Client Reconnect Strategiyasi

### Server Tomoni — Graceful Shutdown

```go
// Graceful shutdown — barcha clientlarga close frame yuborish
func gracefulShutdown(hub *Hub) {
	log.Println("Server yopilmoqda...")

	// Barcha clientlarga close xabari
	for client := range hub.clients {
		client.send <- OutgoingMessage{
			Type:    MsgError,
			Content: "Server qayta ishga tushmoqda",
		}
		close(client.send)
	}

	// Clientlarga close frame yuborish uchun vaqt berish
	time.Sleep(2 * time.Second)
}
```

### Client Tomoni — Reconnect (JavaScript Misoli)

```javascript
class ReconnectingWebSocket {
    constructor(url) {
        this.url = url;
        this.maxRetries = 10;
        this.retryCount = 0;
        this.baseDelay = 1000;  // 1 soniya
        this.maxDelay = 30000;  // 30 soniya
        this.connect();
    }

    connect() {
        this.ws = new WebSocket(this.url);

        this.ws.onopen = () => {
            console.log("Ulandi");
            this.retryCount = 0; // Reset
        };

        this.ws.onclose = (e) => {
            if (e.code === 1000) return; // Normal yopilish

            if (this.retryCount < this.maxRetries) {
                let delay = Math.min(
                    this.baseDelay * Math.pow(2, this.retryCount),
                    this.maxDelay
                );
                // Jitter qo'shish (bir vaqtda hammasi ulana boshlashini oldini olish)
                delay += Math.random() * 1000;

                console.log(`Qayta ulanish ${this.retryCount + 1}/${this.maxRetries} — ${delay}ms`);
                this.retryCount++;
                setTimeout(() => this.connect(), delay);
            }
        };

        this.ws.onerror = () => {}; // onclose da handle qilinadi
    }

    send(data) {
        if (this.ws.readyState === WebSocket.OPEN) {
            this.ws.send(data);
        }
    }
}

// Ishlatish:
// const ws = new ReconnectingWebSocket("ws://localhost:8080/ws?token=...");
```

### Exponential Backoff Diagrammasi

```
Urinish 1: 1 soniya kutish    →  ulana olmadi
Urinish 2: 2 soniya kutish    →  ulana olmadi
Urinish 3: 4 soniya kutish    →  ulana olmadi
Urinish 4: 8 soniya kutish    →  ulana olmadi
Urinish 5: 16 soniya kutish   →  ulana olmadi
Urinish 6: 30 soniya kutish   →  ULANDI!

+ random jitter (0-1s) har safar qo'shiladi
  Bu 1000 ta client bir vaqtda reconnect qilishini oldini oladi
```

---

# PHASE 6 — Scaling (Ko'p Server)

---

## 15-Dars: Redis Pub/Sub Bilan Horizontal Scaling

### Muammo: Bitta Server Cheklovi

```
Bitta server:
  Client A ──→ Server 1 ──→ Client B   ✅ (ikkalasi bitta serverda)

Ikki server (Load Balancer bilan):
  Client A ──→ Server 1
  Client B ──→ Server 2
  Client A → "salom" → Server 1 → ??? → Server 2 → Client B   ❌

Server 1 Client B ni BILMAYDI!
```

### Yechim: Redis Pub/Sub

```
Client A ──→ Server 1 ──publish──→ REDIS ──subscribe──→ Server 2 ──→ Client B
Client C ──→ Server 1                                   Server 2 ──→ Client D

Redis barcha serverlarni bog'laydi!
```

### Arxitektura

```
                    ┌──────────────────┐
                    │   Load Balancer  │
                    │  (nginx/traefik) │
                    └────────┬─────────┘
                        │         │
               ┌────────┘         └────────┐
               ▼                           ▼
      ┌─────────────┐            ┌─────────────┐
      │  Server 1   │            │  Server 2   │
      │  Hub + WS   │            │  Hub + WS   │
      │  Client A,C │            │  Client B,D │
      └──────┬──────┘            └──────┬──────┘
             │                          │
             └──────────┬───────────────┘
                        │
                 ┌──────┴──────┐
                 │    REDIS    │
                 │   Pub/Sub   │
                 └─────────────┘
```

### Redis Pub/Sub Implementatsiyasi

```go
// ~/websocket-course/07-redis/redis_pubsub.go
package main

import (
	"context"
	"encoding/json"
	"log"

	"github.com/redis/go-redis/v9"
)

// RedisClient — Redis bilan ishlash
type RedisClient struct {
	client *redis.Client
	pubsub *redis.PubSub
	hub    *Hub
}

// RedisMessage — Redis orqali yuboriladigan xabar
type RedisMessage struct {
	Room     string          `json:"room"`
	Message  OutgoingMessage `json:"message"`
	ServerID string          `json:"server_id"` // O'z xabarini filtrlash uchun
}

func NewRedisClient(addr string, hub *Hub, serverID string) *RedisClient {
	client := redis.NewClient(&redis.Options{
		Addr: addr,
	})

	rc := &RedisClient{
		client: client,
		hub:    hub,
	}

	return rc
}

// Publish — xabarni Redis ga yuborish (barcha serverlar oladi)
func (rc *RedisClient) Publish(ctx context.Context, channel string, msg RedisMessage) error {
	data, err := json.Marshal(msg)
	if err != nil {
		return err
	}
	return rc.client.Publish(ctx, channel, data).Err()
}

// Subscribe — Redis kanaldan xabarlarni tinglash
func (rc *RedisClient) Subscribe(ctx context.Context, serverID string, channels ...string) {
	rc.pubsub = rc.client.Subscribe(ctx, channels...)

	ch := rc.pubsub.Channel()

	go func() {
		for msg := range ch {
			var redisMsg RedisMessage
			if err := json.Unmarshal([]byte(msg.Payload), &redisMsg); err != nil {
				log.Printf("Redis parse xatosi: %v", err)
				continue
			}

			// O'z serveridan kelgan xabarni e'tiborsiz qoldirish
			// (allaqachon local Hub orqali yuborilgan)
			if redisMsg.ServerID == serverID {
				continue
			}

			// Boshqa serverdan kelgan xabar — local clientlarga yuborish
			rc.hub.broadcast <- HubMessage{
				msg:  redisMsg.Message,
				room: redisMsg.Room,
			}
		}
	}()

	log.Printf("Redis subscribe: %v", channels)
}

// Close — Redis ulanishni yopish
func (rc *RedisClient) Close() error {
	if rc.pubsub != nil {
		rc.pubsub.Close()
	}
	return rc.client.Close()
}
```

### Hub — Redis Bilan

```go
// ~/websocket-course/07-redis/hub.go
package main

import (
	"context"
	"log"
	"time"
)

type Hub struct {
	clients    map[*Client]bool
	rooms      map[string]*Room
	register   chan *Client
	unregister chan *Client
	broadcast  chan HubMessage
	joinRoom   chan RoomAction
	leaveRoom  chan RoomAction
	redis      *RedisClient
	serverID   string
}

func NewHub(serverID string) *Hub {
	return &Hub{
		clients:    make(map[*Client]bool),
		rooms:      make(map[string]*Room),
		register:   make(chan *Client),
		unregister: make(chan *Client),
		broadcast:  make(chan HubMessage, 256),
		joinRoom:   make(chan RoomAction, 256),
		leaveRoom:  make(chan RoomAction, 256),
		serverID:   serverID,
	}
}

func (h *Hub) SetRedis(rc *RedisClient) {
	h.redis = rc
}

func (h *Hub) Run() {
	for {
		select {
		case client := <-h.register:
			h.clients[client] = true
			h.addClientToRoom(client, "general")
			log.Printf("[Hub:%s] +%s", h.serverID, client.username)

		case client := <-h.unregister:
			if _, ok := h.clients[client]; ok {
				for _, room := range h.rooms {
					room.RemoveClient(client)
				}
				delete(h.clients, client)
				close(client.send)
			}

		case action := <-h.joinRoom:
			h.addClientToRoom(action.client, action.room)

		case action := <-h.leaveRoom:
			if room, ok := h.rooms[action.room]; ok {
				room.RemoveClient(action.client)
			}

		case hubMsg := <-h.broadcast:
			// LOCAL clientlarga yuborish
			if hubMsg.room != "" {
				if room, ok := h.rooms[hubMsg.room]; ok {
					room.Broadcast(hubMsg.msg, hubMsg.client)
				}
			} else {
				for client := range h.clients {
					if client == hubMsg.client {
						continue
					}
					select {
					case client.send <- hubMsg.msg:
					default:
						close(client.send)
						delete(h.clients, client)
					}
				}
			}

			// REDIS ga ham yuborish (boshqa serverlarga)
			if h.redis != nil && hubMsg.client != nil {
				// Faqat local clientdan kelgan xabarlarni Redis ga yuboramiz
				// (Redis dan kelganlarni qayta yubormaymiz)
				h.redis.Publish(context.Background(), "ws:messages", RedisMessage{
					Room:     hubMsg.room,
					Message:  hubMsg.msg,
					ServerID: h.serverID,
				})
			}
		}
	}
}

func (h *Hub) addClientToRoom(client *Client, roomName string) {
	room, ok := h.rooms[roomName]
	if !ok {
		room = NewRoom(roomName, h)
		h.rooms[roomName] = room
	}
	room.AddClient(client)
	client.rooms[roomName] = true
}
```

### Main — Multi-Server

```go
// ~/websocket-course/07-redis/main.go
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"os"

	"github.com/gorilla/websocket"
)

var upgrader = websocket.Upgrader{
	ReadBufferSize:  1024,
	WriteBufferSize: 1024,
	CheckOrigin:     func(r *http.Request) bool { return true },
}

func main() {
	// Server ID va port ni env dan olish
	serverID := os.Getenv("SERVER_ID")
	if serverID == "" {
		serverID = "server-1"
	}
	port := os.Getenv("PORT")
	if port == "" {
		port = "8080"
	}
	redisAddr := os.Getenv("REDIS_ADDR")
	if redisAddr == "" {
		redisAddr = "localhost:6379"
	}

	// Hub yaratish
	hub := NewHub(serverID)

	// Redis ulash
	rc := NewRedisClient(redisAddr, hub, serverID)
	defer rc.Close()
	hub.SetRedis(rc)

	// Redis subscribe
	rc.Subscribe(context.Background(), serverID, "ws:messages")

	go hub.Run()

	http.HandleFunc("/ws", func(w http.ResponseWriter, r *http.Request) {
		conn, err := upgrader.Upgrade(w, r, nil)
		if err != nil {
			return
		}
		username := r.URL.Query().Get("username")
		if username == "" {
			username = "anonim"
		}

		client := &Client{
			hub:      hub,
			conn:     conn,
			send:     make(chan OutgoingMessage, 256),
			username: username,
			rooms:    make(map[string]bool),
		}
		hub.register <- client
		go client.writePump()
		go client.readPump()
	})

	log.Printf("[%s] Server ishga tushdi :%s (Redis: %s)", serverID, port, redisAddr)
	log.Fatal(http.ListenAndServe(fmt.Sprintf(":%s", port), nil))
}
```

### Docker Compose — Multi Server Test

```yaml
# ~/websocket-course/07-redis/docker-compose.yml
version: '3.8'

services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  ws-server-1:
    build: .
    ports:
      - "8081:8080"
    environment:
      SERVER_ID: server-1
      PORT: "8080"
      REDIS_ADDR: redis:6379
    depends_on:
      - redis

  ws-server-2:
    build: .
    ports:
      - "8082:8080"
    environment:
      SERVER_ID: server-2
      PORT: "8080"
      REDIS_ADDR: redis:6379
    depends_on:
      - redis

  nginx:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - ws-server-1
      - ws-server-2
```

### Nginx Load Balancer

```nginx
# ~/websocket-course/07-redis/nginx.conf
events {
    worker_connections 1024;
}

http {
    upstream websocket_servers {
        # ip_hash — bitta client doimo bitta serverga yo'naltiriladi
        ip_hash;
        server ws-server-1:8080;
        server ws-server-2:8080;
    }

    server {
        listen 80;

        location /ws {
            proxy_pass http://websocket_servers;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_read_timeout 86400;  # 24 soat (WebSocket timeout)
        }
    }
}
```

### Test

```bash
# Ishga tushirish
docker compose up -d

# Terminal 1: Server 1 ga ulanish
wscat -c "ws://localhost:8081/ws?username=Ali"

# Terminal 2: Server 2 ga ulanish
wscat -c "ws://localhost:8082/ws?username=Vali"

# Ali xabar yozadi → Vali oladi (turli serverlarda!)
> {"type":"chat","content":"Salom!","room":"general"}
```

---

# PHASE 7 — Real Loyiha

---

## 16-Dars: Real-Time Chat Service — To'liq Loyiha

### Loyiha Tuzilishi

```
chat-service/
├── cmd/
│   └── server/
│       └── main.go           ← Entry point
├── internal/
│   ├── auth/
│   │   └── jwt.go            ← Authentication
│   ├── chat/
│   │   ├── client.go         ← WebSocket client
│   │   ├── hub.go            ← Markaziy Hub
│   │   ├── room.go           ← Room boshqaruvi
│   │   └── message.go        ← Xabar turlari
│   ├── middleware/
│   │   └── ratelimit.go      ← Rate limiting
│   └── redis/
│       └── pubsub.go         ← Redis Pub/Sub
├── docker-compose.yml
├── Dockerfile
├── go.mod
└── go.sum
```

### Message Protokoli

```go
// internal/chat/message.go
package chat

import "time"

type Action string

const (
	ActionChat       Action = "chat"
	ActionJoin       Action = "join"
	ActionLeave      Action = "leave"
	ActionRoomJoin   Action = "room.join"
	ActionRoomLeave  Action = "room.leave"
	ActionRoomList   Action = "room.list"
	ActionUserList   Action = "user.list"
	ActionPrivate    Action = "private"
	ActionTyping     Action = "typing"
	ActionError      Action = "error"
	ActionHistory    Action = "history"
)

// ClientMessage — clientdan kelgan xabar
type ClientMessage struct {
	Action  Action `json:"action"`
	Content string `json:"content,omitempty"`
	Room    string `json:"room,omitempty"`
	To      string `json:"to,omitempty"`
}

// ServerMessage — clientga yuboriladigan xabar
type ServerMessage struct {
	Action    Action          `json:"action"`
	Content   string          `json:"content,omitempty"`
	From      string          `json:"from,omitempty"`
	Room      string          `json:"room,omitempty"`
	Timestamp time.Time       `json:"timestamp"`
	Data      interface{}     `json:"data,omitempty"`
}
```

### Client (Production Darajasi)

```go
// internal/chat/client.go
package chat

import (
	"encoding/json"
	"log"
	"time"

	"github.com/gorilla/websocket"
)

const (
	writeWait      = 10 * time.Second
	pongWait       = 60 * time.Second
	pingPeriod     = (pongWait * 9) / 10
	maxMessageSize = 4096
	sendBufferSize = 256
	maxMsgPerMin   = 60 // Daqiqada max xabar
)

type Client struct {
	hub      *Hub
	conn     *websocket.Conn
	send     chan ServerMessage
	userID   int
	username string
	rooms    map[string]bool

	// Rate limiting
	msgCount   int
	msgResetAt time.Time
}

func NewClient(hub *Hub, conn *websocket.Conn, userID int, username string) *Client {
	return &Client{
		hub:        hub,
		conn:       conn,
		send:       make(chan ServerMessage, sendBufferSize),
		userID:     userID,
		username:   username,
		rooms:      make(map[string]bool),
		msgResetAt: time.Now().Add(time.Minute),
	}
}

func (c *Client) ReadPump() {
	defer func() {
		c.hub.unregister <- c
		c.conn.Close()
	}()

	c.conn.SetReadLimit(maxMessageSize)
	c.conn.SetReadDeadline(time.Now().Add(pongWait))
	c.conn.SetPongHandler(func(string) error {
		c.conn.SetReadDeadline(time.Now().Add(pongWait))
		return nil
	})

	for {
		_, raw, err := c.conn.ReadMessage()
		if err != nil {
			if websocket.IsUnexpectedCloseError(err,
				websocket.CloseGoingAway,
				websocket.CloseNormalClosure,
				websocket.CloseNoStatusReceived) {
				log.Printf("Read xatosi [%s]: %v", c.username, err)
			}
			return
		}

		// Rate limit tekshirish
		if !c.checkRateLimit() {
			c.send <- ServerMessage{
				Action:    ActionError,
				Content:   "Juda ko'p xabar! Biroz kuting.",
				Timestamp: time.Now(),
			}
			continue
		}

		var msg ClientMessage
		if err := json.Unmarshal(raw, &msg); err != nil {
			c.send <- ServerMessage{
				Action:    ActionError,
				Content:   "Noto'g'ri JSON format",
				Timestamp: time.Now(),
			}
			continue
		}

		c.handleMessage(msg)
	}
}

func (c *Client) WritePump() {
	ticker := time.NewTicker(pingPeriod)
	defer func() {
		ticker.Stop()
		c.conn.Close()
	}()

	for {
		select {
		case msg, ok := <-c.send:
			c.conn.SetWriteDeadline(time.Now().Add(writeWait))
			if !ok {
				c.conn.WriteMessage(websocket.CloseMessage, []byte{})
				return
			}

			w, err := c.conn.NextWriter(websocket.TextMessage)
			if err != nil {
				return
			}
			json.NewEncoder(w).Encode(msg)

			// Navbatdagi xabarlarni ham yozish
			n := len(c.send)
			for i := 0; i < n; i++ {
				nextMsg := <-c.send
				json.NewEncoder(w).Encode(nextMsg)
			}

			if err := w.Close(); err != nil {
				return
			}

		case <-ticker.C:
			c.conn.SetWriteDeadline(time.Now().Add(writeWait))
			if err := c.conn.WriteMessage(websocket.PingMessage, nil); err != nil {
				return
			}
		}
	}
}

func (c *Client) handleMessage(msg ClientMessage) {
	switch msg.Action {
	case ActionChat:
		room := msg.Room
		if room == "" {
			room = "general"
		}
		if !c.rooms[room] {
			c.send <- ServerMessage{
				Action:  ActionError,
				Content: "Siz bu xonada emassiz: " + room,
			}
			return
		}
		if msg.Content == "" {
			return
		}

		c.hub.broadcast <- &BroadcastMessage{
			msg: ServerMessage{
				Action:    ActionChat,
				Content:   msg.Content,
				From:      c.username,
				Room:      room,
				Timestamp: time.Now(),
			},
			room:   room,
			sender: c,
		}

	case ActionRoomJoin:
		if msg.Room != "" {
			c.hub.joinRoom <- &RoomAction{client: c, room: msg.Room}
		}

	case ActionRoomLeave:
		if msg.Room != "" && msg.Room != "general" {
			c.hub.leaveRoom <- &RoomAction{client: c, room: msg.Room}
			delete(c.rooms, msg.Room)
		}

	case ActionPrivate:
		if msg.To != "" && msg.Content != "" {
			c.hub.privateMsg <- &PrivateMessage{
				from:    c,
				to:      msg.To,
				content: msg.Content,
			}
		}

	case ActionTyping:
		if msg.Room != "" && c.rooms[msg.Room] {
			c.hub.broadcast <- &BroadcastMessage{
				msg: ServerMessage{
					Action: ActionTyping,
					From:   c.username,
					Room:   msg.Room,
				},
				room:   msg.Room,
				sender: c,
			}
		}
	}
}

func (c *Client) checkRateLimit() bool {
	now := time.Now()
	if now.After(c.msgResetAt) {
		c.msgCount = 0
		c.msgResetAt = now.Add(time.Minute)
	}
	c.msgCount++
	return c.msgCount <= maxMsgPerMin
}
```

### Hub (Production Darajasi)

```go
// internal/chat/hub.go
package chat

import (
	"log"
	"runtime"
	"time"
)

type BroadcastMessage struct {
	msg    ServerMessage
	room   string
	sender *Client
}

type RoomAction struct {
	client *Client
	room   string
}

type PrivateMessage struct {
	from    *Client
	to      string
	content string
}

type Hub struct {
	clients    map[*Client]bool
	rooms      map[string]*Room
	register   chan *Client
	unregister chan *Client
	broadcast  chan *BroadcastMessage
	joinRoom   chan *RoomAction
	leaveRoom  chan *RoomAction
	privateMsg chan *PrivateMessage
	serverID   string
}

func NewHub(serverID string) *Hub {
	return &Hub{
		clients:    make(map[*Client]bool),
		rooms:      make(map[string]*Room),
		register:   make(chan *Client),
		unregister: make(chan *Client),
		broadcast:  make(chan *BroadcastMessage, 256),
		joinRoom:   make(chan *RoomAction, 256),
		leaveRoom:  make(chan *RoomAction, 256),
		privateMsg: make(chan *PrivateMessage, 256),
		serverID:   serverID,
	}
}

func (h *Hub) Run() {
	// Monitoring ticker
	statsTicker := time.NewTicker(30 * time.Second)
	defer statsTicker.Stop()

	for {
		select {
		case client := <-h.register:
			h.clients[client] = true
			h.addToRoom(client, "general")

			// Xush kelibsiz xabari
			client.send <- ServerMessage{
				Action:    ActionJoin,
				Content:   "Xush kelibsiz, " + client.username + "!",
				From:      "system",
				Timestamp: time.Now(),
				Data: map[string]interface{}{
					"rooms": h.getRoomNames(),
				},
			}

			log.Printf("[Hub] +%s (clients: %d, goroutines: %d)",
				client.username, len(h.clients), runtime.NumGoroutine())

		case client := <-h.unregister:
			if _, ok := h.clients[client]; ok {
				for name, room := range h.rooms {
					room.Remove(client)
					if room.IsEmpty() && name != "general" {
						delete(h.rooms, name)
					}
				}
				delete(h.clients, client)
				close(client.send)
				log.Printf("[Hub] -%s (clients: %d)", client.username, len(h.clients))
			}

		case bm := <-h.broadcast:
			if bm.room != "" {
				if room, ok := h.rooms[bm.room]; ok {
					room.Broadcast(bm.msg, bm.sender)
				}
			}

		case action := <-h.joinRoom:
			h.addToRoom(action.client, action.room)

		case action := <-h.leaveRoom:
			if room, ok := h.rooms[action.room]; ok {
				room.Remove(action.client)
			}

		case pm := <-h.privateMsg:
			h.handlePrivate(pm)

		case <-statsTicker.C:
			log.Printf("[Stats] Clients: %d, Rooms: %d, Goroutines: %d",
				len(h.clients), len(h.rooms), runtime.NumGoroutine())
		}
	}
}

func (h *Hub) addToRoom(client *Client, roomName string) {
	room, ok := h.rooms[roomName]
	if !ok {
		room = NewRoom(roomName)
		h.rooms[roomName] = room
	}
	room.Add(client)
	client.rooms[roomName] = true

	// Xona foydalanuvchilari ro'yxati
	client.send <- ServerMessage{
		Action:    ActionUserList,
		Room:      roomName,
		Timestamp: time.Now(),
		Data:      room.GetUsernames(),
	}
}

func (h *Hub) handlePrivate(pm *PrivateMessage) {
	for client := range h.clients {
		if client.username == pm.to {
			client.send <- ServerMessage{
				Action:    ActionPrivate,
				Content:   pm.content,
				From:      pm.from.username,
				Timestamp: time.Now(),
			}
			// Yuboruvchiga tasdiqlash
			pm.from.send <- ServerMessage{
				Action:    ActionPrivate,
				Content:   pm.content,
				From:      pm.from.username,
				Timestamp: time.Now(),
				Data:      map[string]string{"to": pm.to, "status": "delivered"},
			}
			return
		}
	}

	pm.from.send <- ServerMessage{
		Action:  ActionError,
		Content: "Foydalanuvchi topilmadi: " + pm.to,
	}
}

func (h *Hub) getRoomNames() []string {
	names := make([]string, 0, len(h.rooms))
	for name := range h.rooms {
		names = append(names, name)
	}
	return names
}
```

### Room

```go
// internal/chat/room.go
package chat

import "log"

type Room struct {
	name    string
	clients map[*Client]bool
}

func NewRoom(name string) *Room {
	return &Room{
		name:    name,
		clients: make(map[*Client]bool),
	}
}

func (r *Room) Add(client *Client) {
	r.clients[client] = true

	r.Broadcast(ServerMessage{
		Action:  ActionRoomJoin,
		Content: client.username + " xonaga qo'shildi",
		From:    "system",
		Room:    r.name,
	}, nil)
}

func (r *Room) Remove(client *Client) {
	if _, ok := r.clients[client]; !ok {
		return
	}
	delete(r.clients, client)

	r.Broadcast(ServerMessage{
		Action:  ActionRoomLeave,
		Content: client.username + " xonadan chiqdi",
		From:    "system",
		Room:    r.name,
	}, nil)
}

func (r *Room) Broadcast(msg ServerMessage, exclude *Client) {
	for client := range r.clients {
		if client == exclude {
			continue
		}
		select {
		case client.send <- msg:
		default:
			log.Printf("[Room:%s] Sekin client: %s", r.name, client.username)
			delete(r.clients, client)
			close(client.send)
		}
	}
}

func (r *Room) GetUsernames() []string {
	users := make([]string, 0, len(r.clients))
	for client := range r.clients {
		users = append(users, client.username)
	}
	return users
}

func (r *Room) IsEmpty() bool {
	return len(r.clients) == 0
}
```

### Entry Point

```go
// cmd/server/main.go
package main

import (
	"log"
	"net/http"
	"os"

	"github.com/gorilla/websocket"
	// import paths loyihangizga qarab o'zgartiring
)

var upgrader = websocket.Upgrader{
	ReadBufferSize:  1024,
	WriteBufferSize: 1024,
	CheckOrigin: func(r *http.Request) bool {
		origin := r.Header.Get("Origin")
		allowed := map[string]bool{
			"http://localhost:3000": true,
			"https://myapp.com":    true,
		}
		if os.Getenv("ENV") == "development" {
			return true
		}
		return allowed[origin]
	},
}

func main() {
	serverID := os.Getenv("SERVER_ID")
	if serverID == "" {
		serverID = "server-1"
	}
	port := os.Getenv("PORT")
	if port == "" {
		port = "8080"
	}

	hub := NewHub(serverID)
	go hub.Run()

	http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
		w.Write([]byte("OK"))
	})

	http.HandleFunc("/ws", func(w http.ResponseWriter, r *http.Request) {
		// Production da auth qo'shing
		conn, err := upgrader.Upgrade(w, r, nil)
		if err != nil {
			return
		}

		username := r.URL.Query().Get("username")
		if username == "" {
			conn.WriteJSON(ServerMessage{Action: ActionError, Content: "username kerak"})
			conn.Close()
			return
		}

		client := NewClient(hub, conn, 0, username)
		hub.register <- client
		go client.WritePump()
		go client.ReadPump()
	})

	log.Printf("[%s] Chat server :%s", serverID, port)
	log.Fatal(http.ListenAndServe(":"+port, nil))
}
```

---

## 17-Dars: Real-Time Notification Service

### Notification Arxitekturasi

```
┌────────────────┐        ┌──────────┐
│ Backend API    │──push──→│          │
│ (REST/gRPC)    │        │  Notif   │        ┌─────────┐
└────────────────┘        │  Service │──ws───→│ Client  │
                          │          │        │(brauzer)│
┌────────────────┐        │ Internal │        └─────────┘
│ Cron Jobs      │──push──→│ HTTP API │
└────────────────┘        │          │        ┌─────────┐
                          │          │──ws───→│ Mobile  │
┌────────────────┐        │          │        │  App    │
│ Kafka Consumer │──push──→│          │        └─────────┘
└────────────────┘        └──────────┘
```

### Notification Service Kodi

```go
// notification-service/main.go
package main

import (
	"encoding/json"
	"log"
	"net/http"
	"sync"
	"time"

	"github.com/gorilla/websocket"
)

// Notification turlari
type NotificationType string

const (
	NotifInfo    NotificationType = "info"
	NotifSuccess NotificationType = "success"
	NotifWarning NotificationType = "warning"
	NotifError   NotificationType = "error"
	NotifOrder   NotificationType = "order"
	NotifPayment NotificationType = "payment"
	NotifMessage NotificationType = "message"
)

type Notification struct {
	ID        string           `json:"id"`
	Type      NotificationType `json:"type"`
	Title     string           `json:"title"`
	Body      string           `json:"body"`
	Data      interface{}      `json:"data,omitempty"`
	Read      bool             `json:"read"`
	CreatedAt time.Time        `json:"created_at"`
}

// UserConnection — foydalanuvchi va uning ulanishlari
type NotifHub struct {
	// userID → []*Client (bitta user bir nechta qurilmada)
	users map[int][]*NotifClient
	mu    sync.RWMutex
}

type NotifClient struct {
	conn   *websocket.Conn
	send   chan Notification
	userID int
	mu     sync.Mutex
}

var (
	hub = &NotifHub{
		users: make(map[int][]*NotifClient),
	}
	upgrader = websocket.Upgrader{
		CheckOrigin: func(r *http.Request) bool { return true },
	}
)

// SendToUser — ma'lum foydalanuvchiga notification yuborish
func (h *NotifHub) SendToUser(userID int, notif Notification) {
	h.mu.RLock()
	clients, ok := h.users[userID]
	h.mu.RUnlock()

	if !ok {
		log.Printf("User %d online emas, notification saqlanadi", userID)
		// Production da: bazaga yoki Redis ga saqlang
		return
	}

	for _, client := range clients {
		select {
		case client.send <- notif:
		default:
			log.Printf("User %d buferi to'la", userID)
		}
	}
}

// HTTP API — tashqi servislar notification yuborishi uchun
func handlePushNotification(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		http.Error(w, "POST only", http.StatusMethodNotAllowed)
		return
	}

	var req struct {
		UserID int          `json:"user_id"`
		Notif  Notification `json:"notification"`
	}

	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, "Bad request", http.StatusBadRequest)
		return
	}

	req.Notif.CreatedAt = time.Now()
	req.Notif.Read = false

	hub.SendToUser(req.UserID, req.Notif)

	w.WriteHeader(http.StatusOK)
	json.NewEncoder(w).Encode(map[string]string{"status": "sent"})
}

// WebSocket handler
func handleWS(w http.ResponseWriter, r *http.Request) {
	conn, err := upgrader.Upgrade(w, r, nil)
	if err != nil {
		return
	}

	// Production da: JWT dan userID oling
	userID := 1 // Soddalashtirilgan

	client := &NotifClient{
		conn:   conn,
		send:   make(chan Notification, 64),
		userID: userID,
	}

	// Ro'yxatga olish
	hub.mu.Lock()
	hub.users[userID] = append(hub.users[userID], client)
	hub.mu.Unlock()

	log.Printf("User %d ulandi (qurilmalar: %d)", userID, len(hub.users[userID]))

	// Cleanup
	defer func() {
		hub.mu.Lock()
		clients := hub.users[userID]
		for i, c := range clients {
			if c == client {
				hub.users[userID] = append(clients[:i], clients[i+1:]...)
				break
			}
		}
		if len(hub.users[userID]) == 0 {
			delete(hub.users, userID)
		}
		hub.mu.Unlock()
		conn.Close()
	}()

	// Write pump
	go func() {
		ticker := time.NewTicker(54 * time.Second)
		defer ticker.Stop()
		for {
			select {
			case notif, ok := <-client.send:
				if !ok {
					return
				}
				conn.SetWriteDeadline(time.Now().Add(10 * time.Second))
				conn.WriteJSON(notif)
			case <-ticker.C:
				conn.WriteMessage(websocket.PingMessage, nil)
			}
		}
	}()

	// Read pump (faqat pong uchun)
	conn.SetReadDeadline(time.Now().Add(60 * time.Second))
	conn.SetPongHandler(func(string) error {
		conn.SetReadDeadline(time.Now().Add(60 * time.Second))
		return nil
	})
	for {
		if _, _, err := conn.ReadMessage(); err != nil {
			break
		}
	}
}

func main() {
	http.HandleFunc("/ws", handleWS)
	http.HandleFunc("/api/notify", handlePushNotification)

	log.Println("Notification service :8080")
	log.Println("  WS:  ws://localhost:8080/ws")
	log.Println("  API: POST http://localhost:8080/api/notify")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### Test

```bash
# Terminal 1: Server
go run main.go

# Terminal 2: WebSocket ulanish
wscat -c ws://localhost:8080/ws

# Terminal 3: Notification yuborish
curl -X POST http://localhost:8080/api/notify \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": 1,
    "notification": {
      "id": "n1",
      "type": "order",
      "title": "Yangi buyurtma!",
      "body": "Buyurtma #ORD-001 qabul qilindi",
      "data": {"order_id": "ORD-001"}
    }
  }'

# Terminal 2 da ko'rinadi:
# {"id":"n1","type":"order","title":"Yangi buyurtma!","body":"..."}
```

---

## 18-Dars: Production Deployment va Monitoring

### Dockerfile

```dockerfile
# Build stage
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o server ./cmd/server

# Run stage
FROM alpine:3.19
RUN apk --no-cache add ca-certificates
WORKDIR /app
COPY --from=builder /app/server .
EXPOSE 8080
CMD ["./server"]
```

### Monitoring Endpointlari

```go
// Health check va metrikalar
http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
	json.NewEncoder(w).Encode(map[string]interface{}{
		"status":     "ok",
		"clients":    len(hub.clients),
		"rooms":      len(hub.rooms),
		"goroutines": runtime.NumGoroutine(),
		"uptime":     time.Since(startTime).String(),
	})
})

http.HandleFunc("/metrics", func(w http.ResponseWriter, r *http.Request) {
	// Prometheus format
	fmt.Fprintf(w, "ws_clients_total %d\n", len(hub.clients))
	fmt.Fprintf(w, "ws_rooms_total %d\n", len(hub.rooms))
	fmt.Fprintf(w, "ws_goroutines %d\n", runtime.NumGoroutine())

	var m runtime.MemStats
	runtime.ReadMemStats(&m)
	fmt.Fprintf(w, "ws_memory_alloc_bytes %d\n", m.Alloc)
	fmt.Fprintf(w, "ws_memory_sys_bytes %d\n", m.Sys)
})
```

### Production Checklist

```
Xavfsizlik:
  ☐ WSS (TLS) ishlatish — production da DOIMO
  ☐ Origin tekshirish
  ☐ JWT Authentication
  ☐ Rate limiting (ulanish va xabar)
  ☐ Input validation va sanitization
  ☐ Max xabar hajmi cheklash (SetReadLimit)

Barqarorlik:
  ☐ Ping/Pong heartbeat
  ☐ Write deadline
  ☐ Read deadline
  ☐ Graceful shutdown
  ☐ Sekin client boshqaruvi (buffered channel + default)
  ☐ Goroutine leak monitoring
  ☐ Memory monitoring

Masshtab:
  ☐ Redis Pub/Sub (ko'p server uchun)
  ☐ Nginx WebSocket proxy (ip_hash/sticky sessions)
  ☐ Docker Compose / Kubernetes
  ☐ Health check endpoint

Monitoring:
  ☐ Client soni
  ☐ Goroutine soni
  ☐ Xotira ishlatishi
  ☐ Xabar tezligi
  ☐ Xato tezligi
  ☐ Ulanish/uzilish tezligi
```

---

# XULOSA

```
┌──────────────────────────────────────────────────────────────┐
│              WEBSOCKET + GO BILIM XARITASI                   │
│                                                              │
│  Phase 1: Asos                                               │
│    ✓ WebSocket protokoli, handshake, frame                  │
│    ✓ Echo server (gorilla/websocket)                        │
│    ✓ JSON xabarlar va xabar turlari                         │
│                                                              │
│  Phase 2: Ko'p Clientlar                                     │
│    ✓ Broadcast pattern                                      │
│    ✓ Hub pattern (markaziy goroutine)                       │
│    ✓ Read Pump / Write Pump                                 │
│                                                              │
│  Phase 3: Room System                                        │
│    ✓ Room arxitekturasi                                     │
│    ✓ Room join/leave, xona ichida xabar                     │
│    ✓ Private messaging                                      │
│                                                              │
│  Phase 4: Authentication                                     │
│    ✓ JWT bilan WebSocket auth                               │
│    ✓ Origin tekshirish, rate limiting                       │
│                                                              │
│  Phase 5: Production Muammolari                              │
│    ✓ Ping/Pong heartbeat                                    │
│    ✓ Slow client                                            │
│    ✓ Goroutine/memory leak                                  │
│    ✓ Client reconnect                                       │
│                                                              │
│  Phase 6: Scaling                                            │
│    ✓ Redis Pub/Sub horizontal scaling                       │
│    ✓ Nginx load balancer                                    │
│    ✓ Docker Compose multi-server                            │
│                                                              │
│  Phase 7: Real Loyihalar                                     │
│    ✓ Chat Service (to'liq arxitektura)                      │
│    ✓ Notification Service                                   │
│    ✓ Production deployment                                  │
│                                                              │
│  Keyingi qadamlar:                                           │
│    → WebSocket + gRPC streaming                             │
│    → WebSocket + Kafka (real-time events)                   │
│    → WebSocket + ClickHouse (real-time analytics dashboard) │
│    → WebSocket + WebRTC (video/audio)                       │
│    → Kubernetes da WebSocket deployment                     │
└──────────────────────────────────────────────────────────────┘
```

---

> **Eslatma:** Har bir darsni ketma-ket o'rganing. Avval kodni yozing, ishga tushiring, keyin o'zgartiring. Production darajasiga faqat amaliyot orqali yetasiz. Omad!
