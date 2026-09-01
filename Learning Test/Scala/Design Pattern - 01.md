# TypeClass + Strategy  + DI 패턴 예시

```scala
case class EmailNotification(message: String)
case class SmsNotification(message: String)

class MockNotificationGateway {
  def setEmailResponse(seq: Long, response: Option[EmailNotification]): Unit = 
    println(s"email response: seq=$seq, response=$response")

  def setSmsResponse(seq: Long, response: Option[SmsNotification]): Unit = 
    println(s"sms response: seq=$seq, response=$response")
}

trait StubResponseGenerator[A] {
  def set(gateway: MockNotificationGateway, seq: Long, response: Option[A]): Unit
}

object StubResponseGenerator {
  implicit val emailSetter: StubResponseGenerator[EmailNotification] =
    (gateway, seq, response) => gateway.setEmailResponse(seq, response)

  implicit val smsSetter:StubResponseGenerator[SmsNotification] =
    (gateway, seq, response) => gateway.setSmsResponse(seq, response)
}

object Example extends App {
  val gateway = new MockNotificationGateway

  def setStubResponse[A](gateway: MockNotificationGateway, seq: Long, response: Option[A])(implicit generator: StubResponseGenerator[A]): Unit =
    generator.set(gateway, seq, response)

  val email = EmailNotification("메일 내용")
  val sms = SmsNotification("문자 내용")

  setStubResponse(gateway, 1L, Some(email))
  setStubResponse(gateway, 2L, Some(sms))
}
```
- setStubResponse(gateway, 1L, Some(email))
```
setStubResponse(gateway, 1L, Some(email))setStubResponse(gateway, 2L, Some(sms))Some(email)
    ↓
A = EmailNotification
    ↓
emailSetter 자동 선택
    ↓
generator.set(...)
    ↓
gateway.setEmailResponse(...)
```
- setStubResponse(gateway, 2L, Some(sms))
```
Some(sms)
    ↓
A = SmsNotification
    ↓
smsSetter 자동 선택
    ↓
generator.set(...)
    ↓
gateway.setSmsResponse(...)
```

## smsSetter는 익명 객체로 SAM으로 정의된 StubResponseGenerator[A] 트레이트를 구현한 객체이다.
- Scala Code
```scala
implicit val smsSetter: StubResponseGenerator[SmsNotification] =
  (gateway, seq, response) => gateway.setSmsResponse(seq, response)
```
- Desugar Scala Code
```scala
new StubResponseGenerator[SmsNotification] {
  override def set(
      gateway: MockNotificationGateway,
      seq: Long,
      response: Option[SmsNotification]
  ): Unit = {
    gateway.setSmsResponse(seq, response)
  }
}
```

### Type Class 패턴
- trait StubResponseGenerator[A]에서 타입 A별 처리 능력을 추상화합니다.
- emailSetter와 smsSetter가 각각 EmailNotification, SmsNotification에 대한 구현을 제공합니다.
- 적용 위치: StubResponseGenerator[A] 선언부와 object StubResponseGenerator 내부의 두 implicit val.

### Strategy 패턴
- emailSetter와 smsSetter는 동일한 set 메서드를 사용하지만 서로 다른 Gateway 메서드를 실행합니다.
- emailSetter는 setEmailResponse, smsSetter는 setSmsResponse를 호출합니다.
- 적용 위치: 각 implicit val의 람다 본문과 setStubResponse 내부의 generator.set(...).

### Implicit Dependency Injection
- setStubResponse[A](...)(implicit generator: StubResponseGenerator[A])에서 setter를 외부에서 직접 전달받지 않습니다.
- 호출 시 A 타입에 맞는 emailSetter 또는 smsSetter를 컴파일러가 자동으로 주입합니다.
- 적용 위치: setStubResponse의 implicit 파라미터와 setStubResponse(gateway, ..., Some(email/sms)) 호출부.
