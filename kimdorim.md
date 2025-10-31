# 도림 9번

```mermaid
sequenceDiagram
    actor User as 사용자
    participant UI as Frontend
    participant Controller as ChatRoomController
    participant Service as ChatRoomService
    participant ChatRoomRepo as ChatRoomRepository
    participant UserRepo as UserRepository
    participant PropertyRepo as PropertyRepository
    participant DB as Database

    User->>UI: 매물 상세 페이지에서 '대화하기' 버튼 클릭
    
    UI->>UI: 현재 로그인 상태 확인
    
    alt 로그인하지 않은 상태
        UI->>UI: 로그인 페이지로 이동 또는<br/>로그인 팝업 표시
        UI-->>User: "로그인이 필요합니다" 메시지
    else 로그인한 상태
        UI->>Controller: GET /api/chat/room<br/>?propertyId={propertyId}<br/>&otherId={otherId}
        
        Controller->>Controller: JWT 토큰 검증
        Controller->>Controller: 현재 userId 추출
        
        Controller->>Service: findOrCreateChatRoom(propertyId, userId, otherId)
        
        Service->>PropertyRepo: findById(propertyId)
        PropertyRepo->>DB: SELECT properties WHERE id = ?
        DB-->>PropertyRepo: Property
        PropertyRepo-->>Service: Property
        
        alt Property 존재하지 않음
            Service-->>Controller: 오류 응답<br/>("존재하지 않는 매물입니다")
            Controller-->>UI: 400 Bad Request
        else Property 존재
            Service->>UserRepo: findById(otherId)
            UserRepo->>DB: SELECT users WHERE id = ?
            DB-->>UserRepo: User (상대방)
            UserRepo-->>Service: User
            
            alt 상대방 사용자가 없음
                Service-->>Controller: 오류 응답<br/>("사용자를 찾을 수 없습니다")
                Controller-->>UI: 400 Bad Request
            else 상대방 사용자 존재
                Service->>Service: 매물과 참여자 조합 확인
                
                Service->>ChatRoomRepo: findExistingRoom(propertyId, userId, otherId)
                ChatRoomRepo->>DB: SELECT chat_rooms<br/>WHERE property_id = ?<br/>AND ((user1_id = ? AND user2_id = ?)<br/>OR (user1_id = ? AND user2_id = ?))
                DB-->>ChatRoomRepo: 기존 ChatRoom (있을 수도, 없을 수도)
                ChatRoomRepo-->>Service: ChatRoom 또는 null
                
                alt 기존 채팅방이 존재함
                    Service->>Service: 기존 채팅방 ID 사용
                    Service-->>Controller: ChatRoom 반환
                else 기존 채팅방이 없음
                    alt 동일한 참여자 조합 중복 생성 방지
                        Service->>Service: 현재 userId = user1<br/>otherId = user2로 설정<br/>(순서 정규화)
                        
                        Service->>Service: ChatRoom 엔티티 생성<br/>(propertyId, user1, user2, user3=null)
                        
                        Service->>ChatRoomRepo: save(ChatRoom)
                        ChatRoomRepo->>DB: INSERT INTO chat_rooms<br/>(property_id, user1_id, user2_id, created_at)
                        DB-->>ChatRoomRepo: 새로운 roomId
                        ChatRoomRepo-->>Service: ChatRoom (생성됨)
                        
                        Service->>Service: 생성 시각(createdAt) 자동 설정<br/>(@PrePersist)
                        
                        Service-->>Controller: ChatRoom 반환<br/>{roomId, propertyId, participants, createdAt}
                    end
                end
            end
        end
        
        Controller-->>UI: 200 OK<br/>{roomId, propertyId,<br/>participants: [user1, user2],<br/>createdAt}
        
        UI->>UI: 채팅 메시지 목록 로드 요청<br/>(roomId 사용)
        
        UI->>UI: 채팅방 화면으로 전환
        
        UI-->>User: 채팅방 진입<br/>메시지 목록 표시
    end

```

# 도림 10번

```mermaid
sequenceDiagram
    actor User as 사용자
    participant UI as 클라이언트
    participant Controller as ChatController
    participant Service as ChatService
    participant Repository as ChatRepository
    participant DB as Database

    User->>UI: 채팅방 입장 (room_id 있음)
    UI->>UI: 주기적 메시지 폴링 시작 (3~5초 간격)

    User->>UI: 채팅 입력창에 메시지 작성
    UI->>UI: 메시지 입력 (텍스트)
    
    User->>UI: 전송 버튼 클릭
    UI->>Controller: POST /api/chat/{roomId}/messages (message content)
    
    Controller->>Controller: 로그인 토큰 검증
    alt 토큰 없음
        Controller-->>UI: 401 Unauthorized
        UI-->>User: "인증이 필요합니다" 오류
    else 토큰 유효
        Controller->>Service: sendMessage(roomId, userId, content)
        
        Service->>Service: 메시지 내용 검증 (비어있지 않은지, 길이 확인)
        alt 메시지 비어있음 또는 길이 초과
            Service-->>Controller: ValidationException
            Controller-->>UI: 400 Bad Request
            UI-->>User: "메시지를 입력해주세요" 또는 "너무 깁니다" 오류
        else 메시지 유효
            Service->>Service: ChatMessage 엔티티 생성
            Service->>Service: sender_id, room_id, content, timestamp 설정
            Service->>Service: is_read = false 설정
            
            Service->>Repository: saveMessage(chatMessage)
            Repository->>DB: INSERT INTO chat_messages (room_id, sender_id, content, created_at, is_read)
            DB-->>Repository: 저장 완료 (message_id 반환)
            Repository-->>Service: ChatMessage Entity (저장됨)
            
            alt DB 저장 실패
                Service->>Service: 재시도 로직 (최대 3회)
                alt 재시도 3회 모두 실패
                    Service-->>Controller: DataAccessException
                    Controller-->>UI: 500 Internal Server Error
                    UI-->>User: "메시지 전송 실패, 재시도 버튼"
                else 재시도 중 성공
                    Service-->>Controller: 201 Created + ChatMessage
                    Controller-->>UI: 201 Created
                    UI->>UI: 입력창 초기화
                    UI->>UI: 메시지를 로컬 목록에 즉시 추가 (Optimistic Update)
                    UI-->>User: 메시지 표시 (검은색, 자신이 보낸 메시지)
                end
            else DB 저장 성공
                Service-->>Controller: 201 Created + ChatMessage
                Controller-->>UI: 201 Created
                UI->>UI: 입력창 초기화
                UI->>UI: 메시지를 로컬 목록에 즉시 추가 (Optimistic Update)
                UI-->>User: 메시지 표시 (검은색, 자신이 보낸 메시지)
            end
        end
    end

    par 주기적 메시지 폴링 (3~5초)
        UI->>Controller: GET /api/chat/{roomId}/messages?after={lastMessageId}
        
        Controller->>Service: getMessages(roomId, afterMessageId)
        
        Service->>Repository: findByRoomIdAfter(roomId, lastMessageId)
        Repository->>DB: SELECT * FROM chat_messages WHERE room_id = ? AND id > ? ORDER BY created_at ASC
        DB-->>Repository: 새 메시지 목록
        Repository-->>Service: List<ChatMessage>
        
        alt 새 메시지 있음
            Service-->>Controller: 200 OK + 새 메시지 목록
            Controller-->>UI: 200 OK
            UI->>UI: 새 메시지 목록 병합
            
            alt 다른 사용자의 메시지
                UI->>UI: 메시지 표시 (회색, 상대방 메시지)
                UI->>UI: 메시지 읽음 상태 업데이트 (처음엔 미읽)
                UI-->>User: 새로운 메시지 알림 표시
            else 자신의 메시지 (서버에서 반환)
                UI->>UI: Optimistic Update와 병합
            end
        else 새 메시지 없음
            Service-->>Controller: 200 OK + 빈 목록
            Controller-->>UI: 200 OK
        end
    end

    User->>UI: 채팅 화면 스크롤 (새 메시지 보임)
    UI->>UI: 화면에 보이는 메시지 감지
    UI->>Controller: PATCH /api/chat/messages/mark-as-read (message_ids)
    
    Controller->>Service: markAsRead(messageIds, userId)
    
    Service->>Repository: updateIsRead(messageIds)
    Repository->>DB: UPDATE chat_messages SET is_read = true WHERE id IN (...)
    DB-->>Repository: 업데이트 완료
    
    Service-->>Controller: 200 OK
    Controller-->>UI: 200 OK
    UI->>UI: 메시지 표시 변경 (회색 → 검은색 체크마크)

    alt 네트워크 오류
        Repository->>DB: Exception
        Repository-->>Service: NetworkException
        Service-->>Controller: 500 Internal Server Error
        Controller-->>UI: 500
        UI->>UI: "연결 오류, 재시도 중..." 표시
        UI->>UI: 자동 재시도 (지수 백오프)
    end

```

# 도림 11번

```mermaid
sequenceDiagram
    actor User as 사용자
    participant UI as Frontend
    participant Controller as ChatMessageController
    participant Service as ChatMessageService
    participant ChatMessageRepo as ChatMessageRepository
    participant ChatRoomRepo as ChatRoomRepository
    participant DB as Database

    User->>UI: 채팅방에 재입장<br/>또는 화면 로드
    
    alt 채팅방 초기 진입
        UI->>Controller: GET /api/chat/messages<br/>?roomId={roomId}<br/>&limit=50
        
        Controller->>Controller: JWT 토큰 검증
        Controller->>Controller: 현재 userId 추출
        
        Controller->>Service: getLatestMessages(roomId, limit=50)
        
        Service->>ChatRoomRepo: findById(roomId)
        ChatRoomRepo->>DB: SELECT chat_rooms WHERE id = ?
        DB-->>ChatRoomRepo: ChatRoom
        ChatRoomRepo-->>Service: ChatRoom
        
        alt ChatRoom 존재하지 않음
            Service-->>Controller: 오류 응답<br/>("존재하지 않는 채팅방입니다")
            Controller-->>UI: 400 Bad Request
        else ChatRoom 존재
            Service->>Service: 권한 확인<br/>(사용자가 채팅방 참여자인지)
            
            alt 권한 없음
                Service-->>Controller: 오류 응답<br/>("채팅방 접근 권한이 없습니다")
                Controller-->>UI: 403 Forbidden
            else 권한 있음
                Service->>ChatMessageRepo: findLatestMessages(roomId, limit=50)
                ChatMessageRepo->>DB: SELECT chat_messages<br/>WHERE room_id = ?<br/>ORDER BY created_at DESC<br/>LIMIT 50
                DB-->>ChatMessageRepo: 최신 50개 메시지
                ChatMessageRepo-->>Service: List<ChatMessage>
                
                Service->>Service: 메시지 정렬<br/>(created_at 오름차순)
                
                Service->>Service: 각 메시지에 발신자 정보 추가<br/>(sender name, avatar)
                
                Service-->>Controller: 메시지 목록 응답
            end
        end
        
        Controller-->>UI: 200 OK<br/>[메시지 목록]
        
        UI->>UI: 최신 메시지부터<br/>오름차순으로 렌더링
        UI-->>User: 메시지 목록 표시
    end
    
    alt 사용자가 상단으로 스크롤<br/>(이전 메시지 요청)
        User->>UI: 스크롤 상단 도달
        
        UI->>UI: 현재 표시 메시지 중<br/>가장 오래된 메시지 ID 계산
        
        UI->>UI: 로딩 표시기 표시
        
        UI->>Controller: GET /api/chat/messages<br/>?roomId={roomId}<br/>&beforeMessageId={messageId}<br/>&limit=50
        
        Controller->>Controller: 요청 검증
        
        Controller->>Service: getMessagesBefore(roomId, messageId, limit=50)
        
        Service->>ChatRoomRepo: 권한 확인
        ChatRoomRepo-->>Service: 권한 검증 완료
        
        alt 권한 없음
            Service-->>Controller: 403 Forbidden
        else 권한 있음
            Service->>ChatMessageRepo: findMessagesBefore(roomId, messageId, limit=50)
            ChatMessageRepo->>DB: SELECT chat_messages<br/>WHERE room_id = ?<br/>AND id < ?<br/>ORDER BY created_at DESC<br/>LIMIT 50
            DB-->>ChatMessageRepo: 이전 50개 메시지<br/>(최신순)
            ChatMessageRepo-->>Service: List<ChatMessage>
            
            alt 조회된 메시지가 없음
                Service->>Service: 빈 목록 반환
                Service-->>Controller: 빈 메시지 목록
                
                Controller-->>UI: 200 OK<br/>[]
                
                UI->>UI: "더 이상 불러올 메시지가 없습니다."<br/>안내 표시
            else 메시지 있음
                Service->>Service: 메시지 정렬<br/>(created_at 오름차순)
                
                Service->>Service: 발신자 정보 추가
                
                Service-->>Controller: 메시지 목록 응답
                
                Controller-->>UI: 200 OK<br/>[이전 메시지 목록]
                
                UI->>UI: 로딩 표시기 숨김
                
                UI->>UI: 기존 메시지 앞쪽에<br/>새 메시지들 병합 추가<br/>(시간순 정렬 유지)
                
                UI->>UI: 스크롤 위치 조정<br/>(추가된 메시지 끝부분으로)
                
                UI-->>User: 이전 메시지 표시
            end
        end
    end
    
    alt 네트워크/DB 오류 발생
        Service->>Service: 오류 감지
        
        Service-->>Controller: 오류 응답
        Controller-->>UI: 400/500 에러
        
        UI->>UI: 로딩 표시기 숨김
        UI->>UI: "메시지를 불러올 수 없습니다."<br/>오류 메시지 표시
        UI->>UI: "재시도" 버튼 표시
        
        User->>UI: "재시도" 버튼 클릭
        UI->>UI: 이전 요청 재시도
    end

```

# 도림 12번

```mermaid
sequenceDiagram
    actor User as 사용자
    participant UI as 클라이언트
    participant Controller as ChatController
    participant Service as ChatService
    participant Repository as ChatRepository
    participant DB as Database

    User->>UI: 채팅방 화면에서 메시지 확인
    UI->>UI: 메시지 노출 감지 (Intersection Observer)
    
    alt 메시지가 화면에 보임
        UI->>UI: 보이는 메시지들의 ID 수집
        UI->>Controller: PATCH /api/chat/messages/mark-read (message_ids)
        
        Controller->>Controller: 로그인 토큰 검증
        alt 토큰 없음
            Controller-->>UI: 401 Unauthorized
        else 토큰 유효
            Controller->>Service: markAsRead(messageIds, userId)
            
            Service->>Service: 메시지 ID 유효성 검증
            Service->>Service: 채팅방 권한 확인
            
            Service->>Repository: updateIsRead(messageIds)
            Repository->>DB: UPDATE chat_messages SET is_read = true WHERE id IN (...)
            DB-->>Repository: 업데이트 완료
            Repository-->>Service: 업데이트된 메시지 개수
            
            alt 업데이트 성공
                Service-->>Controller: 200 OK
                Controller-->>UI: 200 OK
                UI->>UI: 메시지 표시 상태 갱신 (읽음 표시)
            else 업데이트 실패
                Service-->>Controller: 500 Internal Server Error
                Controller-->>UI: 500
                UI-->>User: "읽음 처리 실패, 재시도" 메시지
            end
        end
    end

    User->>UI: "모두 읽음" 버튼 클릭
    UI->>Controller: PATCH /api/chat/{roomId}/mark-all-read
    
    Controller->>Service: markAllAsRead(roomId, userId)
    
    Service->>Service: 채팅방 권한 확인
    Service->>Repository: updateAllReadByRoom(roomId, userId)
    Repository->>DB: UPDATE chat_messages SET is_read = true WHERE room_id = ? AND is_read = false
    DB-->>Repository: 업데이트 완료
    Repository-->>Service: 업데이트된 메시지 개수
    
    Service->>Service: 사용자의 미읽음 개수 = 0 설정
    Service-->>Controller: 200 OK
    Controller-->>UI: 200 OK
    UI->>UI: 모든 메시지 읽음 표시
    UI->>UI: 미읽음 배지 제거 (알림 아이콘)
    UI-->>User: "모든 메시지를 읽음으로 표시했습니다"

    par 주기적 동기화
        UI->>Controller: GET /api/chat/{roomId}/messages (폴링)
        
        Controller->>Service: getMessages(roomId, afterMessageId)
        Service->>Repository: findMessages(roomId)
        Repository->>DB: SELECT * FROM chat_messages WHERE room_id = ? ORDER BY created_at
        DB-->>Repository: 메시지 목록
        Repository-->>Service: List<ChatMessage> (is_read 상태 포함)
        
        Service-->>Controller: 200 OK + 메시지 목록
        Controller-->>UI: 200 OK
        UI->>UI: 메시지 목록 병합 (is_read 상태 반영)
        UI->>UI: 읽음 표시 변경 (회색 → 검은색 체크마크)
    end

    par 사용자가 다른 기기에서 접속
        UI1->>UI1: 기기1에서 읽음 처리
        UI1->>Controller: PATCH /api/chat/messages/mark-read
        Controller->>Service: updateIsRead (메시지1, 메시지2)
        Service->>DB: UPDATE is_read = true
        
        UI2->>UI2: 기기2에서 주기적 폴링
        UI2->>Controller: GET /api/chat/{roomId}/messages
        Controller->>Service: getMessages
        Service->>DB: SELECT * (is_read = true 상태 반영)
        DB-->>Service: 메시지 목록
        Service-->>Controller: 200 OK
        Controller-->>UI2: 200 OK
        UI2->>UI2: 읽음 상태 동기화 (기기1에서의 변경 반영)
    end

    alt DB 오류 또는 네트워크 실패
        DB-->>Repository: Exception
        Repository-->>Service: DatabaseException
        Service->>Service: 로컬 상태 보관 (재시도 큐에 추가)
        Service-->>Controller: 500 Internal Server Error
        Controller-->>UI: 500
        UI->>UI: "읽음 처리 실패, 재시도 중..." 표시
        UI->>UI: 자동 재시도 (지수 백오프: 1초, 2초, 5초)
    end

```

# 도림 13번

```mermaid
sequenceDiagram
    actor User as 사용자
    participant UI as Frontend
    participant Controller as ChatMessageController
    participant Service as ChatMessageService
    participant ChatRoomRepo as ChatRoomRepository
    participant ChatMessageRepo as ChatMessageRepository
    participant UserChatStateRepo as UserChatStateRepository
    participant DB as Database

    User->>UI: 채팅방에 재입장
    
    UI->>UI: 현재 로그인 상태 확인
    
    alt 로그인 상태 아님
        UI->>UI: 로그인 페이지로 리다이렉트
        UI-->>User: "로그인이 필요합니다"
    else 로그인 상태
        UI->>Controller: GET /api/chat/resume<br/>?roomId={roomId}
        
        Controller->>Controller: JWT 토큰 검증
        Controller->>Controller: 현재 userId 추출
        
        Controller->>Service: getResumePoint(roomId, userId)
        
        Service->>ChatRoomRepo: findById(roomId)
        ChatRoomRepo->>DB: SELECT chat_rooms WHERE id = ?
        DB-->>ChatRoomRepo: ChatRoom
        ChatRoomRepo-->>Service: ChatRoom
        
        alt ChatRoom 존재하지 않음
            Service-->>Controller: 오류 응답
            Controller-->>UI: 404 Not Found
        else ChatRoom 존재
            Service->>Service: 권한 확인<br/>(사용자가 참여자인지)
            
            alt 권한 없음
                Service-->>Controller: 403 Forbidden
                Controller-->>UI: 403 오류
            else 권한 있음
                Service->>UserChatStateRepo: findLastReadMessage(roomId, userId)
                UserChatStateRepo->>DB: SELECT user_chat_states<br/>WHERE room_id = ? AND user_id = ?
                DB-->>UserChatStateRepo: 마지막 읽음 포인터<br/>(message_id 또는 null)
                UserChatStateRepo-->>Service: last_read_message_id
                
                alt 마지막 읽음 포인터가 없음<br/>(첫 입장 또는 초기화)
                    Service->>Service: 기본값 사용<br/>(최근 N개 메시지)
                    Service->>ChatMessageRepo: findLatestMessages(roomId, limit=50)
                    ChatMessageRepo->>DB: SELECT chat_messages<br/>WHERE room_id = ?<br/>ORDER BY created_at DESC<br/>LIMIT 50
                    DB-->>ChatMessageRepo: 최신 50개 메시지
                    ChatMessageRepo-->>Service: List<ChatMessage>
                else 마지막 읽음 포인터 존재
                    Service->>ChatMessageRepo: findMessagesAfter(roomId, messageId)
                    ChatMessageRepo->>DB: SELECT chat_messages<br/>WHERE room_id = ?<br/>AND id > ?<br/>ORDER BY created_at ASC
                    DB-->>ChatMessageRepo: 포인터 이후 메시지
                    ChatMessageRepo-->>Service: List<ChatMessage>
                    
                    alt 포인터 이후 새 메시지 없음
                        Service->>Service: 포인터 주변<br/>메시지 조회<br/>(context를 위해 이전 메시지 포함)
                        Service->>ChatMessageRepo: findContextMessages(roomId, messageId, before=10, after=5)
                        ChatMessageRepo->>DB: 컨텍스트 메시지 조회
                        DB-->>ChatMessageRepo: 컨텍스트
                        ChatMessageRepo-->>Service: List<ChatMessage>
                    end
                end
                
                Service->>Service: 메시지 정렬<br/>(created_at 오름차순)
                
                Service->>Service: 각 메시지에 발신자 정보 추가
                
                Service-->>Controller: 메시지 목록 및<br/>마지막 읽음 포인터 응답
            end
        end
        
        Controller-->>UI: 200 OK<br/>{messages, lastReadMessageId, unreadCount}
        
        UI->>UI: 로딩 표시기 표시
        
        alt 새로운 메시지 있음 (포인터 이후)
            UI->>UI: 새 메시지부터 우선 렌더링<br/>(강조 또는 구분)
        else 새로운 메시지 없음
            UI->>UI: 포인터 주변 컨텍스트<br/>메시지 렌더링
            UI->>UI: 포인터 위치로 자동 스크롤
        end
        
        UI->>UI: 로딩 표시기 숨김
        
        UI->>UI: 화면 노출 범위 계산<br/>(뷰포트 내 최하단 메시지)
        
        alt 새 메시지 표시됨 (뷰포트 내)
            UI->>Controller: PUT /api/chat/read-state<br/>?roomId={roomId}<br/>&lastMessageId={messageId}
            
            Controller->>Controller: 요청 검증
            
            Controller->>Service: updateReadState(roomId, userId, lastMessageId)
            
            Service->>UserChatStateRepo: findById(roomId, userId)
            UserChatStateRepo->>DB: SELECT FROM user_chat_states
            DB-->>UserChatStateRepo: 기존 상태
            UserChatStateRepo-->>Service: UserChatState
            
            alt 기존 상태 없음
                Service->>Service: 새로운 UserChatState 생성
                Service->>UserChatStateRepo: save(newState)
                UserChatStateRepo->>DB: INSERT INTO user_chat_states
                DB-->>UserChatStateRepo: 저장 완료
            else 기존 상태 있음
                Service->>Service: 단조 증가 규칙 확인<br/>(새 messageId > 기존)
                
                alt 새 포인터가 더 오래됨
                    Service->>Service: 기존 포인터 유지<br/>(무시)
                else 새 포인터가 더 최신
                    Service->>UserChatStateRepo: update(lastMessageId)
                    UserChatStateRepo->>DB: UPDATE user_chat_states
                    DB-->>UserChatStateRepo: 업데이트 완료
                end
            end
            
            Service-->>Controller: 성공 응답
            Controller-->>UI: 200 OK
        end
        
        UI-->>User: 채팅방 표시<br/>이어서 보기 완료
    end

```

# 도림 19번

```mermaid
sequenceDiagram
    actor User as 사용자
    participant UI as 클라이언트
    participant Controller as PropertyController
    participant Service as PropertyService
    participant Repository as PropertyRepository
    participant DB as Database

    User->>UI: "전체 매물 보기" 메뉴 클릭
    UI->>Controller: GET /api/properties?page=0&size=20
    
    Controller->>Controller: 로그인 토큰 검증
    alt 토큰 없음
        Controller-->>UI: 401 Unauthorized
        UI-->>User: 로그인 화면으로 이동
    else 토큰 유효
        Controller->>Service: getPropertyList(page, size)
        
        Service->>Repository: findAll(pageable)
        Repository->>DB: SELECT * FROM properties WHERE status = AVAILABLE ORDER BY created_at DESC
        DB-->>Repository: 매물 목록 (페이지네이션)
        Repository-->>Service: Page<Property>
        
        alt 조회 성공
            Service->>Service: 매물 엔티티를 PropertyDto로 변환
            Service-->>Controller: 200 OK + PropertyPage
            
            Controller-->>UI: 200 OK
            UI->>UI: 매물 목록 렌더링
            UI->>UI: 페이지네이션 컨트롤 표시
            UI-->>User: 전체 매물 목록 표시 (최신순)
        else 조회 실패
            Service-->>Controller: 500 Internal Server Error
            Controller-->>UI: 500
            UI-->>User: "목록을 불러올 수 없습니다" + 재시도 버튼
        end
    end

    User->>UI: 필터 설정 (상태, 유형, 가격, 지역)
    UI->>UI: 필터 폼 표시
    
    User->>UI: 필터 적용 버튼 클릭
    UI->>Controller: GET /api/properties?status=AVAILABLE&type=JEONSE&priceMin=100&priceMax=500&region=gangnam&page=0
    
    Controller->>Service: getPropertyList(pageable, filters)
    
    Service->>Repository: findByFilters(filters, pageable)
    Repository->>DB: SELECT * FROM properties WHERE status = ? AND type = ? AND price BETWEEN ? AND ? AND region LIKE ? ORDER BY created_at DESC
    DB-->>Repository: 필터된 매물 목록
    Repository-->>Service: Page<Property> (필터링됨)
    
    alt 결과 있음
        Service-->>Controller: 200 OK + PropertyPage
        Controller-->>UI: 200 OK
        UI->>UI: 필터된 매물 목록 렌더링
        UI-->>User: 필터된 매물 표시
    else 결과 없음
        Service-->>Controller: 200 OK + 빈 페이지
        Controller-->>UI: 200 OK
        UI-->>User: "조건에 맞는 매물이 없습니다" 메시지
    end

    User->>UI: 특정 매물 카드 클릭
    UI->>Controller: GET /api/properties/{propertyId}
    
    Controller->>Service: getPropertyDetail(propertyId)
    Service->>Repository: findById(propertyId)
    Repository->>DB: SELECT * FROM properties WHERE id = ?
    DB-->>Repository: 매물 정보
    Repository-->>Service: Property Entity
    
    Service-->>Controller: 200 OK + PropertyDetailDto
    Controller-->>UI: 200 OK
    UI->>UI: 매물 상세 페이지로 이동
    UI-->>User: 매물 상세 정보 표시

    User->>UI: 페이지 네비게이션 (다음 페이지)
    UI->>Controller: GET /api/properties?page=1&size=20
    
    Controller->>Service: getPropertyList(page=1, size=20)
    Service->>Repository: findAll(pageable)
    Repository->>DB: SELECT * FROM properties ORDER BY created_at DESC LIMIT 20 OFFSET 20
    DB-->>Repository: 다음 페이지 매물 (20-40)
    Repository-->>Service: Page<Property>
    
    Service-->>Controller: 200 OK
    Controller-->>UI: 200 OK
    UI->>UI: 다음 페이지 매물 렌더링
    UI-->>User: 다음 매물 목록 표시

    alt 정렬 변경 (최신순 → 가격순)
        User->>UI: 정렬 옵션 선택
        UI->>Controller: GET /api/properties?sort=price&page=0
        
        Controller->>Service: getPropertyList(pageable, sort=price)
        Service->>Repository: findAll(pageable with sort)
        Repository->>DB: SELECT * FROM properties ORDER BY price ASC
        DB-->>Repository: 정렬된 매물
        Repository-->>Service: Page<Property>
        
        Service-->>Controller: 200 OK
        UI-->>User: 가격순 정렬 매물 표시
    end

    alt DB 또는 네트워크 오류
        DB-->>Repository: Exception
        Repository-->>Service: Exception
        Service-->>Controller: 500 Internal Server Error
        Controller-->>UI: 500
        UI-->>User: "오류 발생, 재시도 버튼"
    end

```

# 도림 20번

```mermaid
sequenceDiagram
    actor User as 사용자
    participant UI as Frontend
    participant Controller as PropertyController
    participant Service as PropertyService
    participant PropertyRepo as PropertyRepository
    participant UserRepo as UserRepository
    participant DB as Database

    User->>UI: "내 매물" 메뉴 선택
    
    UI->>UI: 현재 로그인 상태 확인
    
    alt 로그인하지 않은 상태
        UI->>UI: 로그인 페이지로 리다이렉트
        UI-->>User: "로그인이 필요합니다"
    else 로그인한 상태
        UI->>Controller: GET /api/user/properties<br/>?userId={userId}<br/>&page={page}<br/>&sort=-createdAt
        
        Controller->>Controller: JWT 토큰 검증
        Controller->>Controller: 현재 userId 추출
        
        Controller->>Service: getMyProperties(userId, page, sort)
        
        Service->>UserRepo: findById(userId)
        UserRepo->>DB: SELECT users WHERE id = ?
        DB-->>UserRepo: User
        UserRepo-->>Service: User
        
        alt User 존재하지 않음
            Service-->>Controller: 오류 응답<br/>("사용자를 찾을 수 없습니다")
            Controller-->>UI: 404 Not Found
        else User 존재
            Service->>PropertyRepo: findAllByOwnerId(userId, pageable)
            PropertyRepo->>DB: SELECT properties<br/>WHERE owner_id = ?<br/>ORDER BY created_at DESC<br/>LIMIT 20 OFFSET ?
            DB-->>PropertyRepo: 사용자의 매물 목록
            PropertyRepo-->>Service: Page<Property>
            
            Service->>Service: 각 Property에 거래 정보 추가<br/>(PropertyOffer 통합)
            
            Service->>Service: Property 목록 변환<br/>(PropertyResponseDto)
            
            Service-->>Controller: 매물 목록 응답
        end
        
        Controller-->>UI: 200 OK<br/>[매물 목록]
        
        UI->>UI: 매물 목록 렌더링<br/>(제목, 주소, 가격, 상태)
        UI->>UI: 필터/정렬 옵션 표시<br/>(상태, 거래유형, 가격순)
        UI-->>User: 내 매물 목록 표시
        
        alt 사용자가 매물 필터 적용
            User->>UI: 필터 조건 선택<br/>(상태, 거래유형, 가격범위)
            
            UI->>Controller: GET /api/user/properties<br/>?userId={userId}<br/>&status={status}<br/>&dealType={dealType}<br/>&minPrice={minPrice}<br/>&maxPrice={maxPrice}
            
            Controller->>Service: getMyPropertiesFiltered(userId, filters, page)
            
            Service->>PropertyRepo: findAllByOwnerIdAndFilters(userId, filters, pageable)
            PropertyRepo->>DB: SELECT properties<br/>WHERE owner_id = ? AND<br/>status = ? AND<br/>price BETWEEN ? AND ?<br/>ORDER BY created_at DESC
            DB-->>PropertyRepo: 필터링된 매물 목록
            PropertyRepo-->>Service: List<Property>
            
            Service-->>Controller: 필터링된 목록 응답
            Controller-->>UI: 200 OK<br/>[필터링된 매물 목록]
            
            UI->>UI: 필터 적용된 목록 표시
        end
        
        alt 사용자가 특정 매물 선택
            User->>UI: 매물 클릭
            
            UI->>UI: 매물 상세 정보 표시<br/>(제목, 주소, 가격, 거래조건,<br/>이미지, 설명)
            
            UI->>UI: 관리 기능 활성화<br/>("수정", "상태 변경", "삭제" 버튼)
        end
        
        alt 사용자가 매물 상태 변경
            User->>UI: "상태 변경" 버튼 클릭<br/>(AVAILABLE→PENDING→SOLD)
            
            UI->>UI: 상태 변경 옵션 표시
            
            User->>UI: 새로운 상태 선택
            
            UI->>Controller: PUT /api/user/properties/{propertyId}<br/>{status: newStatus}
            
            Controller->>Controller: JWT 토큰 검증
            Controller->>Controller: userId 추출
            
            Controller->>Service: updatePropertyStatus(propertyId, userId, newStatus)
            
            Service->>PropertyRepo: findById(propertyId)
            PropertyRepo->>DB: SELECT properties WHERE id = ?
            DB-->>PropertyRepo: Property
            PropertyRepo-->>Service: Property
            
            alt 매물 존재하지 않음
                Service-->>Controller: 오류 응답<br/>("존재하지 않는 매물입니다")
                Controller-->>UI: 404 Not Found
            else 매물 존재
                Service->>Service: 권한 확인<br/>(owner_id = userId?)
                
                alt 권한 없음
                    Service-->>Controller: 오류 응답<br/>("수정 권한이 없습니다")
                    Controller-->>UI: 403 Forbidden
                else 권한 있음
                    Service->>Service: 상태 변경<br/>(status = newStatus)
                    Service->>Service: updatedAt 갱신<br/>(@PreUpdate)
                    
                    Service->>PropertyRepo: save(Property)
                    PropertyRepo->>DB: UPDATE properties<br/>SET status = ?, updated_at = NOW()
                    DB-->>PropertyRepo: 업데이트 완료
                    PropertyRepo-->>Service: Property (업데이트됨)
                    
                    Service-->>Controller: 업데이트 완료 응답
                end
            end
            
            Controller-->>UI: 200 OK<br/>{매물 정보}
            
            UI->>UI: 성공 메시지 표시
            UI->>UI: 목록 새로고침<br/>업데이트된 상태 표시
            UI-->>User: "상태가 변경되었습니다"
        end
        
        alt 사용자가 매물 정보 수정
            User->>UI: "수정" 버튼 클릭
            
            UI->>UI: 수정 폼 로드<br/>(기존 정보 표시)
            
            User->>UI: 매물 정보 수정<br/>(가격, 설명, 이미지 등)
            
            User->>UI: "저장" 버튼 클릭
            
            UI->>Controller: PUT /api/user/properties/{propertyId}<br/>{title, price, description, ...}
            
            Controller->>Service: updateProperty(propertyId, userId, updateRequest)
            
            Service->>PropertyRepo: findById(propertyId)
            PropertyRepo->>DB: SELECT properties WHERE id = ?
            DB-->>PropertyRepo: Property
            PropertyRepo-->>Service: Property
            
            Service->>Service: 권한 확인
            
            alt 권한 없음
                Service-->>Controller: 403 Forbidden
            else 권한 있음
                Service->>Service: 매물 정보 업데이트<br/>(title, price, description 등)
                Service->>Service: updatedAt 갱신
                
                Service->>PropertyRepo: save(Property)
                PropertyRepo->>DB: UPDATE properties
                DB-->>PropertyRepo: 완료
                PropertyRepo-->>Service: Property (업데이트됨)
                
                Service-->>Controller: 200 OK<br/>{매물 정보}
            end
            
            Controller-->>UI: 200 OK
            
            UI->>UI: 성공 메시지 표시
            UI->>UI: 목록 새로고침
            UI-->>User: "매물 정보가 수정되었습니다"
        end
    end

```

# 도림 21번

```mermaid
sequenceDiagram
    actor User as 사용자
    participant UI as Frontend
    participant Controller as PropertyController
    participant Service as PropertyService
    participant PropertyRepo as PropertyRepository
    participant UserRepo as UserRepository
    participant DB as Database

    User->>UI: "다른 소유자 매물" 메뉴 선택
    
    UI->>UI: 로그인 상태 확인
    
    alt 로그인하지 않은 상태
        UI->>UI: 로그인 페이지로 리다이렉트
        UI-->>User: "로그인이 필요합니다"
    else 로그인한 상태
        UI->>Controller: GET /api/properties/others<br/>?userId={userId}<br/>&page={page}<br/>&sort=-createdAt
        
        Controller->>Controller: JWT 토큰 검증
        Controller->>Controller: 현재 userId 추출
        
        Controller->>Service: getOtherOwnersProperties(userId, page, sort)
        
        Service->>UserRepo: findById(userId)
        UserRepo->>DB: SELECT users WHERE id = ?
        DB-->>UserRepo: User
        UserRepo-->>Service: User
        
        alt User 존재하지 않음
            Service-->>Controller: 오류 응답<br/>("사용자를 찾을 수 없습니다")
            Controller-->>UI: 404 Not Found
        else User 존재
            Service->>PropertyRepo: findAllByOwnerIdNot(userId, pageable)
            PropertyRepo->>DB: SELECT properties<br/>WHERE owner_id != ?<br/>AND status = 'AVAILABLE'<br/>ORDER BY created_at DESC<br/>LIMIT 20 OFFSET ?
            DB-->>PropertyRepo: 다른 소유자의 매물 목록
            PropertyRepo-->>Service: Page<Property>
            
            Service->>Service: 각 Property에 소유자 정보 추가<br/>(BrokerProfile 포함)
            
            Service->>Service: 거래 조건 정보 추가<br/>(PropertyOffer 통합)
            
            Service->>Service: Property 목록 변환<br/>(PropertyListDto)
            
            Service-->>Controller: 매물 목록 응답
        end
        
        Controller-->>UI: 200 OK<br/>[다른 소유자 매물 목록]
        
        UI->>UI: 매물 목록 렌더링<br/>(제목, 주소, 가격, 거래유형,<br/>소유자 이름, 평점)
        UI->>UI: 페이지네이션 컨트롤 표시
        UI->>UI: 필터/정렬 옵션 표시<br/>(거래유형, 가격범위, 지역)
        UI-->>User: 다른 소유자 매물 목록 표시
        
        alt 사용자가 필터 적용
            User->>UI: 필터 조건 선택<br/>(거래유형, 가격범위 등)
            
            UI->>Controller: GET /api/properties/others<br/>?userId={userId}<br/>&dealType={dealType}<br/>&minPrice={minPrice}<br/>&maxPrice={maxPrice}
            
            Controller->>Service: getOtherOwnersPropertiesFiltered(userId, filters, page)
            
            Service->>PropertyRepo: findAllByOwnerIdNotAndFilters(userId, filters, pageable)
            PropertyRepo->>DB: SELECT properties<br/>WHERE owner_id != ?<br/>AND status = 'AVAILABLE'<br/>AND price BETWEEN ? AND ?<br/>AND deal_type = ?<br/>ORDER BY created_at DESC
            DB-->>PropertyRepo: 필터링된 매물 목록
            PropertyRepo-->>Service: List<Property>
            
            Service-->>Controller: 필터링된 목록 응답
            Controller-->>UI: 200 OK<br/>[필터링된 목록]
            
            UI->>UI: 필터 적용된 목록 표시
            UI->>UI: 필터 상태 표시 (활성 필터 배지)
        end
        
        alt 사용자가 페이지 이동
            User->>UI: 다음/이전 페이지 클릭
            
            UI->>Controller: GET /api/properties/others<br/>?userId={userId}&page={nextPage}
            
            Controller->>Service: getOtherOwnersProperties(userId, nextPage, sort)
            Service->>PropertyRepo: findAllByOwnerIdNot(userId, pageable)
            PropertyRepo->>DB: SELECT properties (다음 페이지)
            DB-->>PropertyRepo: 다음 페이지 매물
            PropertyRepo-->>Service: Page<Property>
            
            Service-->>Controller: 다음 페이지 응답
            Controller-->>UI: 200 OK<br/>[다음 페이지 데이터]
            
            UI->>UI: 목록 업데이트
            UI->>UI: 페이지 번호 갱신
        end
        
        alt 사용자가 특정 매물 클릭
            User->>UI: 매물 카드 클릭
            
            UI->>UI: 매물 상세 페이지로 네비게이션
            UI->>UI: propertyId 파라미터 전달
        end
        
        alt 네트워크/DB 오류 발생
            Service->>Service: 오류 감지
            
            Service-->>Controller: 오류 응답
            Controller-->>UI: 400/500 에러
            
            UI->>UI: 로딩 표시기 숨김
            UI->>UI: "매물 목록을 불러올 수 없습니다."<br/>오류 메시지 표시
            UI->>UI: "재시도" 버튼 표시
            
            User->>UI: "재시도" 버튼 클릭
            UI->>UI: 이전 요청 재시도
        end
    end

```

# 도림 22번

```mermaid
sequenceDiagram
    actor User as 사용자
    participant UI as Frontend
    participant Controller as PropertyController
    participant Service as PropertyService
    participant PropertyRepo as PropertyRepository
    participant PropertyImageRepo as PropertyImageRepository
    participant PropertyOfferRepo as PropertyOfferRepository
    participant UserRepo as UserRepository
    participant DB as Database

    User->>UI: 매물 목록에서 특정 매물 클릭
    
    UI->>UI: 선택된 propertyId 추출
    
    UI->>Controller: GET /api/properties/{propertyId}
    
    Controller->>Controller: propertyId 검증
    
    Controller->>Service: getPropertyDetail(propertyId)
    
    Service->>PropertyRepo: findById(propertyId)
    PropertyRepo->>DB: SELECT properties WHERE id = ?
    DB-->>PropertyRepo: Property
    PropertyRepo-->>Service: Property
    
    alt Property 존재하지 않음
        Service-->>Controller: 오류 응답<br/>("존재하지 않는 매물입니다")
        Controller-->>UI: 404 Not Found
        UI->>UI: 오류 메시지 표시
        UI->>UI: "목록으로 돌아가기" 버튼 표시
        UI-->>User: 오류 안내
    else Property 존재
        Service->>PropertyImageRepo: findByPropertyId(propertyId)
        PropertyImageRepo->>DB: SELECT property_images<br/>WHERE property_id = ?<br/>ORDER BY display_order
        DB-->>PropertyImageRepo: 이미지 목록
        PropertyImageRepo-->>Service: List<PropertyImage>
        
        Service->>PropertyOfferRepo: findActiveOffers(propertyId)
        PropertyOfferRepo->>DB: SELECT property_offers<br/>WHERE property_id = ?<br/>AND is_active = true
        DB-->>PropertyOfferRepo: 활성 거래 조건 목록
        PropertyOfferRepo-->>Service: List<PropertyOffer>
        
        Service->>UserRepo: findById(property.ownerId)
        UserRepo->>DB: SELECT users WHERE id = ?
        DB-->>UserRepo: 소유자/중개인 정보
        UserRepo-->>Service: User
        
        Service->>Service: Property + 이미지 통합<br/>(PropertyDetailDto 생성)
        
        Service->>Service: PropertyOffer 데이터 추가<br/>(거래유형, 가격, 조건)
        
        Service->>Service: 소유자 정보 추가<br/>(이름, 평점, 리뷰 수)
        
        Service-->>Controller: PropertyDetailDto 반환
        
        Controller-->>UI: 200 OK<br/>{property 상세정보, images,<br/>offers, owner 정보}
        
        UI->>UI: 로딩 표시기 숨김
        
        UI->>UI: 상세 정보 렌더링<br/>(제목, 주소, 가격, 면적,<br/>건축년도, 설명 등)
        
        UI->>UI: 이미지 갤러리 표시<br/>(슬라이드쇼 또는 그리드)
        
        alt 이미지가 없음
            UI->>UI: "이미지 없음" 플레이스홀더 표시
        end
        
        UI->>UI: 거래 조건 표시<br/>(매매/전세/월세,<br/>보증금, 월세, 관리비)
        
        alt 여러 거래 조건 있음
            UI->>UI: 탭 또는 아코디언으로<br/>거래 조건 선택 가능하게 표시
        end
        
        UI->>UI: 소유자 정보 표시<br/>(프로필 사진, 이름,<br/>평점, 리뷰 수)
        
        UI->>UI: 액션 버튼 표시<br/>("채팅하기", "즐겨찾기",<br/>"공유하기" 등)
        
        UI-->>User: 매물 상세 정보 표시
        
        alt 사용자가 "채팅하기" 클릭
            User->>UI: "채팅하기" 버튼 클릭
            UI->>Controller: POST /api/chat/room<br/>?propertyId={propertyId}&otherId={ownerId}
            Controller->>Service: findOrCreateChatRoom(propertyId, userId, ownerId)
            Service-->>Controller: ChatRoom 반환
            Controller-->>UI: ChatRoom 정보
            UI->>UI: 채팅 화면으로 전환
        end
        
        alt 사용자가 "즐겨찾기" 클릭
            User->>UI: "즐겨찾기" 버튼 클릭
            
            UI->>Controller: POST /api/favorites<br/>{propertyId: {propertyId}}
            
            Controller->>Controller: JWT 검증
            Controller->>Controller: userId 추출
            
            Controller->>Service: addFavorite(userId, propertyId)
            
            Service->>Service: 중복 체크<br/>(이미 즐겨찾기됨)
            
            alt 이미 즐겨찾기됨
                Service-->>Controller: 오류 응답<br/>("이미 즐겨찾기되었습니다")
                Controller-->>UI: 400 Bad Request
            else 미등록 상태
                Service->>Service: Favorite 엔티티 생성<br/>(userId, propertyId)
                
                Service->>Service: 즐겨찾기 저장
                
                Service-->>Controller: 성공 응답
                
                Controller-->>UI: 201 Created
                
                UI->>UI: "즐겨찾기 해제" 버튼으로 변경
                UI->>UI: 성공 메시지 표시
            end
        end
        
        alt 사용자가 이미지 확대 클릭
            User->>UI: 이미지 클릭
            
            UI->>UI: 전체화면 갤러리 모달 표시
            UI->>UI: 이전/다음 이미지 네비게이션 제공
        end
    end

```

