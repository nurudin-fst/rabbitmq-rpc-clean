# Alur RabbitMQ RPC dalam Clean Architecture
## **1. Service A (Requester)**
Service A meminta data atau menjalankan proses di Service B.
Alurnya:
1. **Delivery Layer**: Komponen ini bertanggung jawab mengirimkan permintaan ke RabbitMQ (publish ke `rpc_queue`).
2. **Use Case Layer**: Mengontrol logika aplikasi terkait RPC request, seperti membuat permintaan, mengatur timeout, dan memvalidasi respons.
3. **Repository (Infrastructure)**: Mengatur koneksi RabbitMQ, publish pesan, dan menerima respons.

## **2. Service B(Responder)**
Service B mendengarkan request dari queue RabbitMQ, memprosesnya, dan mengirimkan respons.
Alurnya:

1. **Delivery Layer**: Worker yang mendengarkan queue (rpc_queue) dan meneruskan pesan ke Use Case.
2. **Use Case Layer**: Berisi logika pemrosesan request, seperti mengambil data atau menjalankan logika bisnis.
3. **Repository (Infrastructure)**: Berinteraksi dengan database, external API, atau layanan lainnya untuk mendukung proses.
# Struktur Folder
Contoh struktur folder untuk clean architecture dengan RabbitMQ RPC:
```graphql
project/
├── entity/             # Entitas domain (struktur data)
├── usecase/            # Logika bisnis (Use Case)
├── delivery/           
│   ├── rpc_requester/  # Komunikasi RPC untuk mengirim request
│   └── rpc_responder/  # Komunikasi RPC untuk menerima dan merespons
├── repository/         
│   ├── rabbitmq/       # Infrastruktur RabbitMQ
│   └── database/       # Infrastruktur database (opsional)
├── config/             # Konfigurasi aplikasi (RabbitMQ, dll)
└── main.go             # Entry point aplikasi
```
# Implementasi
## 1. Service A (Requester)
Use Case (Requester Logic)
```go
package usecase

import (
	"context"
	"time"
)

type RPCUseCase interface {
	RequestData(ctx context.Context, payload string) (string, error)
}

type rpcUseCase struct {
	rpcRepository RPCRepository
}

func NewRPCUseCase(repo RPCRepository) RPCUseCase {
	return &rpcUseCase{rpcRepository: repo}
}

func (u *rpcUseCase) RequestData(ctx context.Context, payload string) (string, error) {
	// Set timeout untuk request
	ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
	defer cancel()

	// Kirim request ke repository RabbitMQ
	response, err := u.rpcRepository.SendRPCRequest(ctx, payload)
	if err != nil {
		return "", err
	}

	return response, nil
}
```
###Repository (Requester Communication)
```go
package rabbitmq

import (
	"context"
	"fmt"
	"github.com/streadway/amqp"
	"log"
)

type rpcRepository struct {
	conn *amqp.Connection
}

func NewRPCRepository(conn *amqp.Connection) *rpcRepository {
	return &rpcRepository{conn: conn}
}

func (r *rpcRepository) SendRPCRequest(ctx context.Context, payload string) (string, error) {
	ch, err := r.conn.Channel()
	if err != nil {
		return "", fmt.Errorf("failed to open channel: %v", err)
	}
	defer ch.Close()

	// Declare exclusive reply queue
	replyQueue, err := ch.QueueDeclare("", false, true, true, false, nil)
	if err != nil {
		return "", fmt.Errorf("failed to declare reply queue: %v", err)
	}

	// Consume from reply queue
	msgs, err := ch.Consume(replyQueue.Name, "", true, true, false, false, nil)
	if err != nil {
		return "", fmt.Errorf("failed to consume reply queue: %v", err)
	}

	// Generate correlation ID
	correlationID := "unique-id-123" // Replace with UUID generator

	// Publish request
	err = ch.Publish(
		"", "rpc_queue", false, false,
		amqp.Publishing{
			ContentType:   "text/plain",
			CorrelationId: correlationID,
			ReplyTo:       replyQueue.Name,
			Body:          []byte(payload),
		},
	)
	if err != nil {
		return "", fmt.Errorf("failed to publish RPC request: %v", err)
	}

	// Wait for response or timeout
	for {
		select {
		case d := <-msgs:
			if d.CorrelationId == correlationID {
				return string(d.Body), nil
			}
		case <-ctx.Done():
			return "", fmt.Errorf("timeout waiting for response")
		}
	}
}
```
### Delivery Layer (Trigger Use Case)

```go
package delivery

import (
	"context"
	"log"
	"myapp/usecase"
)

type RPCHandler struct {
	useCase usecase.RPCUseCase
}

func NewRPCHandler(u usecase.RPCUseCase) *RPCHandler {
	return &RPCHandler{useCase: u}
}

func (h *RPCHandler) HandleRequest(payload string) {
	response, err := h.useCase.RequestData(context.Background(), payload)
	if err != nil {
		log.Printf("Error processing RPC request: %v", err)
		return
	}
	log.Printf("RPC Response: %s", response)
}
```
## 2. Service B (Responder)
Use Case (Responder Logic)
```go
package usecase

type RPCResponderUseCase interface {
	ProcessRequest(payload string) (string, error)
}

type rpcResponderUseCase struct{}

func NewRPCResponderUseCase() RPCResponderUseCase {
	return &rpcResponderUseCase{}
}

func (u *rpcResponderUseCase) ProcessRequest(payload string) (string, error) {
	// Simulasi logika bisnis
	return "Processed: " + payload, nil
}
```
### Delivery (RPC Worker)

```go
package delivery

import (
	"log"
	"myapp/usecase"

	"github.com/streadway/amqp"
)

type RPCWorker struct {
	useCase usecase.RPCResponderUseCase
}

func NewRPCWorker(u usecase.RPCResponderUseCase) *RPCWorker {
	return &RPCWorker{useCase: u}
}

func (w *RPCWorker) Start(ch *amqp.Channel) {
	msgs, err := ch.Consume("rpc_queue", "", false, false, false, false, nil)
	if err != nil {
		log.Fatalf("Failed to consume queue: %v", err)
	}

	for d := range msgs {
		// Proses request
		response, err := w.useCase.ProcessRequest(string(d.Body))
		if err != nil {
			log.Printf("Failed to process request: %v", err)
			d.Nack(false, false)
			continue
		}

		// Kirim respons
		err = ch.Publish(
			"", d.ReplyTo, false, false,
			amqp.Publishing{
				ContentType:   "text/plain",
				CorrelationId: d.CorrelationId,
				Body:          []byte(response),
			},
		)
		if err != nil {
			log.Printf("Failed to publish response: %v", err)
		}

		d.Ack(false) // Acknowledge pesan
	}
}
```
Ringkasan Alur
1. **Requester (Service A)**:

- Delivery layer menerima trigger dari klien (misalnya HTTP - request).
- Use Case memproses logika bisnis terkait request.
Repository mengirimkan pesan ke RabbitMQ dan menunggu respons.
2. **Responder (Service B)**:

- Worker di delivery layer mendengarkan request dari RabbitMQ.
- Use Case memproses request berdasarkan logika bisnis.
- Repository mengirim respons kembali ke RabbitMQ.
