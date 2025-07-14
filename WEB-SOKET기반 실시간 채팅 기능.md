## 💡실시간 채팅에서 사용하는 WebSocket 설정
---

```
package com.hdw.CardHive.chat;

import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;

@Configuration // 이 클래스는 Spring 설정 클래스라는 뜻
@EnableWebSocketMessageBroker // STOMP 기반 WebSocket 사용하겠다는 뜻 (실시간 채팅 기능 켜는 설정)
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

```

### 1️⃣ 클라이언트가 메시지를 받을 수 있는 경로 설정 (브로커 설정)
```
    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        // 클라이언트가 구독(subscribe)할 수 있는 주소 prefix 설정
        config.enableSimpleBroker("/topic", "/queue");

        /*
         📌 "/topic" → 여러 명에게 메시지 보낼 때 (예: 채팅방 전체에게)
         📌 "/queue" → 특정 유저에게만 보낼 때 (예: 1:1 채팅, 알림)
        */

        // 클라이언트가 메시지를 보낼 때 사용할 prefix 설정
        config.setApplicationDestinationPrefixes("/app");

        /*
         📌 클라이언트가 서버에 메시지 보낼 땐 "/app/..."으로 보냄
         예: stompClient.send("/app/chat/message", ...)
         → 서버에서는 @MessageMapping("/chat/message")로 받음
        */

        // 1:1 전용 메시지를 위한 prefix 설정
        config.setUserDestinationPrefix("/user");

        /*
         📌 convertAndSendToUser() 로 특정 유저에게 보낼 때,
         → 클라이언트는 "/user/queue/..."로 구독하고 있어야 메시지를 받음
        */
    }
```

### 2️⃣ 클라이언트가 WebSocket으로 연결할 수 있는 주소 등록

```
    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        // WebSocket 연결을 위한 엔드포인트 URL 지정
        registry.addEndpoint("/ws-stomp")
                .setAllowedOriginPatterns("*") // CORS 설정: 모든 도메인 허용 (개발 중엔 OK, 배포 시 주의)
                .withSockJS(); // WebSocket 미지원 브라우저를 위한 fallback 옵션
        /*
         📌 클라이언트는 "ws://localhost:8080/ws-stomp"로 연결 시도
         📌 SockJS를 통해 브라우저 호환성 문제 해결 (자동 fallback 지원)
        */
    }
}

```
---
## 📘간단한 예시 흐름

✅ **클라이언트**가 채팅 메시지 보내기

```
stompClient.send("/app/chat/message", {}, JSON.stringify({
  roomId: "1",
  sender: "user123",
  message: "안녕하세요!"
}));

```

✅ **서버**가 메시지 받아서 모든 사람에게 전달
```
@MessageMapping("/chat/message")
public void sendMessage(ChatMessage message) {
    messagingTemplate.convertAndSend("/topic/chatroom/" + message.getRoomId(), message);
}
```

✅ **클라이언트**가 구독하고 기다리는 구독 코드
```
stompClient.subscribe("/topic/chatroom/1", (msg) => {
  console.log("받은 메시지:", JSON.parse(msg.body));
});

```

---

## 📌전체 흐름 한 눈에 보기
| 동작           | 설명                    | 예시                                                       |
| ------------ | --------------------- | -------------------------------------------------------- |
| WebSocket 연결 | 클라이언트가 서버와 실시간 연결을 생성 | `/ws-stomp`                                              |
| 메시지 보내기      | 클라이언트 → 서버            | `/app/chat/message` → `@MessageMapping("/chat/message")` |
| 메시지 받기 (채팅방) | 서버 → 모든 유저            | `/topic/chatroom/1`                                      |
| 메시지 받기 (1:1) | 서버 → 특정 유저            | `/user/queue/chat` ← convertAndSendToUser() 사용           |


✅ 요약
- setApplicationDestinationPrefixes("/app"): 메시지 보낼 때 prefix

- enableSimpleBroker("/topic", "/queue"): 메시지 받을 때 prefix

- setUserDestinationPrefix("/user"): 1:1 메시지 전용 구독 경로

- addEndpoint("/ws-stomp"): 클라이언트가 WebSocket 연결하는 주소

- .withSockJS(): 구형 브라우저 호환





