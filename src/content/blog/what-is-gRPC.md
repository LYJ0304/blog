---
title: 'gRPC란?'
description: '주문 조회 API를 예로 들어 gRPC의 호출 방식과 스트리밍을 살펴보고, REST API와의 차이를 비교합니다.'
pubDate: 'Nov 24 2026'
tags: ['Network', 'Backend', 'gRPC']
---

주문 서비스에서 주문 한 건을 조회한다고 해봅시다. HTTP REST API라면 `GET /orders/123`으로 주문 리소스를 가리킵니다. gRPC에서는 `GetOrder` 메서드에 주문 ID를 담아 호출합니다. 같은 주문을 조회해도 API 계약을 표현하는 방식이 다릅니다.

## gRPC란?

gRPC는 서비스 간 원격 호출을 구현하는 오픈소스 프레임워크입니다. 일반적으로 Protocol Buffers(Protobuf)로 메서드와 메시지의 타입을 정의하고, HTTP/2로 메시지를 주고받습니다.

### RPC란?

RPC(Remote Procedure Call, 원격 프로시저 호출)는 다른 프로세스에서 실행되는 함수를 내 프로그램의 함수처럼 호출하는 방식입니다. 클라이언트는 서버 주소와 전송 형식을 매번 직접 다루는 대신, 원격 메서드를 호출하는 클라이언트 코드인 **스텁(stub)**을 사용합니다.

함수처럼 호출해도 실행은 네트워크를 거칩니다. 서버에 연결할 수 없거나 응답이 늦어질 수 있으므로 호출에 데드라인을 설정하고 실패를 처리해야 합니다. 특히 주문 생성처럼 상태를 바꾸는 호출은 클라이언트에서 시간이 초과됐어도 서버에서 처리가 끝났을 수 있습니다. 재시도가 필요하다면 중복 실행을 어떻게 방지할지도 함께 설계해야 합니다. [gRPC 공식 문서](https://grpc.io/docs/what-is-grpc/core-concepts/)

주문 조회 계약을 `.proto` 파일로 표현하면 다음과 같습니다.

```proto
syntax = "proto3";

package orders.v1;

service OrderService {
  rpc GetOrder(GetOrderRequest) returns (Order);
}

message GetOrderRequest {
  string id = 1;
}

message Order {
  string id = 1;
  string status = 2;
}
```

이 파일로 언어별 메시지 타입과 클라이언트 스텁, 서버 구현에 필요한 인터페이스를 생성할 수 있습니다. 클라이언트가 스텁의 `GetOrder`를 호출하면 라이브러리가 요청을 직렬화해 서버로 보냅니다. 서버는 요청을 역직렬화한 뒤 개발자가 구현한 핸들러를 실행하고, 클라이언트는 응답을 다시 타입이 있는 메시지로 받습니다. 같은 계약에서 코드를 생성하면 Go 클라이언트와 Java 서버도 메시지 구조를 공유할 수 있습니다. [gRPC 공식 문서](https://grpc.io/docs/what-is-grpc/core-concepts/)

`GetOrder`처럼 요청 하나에 응답 하나가 돌아오는 호출을 **단항(unary) RPC**라고 합니다. `.proto`에서 요청이나 응답 타입 앞에 `stream`을 붙이면 한 번의 호출에서 해당 방향으로 메시지를 여러 개 보낼 수 있습니다.

| 호출 방식 | 요청 → 응답 | 예시 |
| --- | --- | --- |
| 단항 | 하나 → 하나 | 주문 한 건 조회 |
| 서버 스트리밍 | 하나 → 여러 개 | 주문 상태 변경을 발생할 때마다 받기 |
| 클라이언트 스트리밍 | 여러 개 → 하나 | 여러 측정값을 보내고 집계 결과 받기 |
| 양방향 스트리밍 | 여러 개 → 여러 개 | 채팅 서버와 메시지 주고받기 |

양방향 스트리밍의 두 방향은 독립적으로 진행됩니다. 예를 들어 채팅 클라이언트는 자신이 메시지를 보내지 않는 동안에도 서버가 전달한 다른 사용자의 메시지를 받을 수 있습니다. [gRPC 공식 문서](https://grpc.io/docs/what-is-grpc/core-concepts/)

## REST란?

REST(Representational State Transfer)는 분산 하이퍼미디어 시스템을 설계하는 아키텍처 스타일입니다. 무상태성, 캐시 가능성, 일관된 인터페이스 등의 제약을 둡니다. 무상태성이란 서버가 주문 데이터를 저장할 수 없다는 뜻이 아니라, 각 요청을 처리하는 데 필요한 맥락이 요청 자체에 포함되어야 한다는 뜻입니다. [Roy Fielding의 REST 원문](https://ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm)

HTTP에서 REST 방식으로 API를 설계한다면 리소스를 URI로 식별하고, HTTP 메서드로 요청의 의미를 표현합니다. `GET /orders/123`은 주문 리소스의 표현을 조회하는 요청입니다. 서버는 그 표현을 아래처럼 JSON으로 반환할 수 있습니다.

```json
{
  "id": "123",
  "status": "PAID"
}
```

JSON은 자주 쓰이는 표현 형식입니다. 클라이언트와 서버가 합의한 미디어 타입에 따라 다른 형식도 사용할 수 있습니다. [Roy Fielding의 REST 원문](https://ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm)

## gRPC vs REST

### gRPC와 REST의 유사점

둘 다 클라이언트와 서버 사이에 API 계약이 필요합니다. 주문을 조회하려면 어떤 입력을 보내야 하는지, 성공한 응답에는 어떤 값이 들어 있는지, 주문이 없을 때 어떻게 응답하는지 합의해야 합니다. 구현 언어가 달라도 이 계약을 지키면 서로 통신할 수 있습니다.

두 방식에서 모두 스키마를 정의하고 코드를 생성할 수 있습니다. gRPC에서는 `.proto` 기반 코드 생성이 일반적인 개발 흐름에 포함되고, HTTP REST API에서는 필요에 따라 OpenAPI를 도입할 수 있습니다.

### gRPC와 REST의 주요 차이점

| 비교 항목 | gRPC | 일반적인 HTTP REST API |
| --- | --- | --- |
| 인터페이스 | 서비스의 메서드를 호출합니다. 예를 들어 `OrderService.GetOrder`를 사용합니다. | URI로 리소스를 식별하고 HTTP 메서드를 사용합니다. 예를 들어 `GET /orders/123`을 사용합니다. |
| 메시지 형식 | 기본적으로 Protobuf 바이너리 메시지를 주고받습니다. | JSON이 흔하며, 다른 미디어 타입도 사용할 수 있습니다. |
| 전송과 상태 | HTTP/2에서 표준 gRPC 호출은 `POST`로 전송하며, 호출 결과는 `grpc-status`로 전달합니다. | HTTP 메서드와 상태 코드로 요청과 응답의 의미를 전달합니다. |
| 스트리밍 | 계약에서 클라이언트, 서버, 양방향 스트리밍을 선언할 수 있습니다. | 실시간 업데이트에는 서버 전송 이벤트(SSE) 같은 별도 방식을 조합할 수 있습니다. |
| 브라우저 접근 | 브라우저에서는 gRPC-Web과 이를 지원하는 서버나 프록시를 사용할 수 있으며, 스트리밍 지원 범위를 확인해야 합니다. | 브라우저의 `fetch`로 호출할 수 있습니다. 출처가 다르면 CORS 설정이 필요할 수 있습니다. |

표준 gRPC에서 주문을 찾지 못하면 일반적으로 `NOT_FOUND`를 반환합니다. 이때 HTTP 상태가 `200`이어도 RPC 자체는 실패일 수 있으므로 모니터링할 때 gRPC 상태를 함께 봐야 합니다. HTTP REST API에서는 같은 상황을 보통 HTTP `404 Not Found`로 표현합니다. [gRPC over HTTP/2 명세](https://grpc.github.io/grpc/core/md_doc__p_r_o_t_o_c_o_l-_h_t_t_p2.html) · [gRPC-Web 공식 저장소](https://github.com/grpc/grpc-web)

성능은 실제 호출 패턴에 따라 달라집니다. Protobuf는 메시지 크기를 줄일 수 있고, HTTP/2는 하나의 연결에서 여러 호출을 동시에 처리할 수 있습니다. HTTP REST API도 HTTP/2를 사용할 수 있고, 캐시 가능한 조회는 캐시에서 응답해 서버 호출 자체를 줄일 수 있습니다. 성능이 선택 기준이라면 실제 페이로드와 동시 요청 수를 기준으로 응답 지연과 처리량을 비교하는 편이 좋습니다.

## 사용 사례

- **내부 마이크로서비스 통신.** 여러 언어로 작성한 주문 서비스와 재고 서비스가 같은 타입을 공유하고, 서비스 간 호출이 잦다면 gRPC를 고려할 수 있습니다. 스키마 변경 시 각 서비스가 언제 코드를 갱신하고 배포할지도 함께 정해야 합니다.
- **진행 상황을 계속 전달하는 API.** 서버가 작업의 진행률이나 주문 상태 변경을 연속으로 보내야 한다면 서버 스트리밍으로 표현할 수 있습니다. 장시간 연결이 끊겼을 때 어디서부터 다시 읽을지는 애플리케이션에서 정해야 합니다.
- **브라우저와 외부 파트너가 쓰는 API.** 다양한 클라이언트가 접근하고 HTTP 도구로 요청과 응답을 쉽게 확인해야 한다면 HTTP REST API가 잘 맞습니다. 예를 들어 외부 주문 조회는 REST로 제공하고, 요청을 받은 백엔드가 내부 재고 서비스에는 gRPC로 접근할 수 있습니다.

### 주문 조회와 상태 변경 구독을 실행해 보기

같은 주문이라도 현재 상태가 한 번만 필요하다면 `GetOrder`로 충분합니다. 배송 화면처럼 상태가 바뀔 때마다 갱신해야 한다면 `WatchOrder`를 추가할 수 있습니다. 아래 Python 예제에서는 두 호출을 같은 서버에서 실행합니다.

앞에서 본 `.proto`를 `orders.proto`로 저장하고 `OrderService` 정의 안에 다음 줄을 추가합니다. 요청 메시지에는 주문 ID를 그대로 사용하고, 응답 타입 앞에 `stream`을 붙여 서버가 여러 번 응답할 수 있게 합니다.

```proto
rpc WatchOrder(GetOrderRequest) returns (stream Order);
```

같은 디렉터리에서 Python용 메시지 클래스와 서비스 코드를 생성합니다. [gRPC Python 공식 튜토리얼](https://grpc.io/docs/languages/python/basics/)

```bash
python -m pip install grpcio grpcio-tools
python -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. orders.proto
```

`server.py`에서 주문 `123`의 현재 상태를 반환하고, 구독 요청에는 1초 간격으로 바뀌는 예제 상태를 보냅니다.

```python
from concurrent import futures
from time import sleep

import grpc
import orders_pb2
import orders_pb2_grpc


class OrderService(orders_pb2_grpc.OrderServiceServicer):
    def GetOrder(self, request, context):
        if request.id != "123":
            context.abort(grpc.StatusCode.NOT_FOUND, "Order not found")

        return orders_pb2.Order(id=request.id, status="PAID")

    def WatchOrder(self, request, context):
        yield self.GetOrder(request, context)
        for status in ("SHIPPED", "DELIVERED"):
            sleep(1)
            if not context.is_active():
                return
            yield orders_pb2.Order(id=request.id, status=status)


server = grpc.server(futures.ThreadPoolExecutor(max_workers=4))
orders_pb2_grpc.add_OrderServiceServicer_to_server(OrderService(), server)
server.add_insecure_port("127.0.0.1:50051")
server.start()
server.wait_for_termination()
```

`client.py`에서는 같은 요청을 두 메서드에 보냅니다. `GetOrder`는 응답 객체를 바로 반환하고, `WatchOrder`는 도착한 메시지를 `for`로 하나씩 읽을 수 있게 합니다. 스트리밍 호출의 `timeout`은 메시지 하나마다 적용되는 시간이 아니라 호출 전체의 데드라인입니다. [gRPC 공식 문서](https://grpc.io/docs/what-is-grpc/core-concepts/)

```python
import grpc
import orders_pb2
import orders_pb2_grpc


with grpc.insecure_channel("127.0.0.1:50051") as channel:
    stub = orders_pb2_grpc.OrderServiceStub(channel)
    request = orders_pb2.GetOrderRequest(id="123")

    order = stub.GetOrder(request, timeout=3)
    print("조회:", order.status)

    for update in stub.WatchOrder(request, timeout=10):
        print("구독:", update.status)
```

한 터미널에서 `python server.py`를 실행한 뒤 다른 터미널에서 `python client.py`를 실행하면 아래와 같이 출력됩니다. `WatchOrder`가 보내는 상태는 예제용으로 정해 둔 값입니다. 실제 배송 화면이라면 주문 상태가 바뀔 때 서버가 그 이벤트를 스트림에 전달합니다.

```text
조회: PAID
구독: PAID
구독: SHIPPED
구독: DELIVERED
```

### 센서 수집과 채팅에서는 어떤 스트림을 쓸까?

- **클라이언트 스트리밍.** 센서가 측정값을 한 건씩 보내고 전송을 끝내면 서버가 수신 건수와 평균값을 한 번 돌려줍니다. 계약은 `rpc UploadReadings(stream Reading) returns (ReadingSummary);`처럼 작성합니다.
- **양방향 스트리밍.** 채팅 클라이언트는 연결을 열고 메시지를 보냅니다. 서버도 같은 호출에서 다른 참여자의 메시지를 전달할 수 있습니다. 계약은 `rpc Chat(stream ChatMessage) returns (stream ChatMessage);`처럼 작성합니다.

주문 조회 하나만 봐서는 어느 쪽을 써도 같은 결과를 얻을 수 있습니다. 선택은 API를 호출하는 주체, 계약을 함께 관리하는 방식, 그리고 호출 중 메시지가 어느 방향으로 몇 번 오가는지에 달려 있습니다.

## 참고 문서

- [gRPC Core concepts, architecture and lifecycle](https://grpc.io/docs/what-is-grpc/core-concepts/)
- [gRPC over HTTP/2](https://grpc.github.io/grpc/core/md_doc__p_r_o_t_o_c_o_l-_h_t_t_p2.html)
- [Roy Fielding, Representational State Transfer](https://ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm)
- [gRPC-Web](https://github.com/grpc/grpc-web)
- [gRPC와 REST의 차이점은 무엇인가요?](https://aws.amazon.com/ko/compare/the-difference-between-grpc-and-rest/)
- [gRPC란 무엇인가요?](https://www.ibm.com/kr-ko/think/topics/grpc)
