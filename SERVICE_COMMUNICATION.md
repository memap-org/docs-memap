# Service Communication Patterns

## Overview

This document describes the migration strategy from RESTful APIs to more efficient communication patterns for the MeMap microservices platform. It covers:

1. **gRPC** - For high-performance service-to-service communication
2. **WebSockets** - For real-time client-server communication
3. **WebRTC** - For peer-to-peer real-time features (video/audio/data)

## Current State: RESTful Communication

```
┌─────────────────┐   HTTP/REST    ┌─────────────────┐
│ Profile Service │◄──────────────►│Identity Service │
│     (8082)      │   (JSON/HTTP)  │     (8081)      │
└─────────────────┘                └─────────────────┘
        │
        │ HTTP/REST
        ▼
┌─────────────────┐
│ Roadmap Service │
│     (8083)      │
└─────────────────┘
```

**Current Limitations:**

- High latency due to HTTP overhead and JSON serialization
- No bidirectional streaming
- No real-time push notifications
- Verbose data format (JSON)
- Connection overhead for each request

---

## Part 1: gRPC for Service-to-Service Communication

### Why gRPC Instead of REST?

| Aspect              | REST (Current) | gRPC (Proposed)           |
| ------------------- | -------------- | ------------------------- |
| **Protocol**        | HTTP/1.1       | HTTP/2                    |
| **Data Format**     | JSON (text)    | Protocol Buffers (binary) |
| **Performance**     | ~10x slower    | Optimized, low latency    |
| **Streaming**       | Not supported  | Bidirectional streaming   |
| **Type Safety**     | None           | Strong typing via .proto  |
| **Code Generation** | Manual         | Auto-generated stubs      |
| **Payload Size**    | Large          | ~30% smaller              |

### Target Architecture with gRPC

```
┌─────────────────┐     gRPC       ┌─────────────────┐
│ Profile Service │◄─────────────►│Identity Service │
│     (8082)      │  (HTTP/2+PB)  │     (8081)      │
└─────────────────┘                └─────────────────┘
        │
        │ gRPC (HTTP/2)
        ▼
┌─────────────────┐     gRPC       ┌─────────────────┐
│ Roadmap Service │◄─────────────►│   AI Service    │
│     (8083)      │               │     (8084)      │
└─────────────────┘                └─────────────────┘
```

### Implementation Guide

#### Step 1: Define Protocol Buffers (.proto files)

Create a shared `proto/` directory:

```
proto/
├── identity/
│   └── identity.proto
├── profile/
│   └── profile.proto
├── roadmap/
│   └── roadmap.proto
└── common/
    └── common.proto
```

**Example: `proto/identity/identity.proto`**

```protobuf
syntax = "proto3";

package identity;

option java_multiple_files = true;
option java_package = "com.memap.identity.grpc";
option java_outer_classname = "IdentityProto";

// Service definition
service IdentityService {
  // Synchronous token validation
  rpc ValidateToken(ValidateTokenRequest) returns (ValidateTokenResponse);

  // Get user authentication details
  rpc GetUserAuth(GetUserAuthRequest) returns (UserAuthResponse);

  // Bidirectional streaming for real-time auth events
  rpc AuthEventStream(stream AuthEvent) returns (stream AuthEventResponse);
}

// Messages
message ValidateTokenRequest {
  string token = 1;
}

message ValidateTokenResponse {
  bool valid = 1;
  string user_id = 2;
  repeated string roles = 3;
  int64 expires_at = 4;
}

message GetUserAuthRequest {
  string user_id = 1;
}

message UserAuthResponse {
  string user_id = 1;
  string username = 2;
  repeated string roles = 3;
  bool active = 4;
}

message AuthEvent {
  string event_type = 1;
  string user_id = 2;
  int64 timestamp = 3;
}

message AuthEventResponse {
  bool acknowledged = 1;
  string message = 2;
}
```

**Example: `proto/profile/profile.proto`**

```protobuf
syntax = "proto3";

package profile;

option java_multiple_files = true;
option java_package = "com.memap.profile.grpc";

service ProfileService {
  rpc GetProfile(GetProfileRequest) returns (ProfileResponse);
  rpc UpdateProfile(UpdateProfileRequest) returns (ProfileResponse);
  rpc SearchProfiles(SearchRequest) returns (stream ProfileResponse);
}

message GetProfileRequest {
  string user_id = 1;
}

message ProfileResponse {
  string user_id = 1;
  string display_name = 2;
  string email = 3;
  string avatar_url = 4;
  string bio = 5;
  int64 created_at = 6;
}

message UpdateProfileRequest {
  string user_id = 1;
  optional string display_name = 2;
  optional string bio = 3;
  optional string avatar_url = 4;
}

message SearchRequest {
  string query = 1;
  int32 page = 2;
  int32 size = 3;
}
```

#### Step 2: Spring Boot gRPC Integration

**Add Dependencies (pom.xml)**

```xml
<dependencies>
    <!-- gRPC Spring Boot Starter -->
    <dependency>
        <groupId>net.devh</groupId>
        <artifactId>grpc-spring-boot-starter</artifactId>
        <version>2.15.0.RELEASE</version>
    </dependency>

    <!-- Protocol Buffers -->
    <dependency>
        <groupId>com.google.protobuf</groupId>
        <artifactId>protobuf-java</artifactId>
        <version>3.25.1</version>
    </dependency>

    <!-- gRPC dependencies -->
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-netty-shaded</artifactId>
        <version>1.60.0</version>
    </dependency>
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-protobuf</artifactId>
        <version>1.60.0</version>
    </dependency>
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-stub</artifactId>
        <version>1.60.0</version>
    </dependency>
</dependencies>

<build>
    <extensions>
        <extension>
            <groupId>kr.motd.maven</groupId>
            <artifactId>os-maven-plugin</artifactId>
            <version>1.7.1</version>
        </extension>
    </extensions>
    <plugins>
        <plugin>
            <groupId>org.xolstice.maven.plugins</groupId>
            <artifactId>protobuf-maven-plugin</artifactId>
            <version>0.6.1</version>
            <configuration>
                <protocArtifact>com.google.protobuf:protoc:3.25.1:exe:${os.detected.classifier}</protocArtifact>
                <pluginId>grpc-java</pluginId>
                <pluginArtifact>io.grpc:protoc-gen-grpc-java:1.60.0:exe:${os.detected.classifier}</pluginArtifact>
            </configuration>
            <executions>
                <execution>
                    <goals>
                        <goal>compile</goal>
                        <goal>compile-custom</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

#### Step 3: Implement gRPC Server

**Identity Service gRPC Server:**

```java
package com.memap.identity.grpc;

import io.grpc.stub.StreamObserver;
import net.devh.boot.grpc.server.service.GrpcService;

@GrpcService
public class IdentityGrpcService extends IdentityServiceGrpc.IdentityServiceImplBase {

    private final TokenService tokenService;
    private final UserRepository userRepository;

    public IdentityGrpcService(TokenService tokenService, UserRepository userRepository) {
        this.tokenService = tokenService;
        this.userRepository = userRepository;
    }

    @Override
    public void validateToken(ValidateTokenRequest request,
                              StreamObserver<ValidateTokenResponse> responseObserver) {
        try {
            TokenValidationResult result = tokenService.validate(request.getToken());

            ValidateTokenResponse response = ValidateTokenResponse.newBuilder()
                .setValid(result.isValid())
                .setUserId(result.getUserId())
                .addAllRoles(result.getRoles())
                .setExpiresAt(result.getExpiresAt())
                .build();

            responseObserver.onNext(response);
            responseObserver.onCompleted();
        } catch (Exception e) {
            responseObserver.onError(
                Status.INTERNAL.withDescription(e.getMessage()).asRuntimeException()
            );
        }
    }

    @Override
    public void getUserAuth(GetUserAuthRequest request,
                           StreamObserver<UserAuthResponse> responseObserver) {
        userRepository.findById(request.getUserId())
            .ifPresentOrElse(
                user -> {
                    UserAuthResponse response = UserAuthResponse.newBuilder()
                        .setUserId(user.getId())
                        .setUsername(user.getUsername())
                        .addAllRoles(user.getRoles())
                        .setActive(user.isActive())
                        .build();
                    responseObserver.onNext(response);
                    responseObserver.onCompleted();
                },
                () -> responseObserver.onError(
                    Status.NOT_FOUND.withDescription("User not found").asRuntimeException()
                )
            );
    }
}
```

#### Step 4: Implement gRPC Client

**Profile Service calling Identity Service:**

```java
package com.memap.profile.client;

import com.memap.identity.grpc.*;
import io.grpc.StatusRuntimeException;
import net.devh.boot.grpc.client.inject.GrpcClient;
import org.springframework.stereotype.Service;

@Service
public class IdentityGrpcClient {

    @GrpcClient("identity-service")
    private IdentityServiceGrpc.IdentityServiceBlockingStub identityStub;

    public TokenValidationResult validateToken(String token) {
        try {
            ValidateTokenRequest request = ValidateTokenRequest.newBuilder()
                .setToken(token)
                .build();

            ValidateTokenResponse response = identityStub.validateToken(request);

            return new TokenValidationResult(
                response.getValid(),
                response.getUserId(),
                response.getRolesList(),
                response.getExpiresAt()
            );
        } catch (StatusRuntimeException e) {
            throw new AuthenticationException("Token validation failed: " + e.getStatus());
        }
    }
}
```

#### Step 5: Configuration

**application.yml:**

```yaml
# gRPC Server Configuration
grpc:
  server:
    port: 9091
    security:
      enabled: false  # Enable TLS in production

# gRPC Client Configuration
grpc:
  client:
    identity-service:
      address: 'static://localhost:9091'
      negotiationType: plaintext  # Use TLS in production
    profile-service:
      address: 'static://localhost:9092'
      negotiationType: plaintext
    roadmap-service:
      address: 'static://localhost:9093'
      negotiationType: plaintext
```

### Service Port Mapping (gRPC)

| Service            | REST Port | gRPC Port |
| ------------------ | --------- | --------- |
| Identity Service   | 8081      | 9091      |
| Profile Service    | 8082      | 9092      |
| Roadmap Service    | 8083      | 9093      |
| Roadmap AI Service | 8084      | 9094      |
| File Service       | 8085      | 9095      |
| Learning Service   | 8086      | 9096      |
| Payment Service    | 8087      | 9097      |

---

## Part 2: WebSockets for Real-Time Client Communication

### Use Cases

- Real-time notifications
- Live progress updates
- Collaborative editing
- Chat messages
- AI streaming responses

### Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                          Clients                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │   Browser   │  │   Mobile    │  │   Desktop   │              │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘              │
└─────────┼────────────────┼────────────────┼─────────────────────┘
          │                │                │
          └────────────────┼────────────────┘
                           │ WebSocket (ws://)
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│                    WebSocket Gateway                              │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │              Spring WebSocket / Socket.IO                   │  │
│  │                                                             │  │
│  │  Channels:                                                  │  │
│  │  • /user/{userId}/notifications                            │  │
│  │  • /roadmap/{roadmapId}/updates                            │  │
│  │  • /ai/stream/{sessionId}                                  │  │
│  │  • /learning/quiz/{quizId}/live                            │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
                           │
                           │ Events (RabbitMQ)
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│                      Backend Services                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │ Identity │  │ Profile  │  │ Roadmap  │  │ AI Svc   │        │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘        │
└──────────────────────────────────────────────────────────────────┘
```

### Spring WebSocket Implementation

**1. WebSocket Configuration:**

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        // Enable a simple in-memory broker for subscriptions
        config.enableSimpleBroker("/topic", "/queue", "/user");

        // Prefix for client-to-server messages
        config.setApplicationDestinationPrefixes("/app");

        // User destination prefix for private messages
        config.setUserDestinationPrefix("/user");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
            .setAllowedOrigins("http://localhost:5173")
            .withSockJS();
    }
}
```

**2. WebSocket Security:**

```java
@Configuration
public class WebSocketSecurityConfig {

    @Bean
    public ChannelInterceptor authChannelInterceptor(JwtTokenProvider tokenProvider) {
        return new ChannelInterceptor() {
            @Override
            public Message<?> preSend(Message<?> message, MessageChannel channel) {
                StompHeaderAccessor accessor =
                    StompHeaderAccessor.wrap(message);

                if (StompCommand.CONNECT.equals(accessor.getCommand())) {
                    String token = accessor.getFirstNativeHeader("Authorization");
                    if (token != null && token.startsWith("Bearer ")) {
                        String jwt = token.substring(7);
                        Authentication auth = tokenProvider.getAuthentication(jwt);
                        accessor.setUser(auth);
                    }
                }
                return message;
            }
        };
    }
}
```

**3. WebSocket Controller:**

```java
@Controller
public class NotificationWebSocketController {

    private final SimpMessagingTemplate messagingTemplate;

    @MessageMapping("/notifications/subscribe")
    public void subscribeToNotifications(Principal principal) {
        // User is now subscribed to their personal channel
        log.info("User {} subscribed to notifications", principal.getName());
    }

    // Called from other services via RabbitMQ
    @RabbitListener(queues = "notification.queue")
    public void handleNotificationEvent(NotificationEvent event) {
        messagingTemplate.convertAndSendToUser(
            event.getUserId(),
            "/queue/notifications",
            new NotificationMessage(event)
        );
    }

    // Broadcast to all users watching a roadmap
    public void broadcastRoadmapUpdate(String roadmapId, RoadmapUpdate update) {
        messagingTemplate.convertAndSend(
            "/topic/roadmap/" + roadmapId,
            update
        );
    }
}
```

**4. Frontend WebSocket Client (React):**

```typescript
// src/hooks/useWebSocket.ts
import { Client } from '@stomp/stompjs';
import SockJS from 'sockjs-client';
import { useEffect, useRef, useCallback } from 'react';

export function useWebSocket(userId: string, token: string) {
  const clientRef = useRef<Client | null>(null);

  useEffect(() => {
    const client = new Client({
      webSocketFactory: () => new SockJS('http://localhost:8082/ws'),
      connectHeaders: {
        Authorization: `Bearer ${token}`,
      },
      onConnect: () => {
        console.log('WebSocket connected');

        // Subscribe to personal notifications
        client.subscribe(`/user/${userId}/queue/notifications`, (message) => {
          const notification = JSON.parse(message.body);
          handleNotification(notification);
        });
      },
      onDisconnect: () => {
        console.log('WebSocket disconnected');
      },
    });

    client.activate();
    clientRef.current = client;

    return () => {
      client.deactivate();
    };
  }, [userId, token]);

  const subscribeToRoadmap = useCallback(
    (roadmapId: string, callback: (update: any) => void) => {
      if (clientRef.current?.connected) {
        return clientRef.current.subscribe(
          `/topic/roadmap/${roadmapId}`,
          (message) => {
            callback(JSON.parse(message.body));
          },
        );
      }
    },
    [],
  );

  return { subscribeToRoadmap };
}
```

---

## Part 3: WebRTC for Peer-to-Peer Communication

### Use Cases

- **Video/Audio Calls** - Tutoring sessions, live mentoring
- **Screen Sharing** - Collaborative roadmap editing
- **Peer-to-Peer Data** - Real-time collaborative features
- **Live Streaming** - Educational content streaming

### Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                    WebRTC Flow                                   │
│                                                                                  │
│  ┌──────────────┐                                      ┌──────────────┐         │
│  │   Client A   │                                      │   Client B   │         │
│  │  (Student)   │                                      │  (Teacher)   │         │
│  └──────┬───────┘                                      └───────┬──────┘         │
│         │                                                      │                │
│         │  1. Create Offer (SDP)                               │                │
│         │─────────────────────────────────────────────────────►│                │
│         │                                                      │                │
│         │  2. Create Answer (SDP)                              │                │
│         │◄─────────────────────────────────────────────────────│                │
│         │                                                      │                │
│         │  3. Exchange ICE Candidates                          │                │
│         │◄────────────────────────────────────────────────────►│                │
│         │                                                      │                │
│         │           ═══════════════════════════                │                │
│         │           ║   Direct P2P Connection  ║               │                │
│         │           ║   (Video/Audio/Data)     ║               │                │
│         │◄══════════╩══════════════════════════╩══════════════►│                │
│         │                                                      │                │
└─────────┼──────────────────────────────────────────────────────┼────────────────┘
          │                      │                               │
          │         ┌────────────┴────────────┐                  │
          │         │    Signaling Server     │                  │
          └────────►│    (WebSocket Based)    │◄─────────────────┘
                    │                         │
                    │  • Room management      │
                    │  • SDP exchange         │
                    │  • ICE candidate relay  │
                    └─────────────────────────┘
```

### Signaling Server Implementation

**1. WebRTC Signaling with Spring WebSocket:**

```java
@Controller
public class WebRTCSignalingController {

    private final Map<String, Set<String>> rooms = new ConcurrentHashMap<>();
    private final SimpMessagingTemplate messagingTemplate;

    @MessageMapping("/webrtc/join/{roomId}")
    public void joinRoom(@DestinationVariable String roomId,
                         Principal principal,
                         JoinRoomMessage message) {
        String oderId = principal.getName();

        rooms.computeIfAbsent(roomId, k -> ConcurrentHashMap.newKeySet())
             .add(userId);

        // Notify other participants
        messagingTemplate.convertAndSend(
            "/topic/webrtc/room/" + roomId,
            new UserJoinedEvent(userId, message.getDisplayName())
        );
    }

    @MessageMapping("/webrtc/offer/{roomId}")
    public void handleOffer(@DestinationVariable String roomId,
                           Principal principal,
                           WebRTCOffer offer) {
        // Forward offer to the target peer
        messagingTemplate.convertAndSendToUser(
            offer.getTargetUserId(),
            "/queue/webrtc/offer",
            new WebRTCOfferMessage(principal.getName(), offer.getSdp())
        );
    }

    @MessageMapping("/webrtc/answer/{roomId}")
    public void handleAnswer(@DestinationVariable String roomId,
                            Principal principal,
                            WebRTCAnswer answer) {
        // Forward answer to the peer
        messagingTemplate.convertAndSendToUser(
            answer.getTargetUserId(),
            "/queue/webrtc/answer",
            new WebRTCAnswerMessage(principal.getName(), answer.getSdp())
        );
    }

    @MessageMapping("/webrtc/ice-candidate/{roomId}")
    public void handleIceCandidate(@DestinationVariable String roomId,
                                   Principal principal,
                                   IceCandidate candidate) {
        // Forward ICE candidate to the target peer
        messagingTemplate.convertAndSendToUser(
            candidate.getTargetUserId(),
            "/queue/webrtc/ice-candidate",
            new IceCandidateMessage(principal.getName(), candidate.getCandidate())
        );
    }

    @MessageMapping("/webrtc/leave/{roomId}")
    public void leaveRoom(@DestinationVariable String roomId,
                         Principal principal) {
        String userId = principal.getName();
        Set<String> roomUsers = rooms.get(roomId);

        if (roomUsers != null) {
            roomUsers.remove(userId);

            messagingTemplate.convertAndSend(
                "/topic/webrtc/room/" + roomId,
                new UserLeftEvent(userId)
            );

            if (roomUsers.isEmpty()) {
                rooms.remove(roomId);
            }
        }
    }
}
```

**2. WebRTC DTOs:**

```java
public record WebRTCOffer(
    String targetUserId,
    String sdp,
    String type
) {}

public record WebRTCAnswer(
    String targetUserId,
    String sdp,
    String type
) {}

public record IceCandidate(
    String targetUserId,
    String candidate,
    String sdpMid,
    Integer sdpMLineIndex
) {}

public record UserJoinedEvent(
    String oderId,
    String displayName
) {}

public record UserLeftEvent(
    String userId
) {}
```

**3. Frontend WebRTC Client (React):**

```typescript
// src/hooks/useWebRTC.ts
import { useRef, useCallback, useState } from 'react';
import { Client } from '@stomp/stompjs';

interface PeerConnection {
  connection: RTCPeerConnection;
  dataChannel?: RTCDataChannel;
}

export function useWebRTC(userId: string, roomId: string, stompClient: Client) {
  const [peers, setPeers] = useState<Map<string, PeerConnection>>(new Map());
  const localStreamRef = useRef<MediaStream | null>(null);
  const [remoteStreams, setRemoteStreams] = useState<Map<string, MediaStream>>(
    new Map(),
  );

  const configuration: RTCConfiguration = {
    iceServers: [
      { urls: 'stun:stun.l.google.com:19302' },
      { urls: 'stun:stun1.l.google.com:19302' },
      // Add TURN server for production
      // {
      //   urls: 'turn:your-turn-server.com:3478',
      //   username: 'username',
      //   credential: 'password'
      // }
    ],
  };

  const createPeerConnection = useCallback(
    (peerId: string): RTCPeerConnection => {
      const pc = new RTCPeerConnection(configuration);

      // Add local stream tracks
      if (localStreamRef.current) {
        localStreamRef.current.getTracks().forEach((track) => {
          pc.addTrack(track, localStreamRef.current!);
        });
      }

      // Handle incoming tracks
      pc.ontrack = (event) => {
        setRemoteStreams((prev) => {
          const newMap = new Map(prev);
          newMap.set(peerId, event.streams[0]);
          return newMap;
        });
      };

      // Handle ICE candidates
      pc.onicecandidate = (event) => {
        if (event.candidate) {
          stompClient.publish({
            destination: `/app/webrtc/ice-candidate/${roomId}`,
            body: JSON.stringify({
              targetUserId: peerId,
              candidate: event.candidate.candidate,
              sdpMid: event.candidate.sdpMid,
              sdpMLineIndex: event.candidate.sdpMLineIndex,
            }),
          });
        }
      };

      pc.onconnectionstatechange = () => {
        console.log(`Connection state with ${peerId}:`, pc.connectionState);
      };

      return pc;
    },
    [roomId, stompClient],
  );

  const startCall = useCallback(
    async (targetUserId: string) => {
      const pc = createPeerConnection(targetUserId);

      // Create data channel for messaging
      const dataChannel = pc.createDataChannel('chat');
      dataChannel.onmessage = (event) => {
        console.log('Received message:', event.data);
      };

      setPeers((prev) => {
        const newMap = new Map(prev);
        newMap.set(targetUserId, { connection: pc, dataChannel });
        return newMap;
      });

      // Create and send offer
      const offer = await pc.createOffer();
      await pc.setLocalDescription(offer);

      stompClient.publish({
        destination: `/app/webrtc/offer/${roomId}`,
        body: JSON.stringify({
          targetUserId,
          sdp: offer.sdp,
          type: offer.type,
        }),
      });
    },
    [createPeerConnection, roomId, stompClient],
  );

  const handleOffer = useCallback(
    async (fromUserId: string, sdp: string) => {
      const pc = createPeerConnection(fromUserId);

      setPeers((prev) => {
        const newMap = new Map(prev);
        newMap.set(fromUserId, { connection: pc });
        return newMap;
      });

      await pc.setRemoteDescription(
        new RTCSessionDescription({ type: 'offer', sdp }),
      );

      const answer = await pc.createAnswer();
      await pc.setLocalDescription(answer);

      stompClient.publish({
        destination: `/app/webrtc/answer/${roomId}`,
        body: JSON.stringify({
          targetUserId: fromUserId,
          sdp: answer.sdp,
          type: answer.type,
        }),
      });
    },
    [createPeerConnection, roomId, stompClient],
  );

  const handleAnswer = useCallback(
    async (fromUserId: string, sdp: string) => {
      const peer = peers.get(fromUserId);
      if (peer) {
        await peer.connection.setRemoteDescription(
          new RTCSessionDescription({ type: 'answer', sdp }),
        );
      }
    },
    [peers],
  );

  const handleIceCandidate = useCallback(
    async (fromUserId: string, candidate: RTCIceCandidateInit) => {
      const peer = peers.get(fromUserId);
      if (peer) {
        await peer.connection.addIceCandidate(new RTCIceCandidate(candidate));
      }
    },
    [peers],
  );

  const initializeMedia = useCallback(
    async (video: boolean = true, audio: boolean = true) => {
      const stream = await navigator.mediaDevices.getUserMedia({
        video,
        audio,
      });
      localStreamRef.current = stream;
      return stream;
    },
    [],
  );

  const endCall = useCallback(() => {
    peers.forEach((peer) => {
      peer.connection.close();
      peer.dataChannel?.close();
    });
    setPeers(new Map());
    setRemoteStreams(new Map());

    if (localStreamRef.current) {
      localStreamRef.current.getTracks().forEach((track) => track.stop());
      localStreamRef.current = null;
    }
  }, [peers]);

  return {
    peers,
    remoteStreams,
    localStream: localStreamRef.current,
    startCall,
    handleOffer,
    handleAnswer,
    handleIceCandidate,
    initializeMedia,
    endCall,
  };
}
```

**4. Video Call Component:**

```tsx
// src/components/VideoCall.tsx
import React, { useEffect, useRef, useState } from 'react';
import { useWebRTC } from '../hooks/useWebRTC';
import { useWebSocket } from '../hooks/useWebSocket';

interface VideoCallProps {
  roomId: string;
  userId: string;
}

export const VideoCall: React.FC<VideoCallProps> = ({ roomId, userId }) => {
  const localVideoRef = useRef<HTMLVideoElement>(null);
  const { stompClient } = useWebSocket(userId);
  const {
    remoteStreams,
    localStream,
    startCall,
    handleOffer,
    handleAnswer,
    handleIceCandidate,
    initializeMedia,
    endCall,
  } = useWebRTC(userId, roomId, stompClient);

  const [isCallActive, setIsCallActive] = useState(false);

  useEffect(() => {
    if (localVideoRef.current && localStream) {
      localVideoRef.current.srcObject = localStream;
    }
  }, [localStream]);

  // Subscribe to WebRTC signaling messages
  useEffect(() => {
    if (stompClient?.connected) {
      const offerSub = stompClient.subscribe(
        `/user/queue/webrtc/offer`,
        (message) => {
          const { fromUserId, sdp } = JSON.parse(message.body);
          handleOffer(fromUserId, sdp);
        },
      );

      const answerSub = stompClient.subscribe(
        `/user/queue/webrtc/answer`,
        (message) => {
          const { fromUserId, sdp } = JSON.parse(message.body);
          handleAnswer(fromUserId, sdp);
        },
      );

      const iceSub = stompClient.subscribe(
        `/user/queue/webrtc/ice-candidate`,
        (message) => {
          const { fromUserId, candidate, sdpMid, sdpMLineIndex } = JSON.parse(
            message.body,
          );
          handleIceCandidate(fromUserId, { candidate, sdpMid, sdpMLineIndex });
        },
      );

      return () => {
        offerSub.unsubscribe();
        answerSub.unsubscribe();
        iceSub.unsubscribe();
      };
    }
  }, [stompClient, handleOffer, handleAnswer, handleIceCandidate]);

  const handleStartCall = async () => {
    await initializeMedia();
    setIsCallActive(true);

    // Join the room
    stompClient.publish({
      destination: `/app/webrtc/join/${roomId}`,
      body: JSON.stringify({ displayName: 'User' }),
    });
  };

  const handleEndCall = () => {
    endCall();
    setIsCallActive(false);

    stompClient.publish({
      destination: `/app/webrtc/leave/${roomId}`,
    });
  };

  return (
    <div className="video-call-container">
      <div className="local-video">
        <video ref={localVideoRef} autoPlay muted playsInline />
      </div>

      <div className="remote-videos">
        {Array.from(remoteStreams.entries()).map(([oderId, stream]) => (
          <RemoteVideo key={peerId} stream={stream} peerId={peerId} />
        ))}
      </div>

      <div className="controls">
        {!isCallActive ? (
          <button onClick={handleStartCall}>Start Call</button>
        ) : (
          <button onClick={handleEndCall}>End Call</button>
        )}
      </div>
    </div>
  );
};

const RemoteVideo: React.FC<{ stream: MediaStream; peerId: string }> = ({
  stream,
  peerId,
}) => {
  const videoRef = useRef<HTMLVideoElement>(null);

  useEffect(() => {
    if (videoRef.current) {
      videoRef.current.srcObject = stream;
    }
  }, [stream]);

  return (
    <div className="remote-video">
      <video ref={videoRef} autoPlay playsInline />
      <span className="peer-id">{peerId}</span>
    </div>
  );
};
```

---

## Part 4: Migration Strategy

### Phase 1: Foundation (Weeks 1-2)

1. **Setup Proto Repository**
   - Create shared proto definitions
   - Configure build plugins for all services

2. **Add gRPC Dependencies**
   - Update all service `pom.xml` files
   - Configure gRPC servers alongside REST

3. **Implement Core gRPC Services**
   - Identity Service: Token validation
   - Profile Service: Profile lookup

### Phase 2: Service Migration (Weeks 3-4)

1. **Migrate Internal Calls**
   - Replace OpenFeign clients with gRPC clients
   - Keep REST endpoints for external clients

2. **Add WebSocket Support**
   - Configure WebSocket gateway
   - Implement notification channels

3. **Testing**
   - Integration tests for gRPC
   - Load testing comparison

### Phase 3: Real-Time Features (Weeks 5-6)

1. **Implement WebRTC Signaling**
   - Add signaling server
   - Create room management

2. **Frontend Integration**
   - Add WebSocket hooks
   - Implement WebRTC hooks

3. **Feature Rollout**
   - Live notifications
   - AI streaming responses
   - Video call capabilities (optional)

### Phase 4: Optimization (Week 7+)

1. **Performance Tuning**
   - Connection pooling
   - Load balancing

2. **Security Hardening**
   - TLS for gRPC
   - TURN server for WebRTC

3. **Monitoring**
   - gRPC metrics
   - WebSocket connection tracking

---

## Comparison Summary

| Feature               | REST         | gRPC               | WebSocket      | WebRTC    |
| --------------------- | ------------ | ------------------ | -------------- | --------- |
| **Use Case**          | External API | Service-to-service | Real-time push | P2P media |
| **Protocol**          | HTTP/1.1     | HTTP/2             | WS             | UDP/DTLS  |
| **Latency**           | High         | Low                | Medium         | Very Low  |
| **Streaming**         | ❌           | ✅                 | ✅             | ✅        |
| **Browser Support**   | ✅           | ❌ (via proxy)     | ✅             | ✅        |
| **Type Safety**       | ❌           | ✅                 | ❌             | ❌        |
| **Firewall Friendly** | ✅           | ⚠️                 | ✅             | ⚠️        |

---

## Recommended Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                  Client Layer                                        │
│  ┌──────────────────────────────────────────────────────────────────────────────┐   │
│  │                              Frontend (React)                                 │   │
│  │  • REST for CRUD operations                                                  │   │
│  │  • WebSocket for real-time updates                                           │   │
│  │  • WebRTC for video/audio (optional future feature)                         │   │
│  └──────────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────────┘
                    │                         │                        │
                    │ REST (HTTPS)            │ WebSocket              │ WebRTC
                    ▼                         ▼                        ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                  Gateway Layer                                       │
│  ┌────────────────────┐  ┌────────────────────┐  ┌────────────────────┐            │
│  │   API Gateway      │  │  WebSocket Gateway │  │  Signaling Server  │            │
│  │   (REST → gRPC)    │  │                    │  │                    │            │
│  └─────────┬──────────┘  └─────────┬──────────┘  └────────────────────┘            │
└────────────┼───────────────────────┼────────────────────────────────────────────────┘
             │ gRPC                  │ Events
             ▼                       ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                  Service Layer                                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐             │
│  │ Identity │◄─┤ Profile  │◄─┤ Roadmap  │◄─┤   AI     │◄─┤ Payment  │             │
│  │ Service  │  │ Service  │  │ Service  │  │ Service  │  │ Service  │             │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘             │
│       ▲              ▲              ▲              ▲              ▲                 │
│       └──────────────┴──────────────┴──────────────┴──────────────┘                 │
│                              gRPC (Internal)                                         │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## References

- [gRPC Official Documentation](https://grpc.io/docs/)
- [Spring gRPC](https://github.com/grpc-ecosystem/grpc-spring)
- [Protocol Buffers Guide](https://protobuf.dev/programming-guides/proto3/)
- [WebRTC API](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API)
- [Spring WebSocket](https://docs.spring.io/spring-framework/reference/web/websocket.html)
- [STOMP Protocol](https://stomp.github.io/stomp-specification-1.2.html)
