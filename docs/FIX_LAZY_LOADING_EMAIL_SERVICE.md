# FIX: NullPointerException - Lazy Loading in Email Service

**Date**: 2025-11-23  
**Error**: `NullPointerException: Cannot invoke "com.mysql.cj.protocol.ColumnDefinition.getFields()"`

---

## 🔴 Vấn đề

### Lỗi gặp phải:
```
java.lang.NullPointerException: Cannot invoke "com.mysql.cj.protocol.ColumnDefinition.getFields()" 
because "this.columnDefinition" is null
	at com.mysql.cj.jdbc.result.ResultSetImpl.checkColumnBounds(ResultSetImpl.java:519)
	...
	at com.swp.evchargingstation.service.EmailService.sendChargingStartEmail(EmailService.java:46)
```

### Nguyên nhân:
1. **Lazy Loading Issue**: `ChargingSession.driver.user` được fetch lazily
2. **Async Execution**: Email service chạy async trong thread riêng
3. **Transaction đã đóng**: Khi async method chạy, Hibernate session đã đóng
4. **Proxy không load được**: Hibernate proxy cố gắng load User nhưng ResultSet đã null

### Flow gây lỗi:
```
ChargingSessionService.startSession()
  ↓
  publishEvent(ChargingSessionStartedEvent) 
  ↓
  [Transaction COMMIT - Hibernate session đóng]
  ↓
  ChargingSessionEventListener.sendStartNotification() [@Async]
  ↓
  EmailService.sendChargingStartEmail() [@Async]
  ↓
  session.getDriver().getUser() ← ❌ LazyInitializationException
```

---

## ✅ Giải pháp

### 1. Thêm Query Fetch Eager trong ChargingSessionRepository

**File**: `ChargingSessionRepository.java`

```java
/**
 * Fetch ChargingSession with Driver and User eagerly for email sending
 * Prevents lazy loading issues in async operations
 */
@Query("SELECT cs FROM ChargingSession cs " +
       "LEFT JOIN FETCH cs.driver d " +
       "LEFT JOIN FETCH d.user " +
       "LEFT JOIN FETCH cs.chargingPoint cp " +
       "LEFT JOIN FETCH cp.station " +
       "WHERE cs.sessionId = :sessionId")
java.util.Optional<ChargingSession> findByIdWithUserEager(@Param("sessionId") String sessionId);
```

### 2. Inject ChargingSessionRepository vào EmailService

**File**: `EmailService.java`

```java
@Service
@Slf4j
@FieldDefaults(level = AccessLevel.PRIVATE)
public class EmailService {

    final JavaMailSender mailSender;
    final ChargingSessionRepository chargingSessionRepository; // ← Thêm dependency

    @Value("${mail.from}")
    String fromEmail;

    public EmailService(JavaMailSender mailSender, 
                       ChargingSessionRepository chargingSessionRepository) {
        this.mailSender = mailSender;
        this.chargingSessionRepository = chargingSessionRepository;
    }
}
```

### 3. Refetch Session với Eager Loading

**File**: `EmailService.java`

```java
@Async
@Transactional(propagation = Propagation.REQUIRES_NEW, readOnly = true)
public void sendChargingStartEmail(ChargingSession session) {
    try {
        // ✅ Refetch session với eager loading để tránh lazy loading
        ChargingSession freshSession = chargingSessionRepository
                .findByIdWithUserEager(session.getSessionId())
                .orElse(session);
        
        User user = freshSession.getDriver().getUser();
        if (user == null || user.getEmail() == null) {
            log.warn("Cannot send email: User or email is null for session {}", 
                    freshSession.getSessionId());
            return;
        }

        String subject = "Phiên sạc của bạn đã bắt đầu";
        String htmlContent = buildChargingStartEmailTemplate(freshSession);

        sendHtmlEmail(user.getEmail(), subject, htmlContent);
        log.info("Sent charging start email to {} for session {}", 
                user.getEmail(), freshSession.getSessionId());
    } catch (Exception e) {
        log.error("Failed to send charging start email for session {}: {}", 
                session.getSessionId(), e.getMessage(), e);
    }
}

@Async
@Transactional(propagation = Propagation.REQUIRES_NEW, readOnly = true)
public void sendChargingCompleteEmail(ChargingSession session) {
    try {
        // ✅ Refetch session với eager loading
        ChargingSession freshSession = chargingSessionRepository
                .findByIdWithUserEager(session.getSessionId())
                .orElse(session);
        
        // ... rest of the code
    } catch (Exception e) {
        log.error("Failed to send charging complete email: {}", e.getMessage(), e);
    }
}
```

### 4. Thêm @Transactional với REQUIRES_NEW

```java
@Async
@Transactional(propagation = Propagation.REQUIRES_NEW, readOnly = true)
public void sendChargingStartEmail(ChargingSession session) {
    // ← Transaction mới được mở trong async thread
    // ← Cho phép refetch data từ database
}
```

---

## 🔧 Cách hoạt động

### Trước khi fix (❌ Lỗi):
```
Main Thread:
  - Start session
  - Commit transaction ✓
  - Close Hibernate session ✓
  
Async Thread:
  - Receive ChargingSession proxy
  - Try to access session.driver.user ❌
  - LazyInitializationException!
```

### Sau khi fix (✅ OK):
```
Main Thread:
  - Start session
  - Commit transaction ✓
  - Close Hibernate session ✓
  - Publish event with sessionId
  
Async Thread:
  - Open NEW transaction ✓
  - Refetch session WITH EAGER JOIN ✓
  - User, Driver, Station all loaded ✓
  - Send email successfully ✓
  - Commit transaction ✓
```

---

## 📊 So sánh

| Approach | Main Transaction | Async Transaction | Lazy Loading | Result |
|----------|-----------------|-------------------|--------------|--------|
| **Cũ** | Load Session (lazy) | ❌ Không có | ❌ Fail | ❌ Error |
| **Mới** | Load Session (lazy) | ✅ REQUIRES_NEW | ✅ Eager fetch | ✅ OK |

---

## 🧪 Testing

### Test Case 1: Start Charging Session
```bash
# Start a charging session
POST /api/sessions/start

# Expected:
✅ Session started successfully
✅ Email sent to user
✅ No NullPointerException in logs
```

### Test Case 2: Complete Charging Session
```bash
# Complete a charging session
POST /api/sessions/{sessionId}/stop

# Expected:
✅ Session completed
✅ Completion email sent
✅ No lazy loading errors
```

### Kiểm tra logs:
```
✅ [Event] Sending start email for session: xxx
✅ [Event] Start email sent successfully for session: xxx
✅ Sent charging start email to user@example.com for session xxx

❌ KHÔNG CÒN: Failed to send charging start email: NullPointerException
```

---

## 📝 Files Changed

1. ✅ `ChargingSessionRepository.java` - Added `findByIdWithUserEager()`
2. ✅ `EmailService.java` - Refetch with eager loading + @Transactional

---

## 🚨 Lưu ý quan trọng

### 1. **Propagation.REQUIRES_NEW**
- Mở transaction MỚI trong async thread
- Không phụ thuộc vào transaction của caller
- Cho phép fetch data từ database

### 2. **Eager Fetch Query**
- `LEFT JOIN FETCH` load tất cả relationships cần thiết
- Tránh N+1 query problem
- Chỉ 1 query để load all data

### 3. **Async + Transactional**
- `@Async` phải đi với `@Transactional(REQUIRES_NEW)`
- Không dùng transaction của main thread
- Mỗi async method có transaction riêng

### 4. **Fallback Strategy**
- `findByIdWithUserEager().orElse(session)`
- Nếu refetch fail, dùng session cũ
- Graceful degradation

---

## 🎯 Kết quả

Sau khi fix:
- ✅ Email được gửi thành công
- ✅ Không còn NullPointerException
- ✅ Async operation hoạt động bình thường
- ✅ User data được load đầy đủ
- ✅ Performance tốt (single query with joins)

---

## 📚 Best Practices Learned

1. **Lazy Loading + Async = ❌ Bad Combo**
   - Luôn eager fetch data trước khi async
   - Hoặc refetch trong async với transaction mới

2. **Event Publishing**
   - Publish sau COMMIT với `@TransactionalEventListener(AFTER_COMMIT)`
   - Đảm bảo data đã được persist trước khi async chạy

3. **Repository Queries**
   - Tạo query riêng cho async operations
   - Sử dụng `JOIN FETCH` để load relationships

4. **Transaction Management**
   - Async methods cần `REQUIRES_NEW` propagation
   - Không share transaction giữa threads

---

## 🔗 Related Issues

- Lazy Loading trong Hibernate
- Async execution với Spring Events
- Transaction propagation
- N+1 query problem

---

**Fix Status**: ✅ RESOLVED  
**Root Cause**: Lazy loading trong async context  
**Solution**: Refetch với eager loading + new transaction

