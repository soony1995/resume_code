## REST API 구현

### 어드민 페이지 API 
    - 미디어 서버 CRUD API 
    - CONFIG 파일 CURD API 
        - ORIGIN, RELAY, EDGE 미디어 서버의 CONFIG 파일(NGINX,JANUS,COTURN)을 역할에 따라 비동기 처리  
### 미디어 서버 API 
    - 비디오룸 
        - 생성, 참가, 유저 강제 퇴장, 레코딩
    - 텍스트룸 
        - 생성, 참가, 레코딩 
    - 스트리밍룸 
        - ORIGIN 서버에서 송출하는 미디어 스트림을 RELAY, EDGE 서버에 전달하는 API
```GO
case "streaming-forward":
    // *models.PubInfo, []models.Accesspoint, *models.Streaming    
    countByRelay, err := w.PQuery.GetCountServersByRole(w.Account, relay) 
    if err != nil {
        ch <- err
        continue
    }
    if countByRelay != 0 { // case of existing relay & relay server status = true
        if v.Role == "relay" {
            var ports = []uint16{args[3].(*models.Streaming).VideoPort - 1, args[3].(*models.Streaming).VideoPort2 - 1,
                args[3].(*models.Streaming).VideoPort3 - 1, args[3].(*models.Streaming).AudioPort - 1}
            edgeServers, err := w.PQuery.ServerByRole(2, v.RelayGp)
            if err != nil {
                ch <- err
                continue
            }
            var wg sync.WaitGroup
            for _, port := range ports {
                wg.Add(1)
                go func(port uint16) {
                    defer wg.Done()
                    destination := fmt.Sprintf("127.0.0.1/%d ", port+1)
                    for _, internalServer := range edgeServers {
                        destination += fmt.Sprintf("%s/%d ", internalServer, port+1)
                    }
                    // forwarding code
                    }
                }(port)
            }
            wg.Wait()
            if err != nil {
                ch <- fmt.Errorf("%s", v.Internal)
                continue
            }
        }
        if v.Role != "edge" {
            go w.execStreamingForward(args[1].(*models.PubInfo), args[3].(*models.Streaming), v.Server, v.Internal, v.Role, ch)
        } else {
            ch <- nil
        }
    else {
        go w.execStreamingForward(args[1].(*models.PubInfo), args[3].(*models.Streaming), v.Server, v.Internal, v.Role, ch)
    }
```


## CUSTOM ERROR CODE 추가를 통해 명확한 에러 정보 제공과 디버깅 용이성 향상

```GO
    //500
	// janus
	NewSessionHandleError = "FAILED_CREATE_NEW_SESSION_HANDLE"
	GetSessionHandleError = "FAILED_GET_SESSION_HANDLE"

	// streaming
	StreamingCreateError       = "FAILED_STREAMING_CREATE"
	StreamingForwardError      = "FAILED_STREAMING_FORWARD"
	StreamingForwardStopError  = "FAILED_STREAMING_FORWARD_STOP"
	StreamingGetConfigureError = "FAILED_GET_STREAMING_CONFIGURE"
	StreamingWatchError        = "FAILED_STREAMING_WATCH"
	StreamingStartError        = "FAILED_STREAMING_START"
	StreamingStopError         = "FAILED_STREAMING_STOP"
```    
    에러 핸들러를 이용해 중복을 최소화 및 유지 보수 용이

```go
    func ErrorHandler(c *fiber.Ctx, status int, errCode string) error {
        errMsg := configs.ErrorMessages[errCode]
        response := &models.HttpResponse{
            Code:     errCode,
            Response: errMsg,
        }
	    return c.Status(status).JSON(response)  
    }

    func SuccessResponse(c *fiber.Ctx, resp interface{}) error {
        response := &models.HttpResponse{
            Code:     "ok",
            Response: resp,
        }
        return c.Status(fiber.StatusOK).JSON(response)
    }
```
    실제 코드
```GO
    err := n.AQuery.DemoUpdateAccount(m)
	if err != nil {
		n.Log.Errorf("Register: n.AQuery.DemoUpdateAccount(%+v): %v", param, err)
		if errors.Is(err, cfg.ErrRegister) {
			return utils.ErrorHandler(c, fiber.StatusBadRequest, cfg.RegisterError)
		} else {
			return utils.ErrorHandler(c, fiber.StatusInternalServerError, cfg.DBError)
		}
	}
```
## DB 튜닝 
> 간단한 SELECT 문에 사용되어 있던 트랜잭션 삭제를 통해 이후에 추가적인 오버헤드 방지
```GO
func (a *SqlQuery) CheckDuplicated(m *models.AdminTableQuery) error {
	var checker bool	    
	result := database.DB.MariaDB.Raw(` 
	select exists(
		select * 
		from admin 
		where admin_id = ?
	) as existence`, m.AdminId).Scan(&checker)
	if result.Error != nil {	
		return result.Error
	}
	// check if the account has already been registered
	if checker {
		return configs.ErrRegister
	}
	return nil
}
```
