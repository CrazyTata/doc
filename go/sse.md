## go 语言中使用SSE流式输出内容

```go
package chat

import (
	"encoding/json"
	"github.com/zeromicro/go-zero/core/logx"
	"github.com/zeromicro/go-zero/rest/httpx"
	"log"
	"net/http"
	"X/common/response"
	"X/common/util"
	"X/internal/logic/frontend/chat"
	"X/internal/svc"
	"X/internal/types"

	"github.com/r3labs/sse/v2"
)

func SendSSEMessage(server *sse.Server, s, messageId, kfId, streamId string, baseInfo any) {

	var (
		state      int64
		relationId string
	)
	if baseInfo != nil {
		switch d := baseInfo.(type) {
		case *types.ChatResponse:
			state = d.State
			relationId = d.RelationId
		case *types.EnquireResponse:
			state = d.State
			relationId = d.RelationId
		case *types.PracticeResponse:
			state = d.State
			relationId = d.RelationId
		default:
			log.Println("Unsupported type:", d)
		}
	}

	returnMessage, _ := json.Marshal(types.ChatSseResponse{
		Message:    s,
		State:      state,
		RelationId: relationId,
	})
	server.Publish(streamId, &sse.Event{
		Data: returnMessage,
	})
}

func ChatSseHandler(svcCtx *svc.ServiceContext) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		logger := logx.WithContext(r.Context())
		streamId := r.URL.Query().Get("stream")
		if streamId == "" {
			logger.Errorf("stream is empty ")
			return
		}

		server := sse.New()
		server.CreateStream(streamId)

		var req types.ChatSseRequest
		if err := httpx.Parse(r, &req); err != nil {
			httpx.ErrorCtx(r.Context(), w, err)
			return
		}

		l := chat.NewChatSseLogic(r.Context(), svcCtx)

		channel := make(chan string, 50)
		baseInfoCh := make(chan any, 1)

		go func() {
			server.ServeHTTP(w, r)
		}()
		go func() {
			defer func() {
				close(channel)
				close(baseInfoCh)
			}()

			res := &types.ChatSseResponse{}
			var errChat error

			res, errChat = l.ChatSse(&req, channel, baseInfoCh)
			if errChat != nil {
				logger.Error("ChatSseHandler error:", errChat)
			}

			res.ErrorMessage = response.GetErrorMessage(errChat)
			returnMessage, _ := json.Marshal(res)
			server.Publish(streamId, &sse.Event{
				Data: returnMessage,
			})
		}()
		baseInfo := <-baseInfoCh
		var rs []rune
		length := 4
		for {
			s, ok := <-channel
			if !ok {
				if len(rs) > 0 {
					SendSSEMessage(server, string(rs), req.MessageId, req.OpenKfID, streamId, baseInfo)
					rs = []rune{}
				}
				break
			}
			rs = append(rs, []rune(s)...)

			if len(rs) > length {
				SendSSEMessage(server, string(rs), req.MessageId, req.OpenKfID, streamId, baseInfo)
				rs = []rune{}
				if length < 4 {
					length++
				}
			}
		}
	}
}

```