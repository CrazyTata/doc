```go

package chat

import (
	"encoding/json"
	"github.com/gorilla/websocket"
	"github.com/zeromicro/go-zero/core/logx"
	"log"
	"net/http"
	"sync"
)

type Client struct {
	conn         *websocket.Conn
	messageQueue chan []byte
	mu           sync.Mutex
	user         string
}

func NewClient(user string, conn *websocket.Conn) *Client {
	return &Client{
		conn:         conn,
		user:         user,
		messageQueue: make(chan []byte, 100),
	}
}

func (c *Client) ReadPump() {
	defer func() {
		c.conn.Close()
	}()

	for {
		mt, message, err := c.conn.ReadMessage()
		if err != nil {
			log.Println("read:", err)
			manager.mu.Lock()
			delete(manager.clients, c.user)
			_ = c.conn.Close()
			manager.mu.Unlock()
			break
		}
		if mt == websocket.TextMessage || mt == websocket.PingMessage {
			c.mu.Lock()
			c.messageQueue <- message
			c.mu.Unlock()
		}
	}
}

func Send(user string, returnMessage []byte, logger logx.Logger) {
	manager.mu.RLock()
	client, exists := manager.clients[user]
	manager.mu.RUnlock()
	if !exists {
		logger.Infof("client not found for user:%s message:%s", user, string(returnMessage))
		return
	}
	client.mu.Lock()
	err := client.conn.WriteMessage(websocket.TextMessage, returnMessage)
	client.mu.Unlock()
	if err != nil {
		logger.Errorf("client.conn.WriteMessage error %s", err.Error())
		// 主动从 manager 中移除无效连接
		manager.mu.Lock()
		delete(manager.clients, user)
		manager.mu.Unlock()
		_ = client.conn.Close()
	}
}

type ClientManager struct {
	clients map[string]*Client
	mu      sync.RWMutex
}

var manager = ClientManager{
	clients: make(map[string]*Client),
}

func ChatWebsocketHandler(svcCtx *svc.ServiceContext) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		conn, err := upgrader.Upgrade(w, r, nil)
		logger := logx.WithContext(r.Context())
		if err != nil {
			logger.Errorf("upgrade:%+v", err)
			return
		}
		// 假设前端会发送一个用户 ID 或会话 ID 用于识别连接
		user := r.URL.Query().Get("user")
		if user == "" {
			logger.Errorf("user is empty:")
			_ = conn.Close()
			return
		}
		client := NewClient(user, conn)
		// 将新的连接存储到连接管理器中
		manager.mu.Lock()
		oldClient, exists := manager.clients[user]
		if exists {
			// 关闭旧的连接
			_ = oldClient.conn.Close()
		}
		manager.clients[user] = client
		manager.mu.Unlock()

		go client.ReadPump()

		l := chat.NewChatWebsocketLogic(r.Context(), svcCtx)

		for {
			select {
			case message := <-client.messageQueue:
				var req types.ChatWebsocketRequest
				_ = json.Unmarshal(message, &req)

				if req.Heartbeat {
					// 处理心跳消息
					err = client.conn.WriteMessage(websocket.PongMessage, []byte(""))
					if err != nil {
						logger.Errorf("write pong message failed:", err)
						manager.mu.Lock()
						delete(manager.clients, user)
						manager.mu.Unlock()
						_ = client.conn.Close()
						return
					}
					returnMessage, _ := json.Marshal(true)
					Send(user, returnMessage, logger)
					continue
				}

				channel := make(chan string, 50)

				go func() {
					defer func() {
						close(channel)
						close(baseInfoCh)
					}()
					res := &types.ChatWebsocketResponse{}
					res, errChat := l.ChatWebsocket(&req, channel, baseInfoCh)
					if errChat != nil {
						logger.Error("ChatWebsocketHandler error :", errChat)
						res.ErrorMessage = response.GetErrorMessage(errChat)
					}

					returnMessage, _ := json.Marshal(res)
					Send(user, returnMessage, logger)
				}()

				var rs []rune
				length := 1
				for {
					s, ok := <-channel
					if !ok {
						if len(rs) > 0 {
							SendSocketMessage(string(rs), req.MessageId)
							rs = []rune{}
						}
						break
					}
					rs = append(rs, []rune(s)...)

					if len(rs) > length {
						SendSocketMessage(string(rs), req.MessageId)
						rs = []rune{}
						if length < 4 {
							length++
						}
					}
				}
			}
		}
	}
}

func SendSocketMessage(message, messageId string) {
	baseReturn := &types.ChatWebsocketResponse{}
	returnMessage, _ := json.Marshal(types.ChatWebsocketResponse{
		Message:    message,
		MessageId:  messageId,
	})
	Send(user, returnMessage, logger)
}

```