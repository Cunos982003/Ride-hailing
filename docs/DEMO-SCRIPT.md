# KỊCH BẢN DEMO 7 PHÚT - RIDE HAILING SYSTEM

## Checklist trước khi quay
- [ ] `docker compose up -d --profile apps` healthy (7 service + infra)
- [ ] `docker compose ps` tất cả `healthy`
- [ ] VPS DNS `ridehailing.duckdns.org` trỏ đúng IP, cert SSL valid
- [ ] Postman collection + environment local/vps đã import
- [ ] `tools/loadtest/drivers.py` chạy 100 driver ảo (cho luồng B)
- [ ] Screen recording software ready (OBS/ShareX)
- [ ] Terminal font size ≥ 14, theme dark

---

## 0:00 – 1:00 | KIẾN TRÚC (1 phút)
**Lệnh demo:**
```bash
docker compose ps
```
**Nói:** "7 microservice độc lập, không shared code, giao tiếp qua RabbitMQ events + Redis GEO. API Gateway ở cổng 8000 route theo path prefix. WS Gateway 2 instance (8001, 8011) HA qua Redis pub/sub."

---

## 1:00 – 3:00 | WEB + 100 TÀI XẾ ẢO (2 phút)
**Lệnh demo:**
```bash
# Terminal 1: Chạy 100 driver ảo
python tools/loadtest/drivers.py --count 100 --duration 120 --ws-url ws://localhost:8001

# Terminal 2: Xem location-service Redis GEO
docker exec ride-hailing-redis-1 redis-cli GEORADIUS drivers:geo 106.7009 10.7769 2000 m COUNT 5
```
**Nói:** "100 driver kết nối WS `/ws/driver`, gửi location 1Hz. Location-service index Redis GEO. Dispatch-service query nearby 2km top 5 khi có ride request."

---

## 3:00 – 4:00 | GHÉP CHUYẾN & THANH TOÁN (1 phút)
**Lệnh demo (Postman/Newman):**
```powershell
cd docs/postman/scripts
.\run-postman.ps1 -Env local -Folder "01 Auth,02 User & Driver,03 Quote,04 Rides,05 Trip actions (tài xế),06 Wallet" -DelayMs 200
```
**Nói khi chạy:** "Khách đăng ký → login → ví topup 500k → quote → đặt xe (idempotent). Dispatch matching sequential near→far, offer qua RabbitMQ `trips.offered` → WS Gateway push `offer` real-time. Tài xế accept → `trips.offered` confirm → dispatch lock Redis + DB unique index chống trùng. Arrive → Start → Complete → `trips.completed` → payment-service trừ khách, cộng hoa hồng tài xế idempotent qua `eventId`."

---

## 4:00 – 5:00 | POSTMAN/NEWMAN (1 phút)
**Lệnh demo:**
```powershell
# Chạy negative tests
.\run-postman.ps1 -Env local -Folder "07 Negative & Security" -DelayMs 500
```
**Nói:** "Collection 7 folder, 40+ request. Pre-request: email random, Idempotency-Key UUID. Tests: status code, field validation, response time, lưu biến. Negative: 401 no token, 403 role, 404 internal via gateway, 409 idempotent/state, 429 rate limit (chạy riêng). HTML report by newman-reporter-htmlextra."

---

## 5:00 – 6:00 | SỐ LIỆU (1 phút)
**Lệnh demo:**
```powershell
cd tools/report
.\collect-evidence.ps1
.\build-report.ps1
cat ../../docs/reports/KET-QUA-KIEM-THU.md
```
**Nói:** "Báo cáo tự động: pass/fail per module, R1–R16 traceability, p50/p95/p99 từ load test, RabbitMQ queue depth, Redis memory. Evidence JSON cho CI artifact."

---

## 6:00 – 7:00 | CI/CD & VPS (1 phút)
**Lệnh demo:**
```bash
# GitHub Actions workflow (xem .github/workflows/ci.yml)
gh run list --limit 5
# VPS deploy
ssh vps "cd ride-hailing && docker compose pull && docker compose up -d --no-deps api-gateway"
curl -s https://ridehailing.duckdns.org/actuator/health | jq .status
```
**Nói:** "CI: build → unit test → integration test → docker build → push GHCR. CD: VPS pull image mới → rolling restart gateway → health check → rollback nếu fail. Backup Postgres hàng ngày qua pg_dump cron."

---

## 10 CÂU HỎI GIẢNG VIÊN + TRẢ LỜI NGẮN

| # | Câu hỏi | Trả lời 30s |
|---|---------|-------------|
| 1 | RabbitMQ thay Kafka? | Latency thấp, routing topic linh hoạt, DLQ built-in, phù hợp request-reply ride-hailing. Kafka overkill. |
| 2 | Busy state ở location-service? | Location là source of truth vị trí real-time; Redis GEO cần biết ONLINE/BUSY để lọc GEOSEARCH. Tách concern. |
| 3 | Chống ghép trùng 3 lớp? | 1) Redis lock serialize offer/driver. 2) DB partial unique index chặn accept song song. 3) Idempotency key trên /rides + outbox eventId. |
| 4 | Outbox pattern làm gì? | Ghi event + nghiệp vụ 1 transaction → đảm bảo at-least-once publish, không mất event khi crash. |
| 5 | Consumer idempotent vì sao? | Publisher confirm + network partition có thể duplicate delivery. `eventId` + PK `processed_event` đảm bảo xử lý 1 lần. |
| 6 | Surge pricing công thức? | `surge = clamp(1 + (demand-supply)/supply * 0.2, 1.0, 3.0)`. Supply=0 → 3. Location error → 1. |
| 7 | WS multi-instance sync? | 2 instance share Redis channel `ws:out:{userId}`. `/internal/push` publish → instance hold session deliver. |
| 8 | Rate limit test tránh 429? | Email random `test-{timestamp}`, Idempotency-Key UUID, delay 200ms. Auth 60 RPM, User 300 RPM. |
| 9 | Rollback deployment? | Blue-green qua Docker tag. `docker compose up -d --no-deps api-gateway` health check → switch DNS. Rollback = tag cũ. |
| 10 | Backup/restore Postgres? | `pg_dump -Fc > backup.dump` ; `pg_restore -c -d db backup.dump`. Volume snapshot cũng được. |

---

## Phụ lục: Lệnh hữu ích
```bash
# Xem logs service
docker compose logs -f dispatch-service

# RabbitMQ management UI
open http://localhost:15672  # guest/guest

# Redis CLI
docker exec -it ride-hailing-redis-1 redis-cli

# Xem trip state trong DB
docker exec -it ride-hailing-postgres-1 psql -U dispatch_app -d dispatch -c "SELECT * FROM trips ORDER BY created_at DESC LIMIT 10;"

# Xem outbox
docker exec -it ride-hailing-postgres-1 psql -U dispatch_app -d dispatch -c "SELECT * FROM outbox WHERE sent_at IS NULL;"
```