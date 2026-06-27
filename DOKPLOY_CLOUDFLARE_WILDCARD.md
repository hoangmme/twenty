# Triển khai Twenty CRM wildcard với Cloudflare và Dokploy

Tài liệu này mô tả cấu hình production hiện tại cho Twenty multi-workspace:

- Trang đăng nhập chính: `https://crm.mme.vn`
- Workspace tự sinh: `https://<slug>-crm.mme.vn`
- Ví dụ: `https://glad-mustard-llama-crm.mme.vn`
- Các dịch vụ khác như `mme.mme.vn` không bị router của deployment này bắt nhầm.

## Luồng request

```text
Trình duyệt (HTTPS)
  -> Cloudflare: DNS *.mme.vn + SSL rule *crm.mme.vn/*
  -> HTTP port 80 tại server (SSL mode Flexible)
  -> Traefik của Dokploy
  -> service server, port 3000
  -> Twenty CRM
```

Twenty được cấu hình với domain gốc `mme.vn` và default subdomain `crm`. Vì vậy Twenty tạo URL đăng nhập chính là `crm.mme.vn`. Các workspace mới được thêm hậu tố `-crm`, tạo URL một tầng như `<slug>-crm.mme.vn` thay vì `<slug>.crm.mme.vn`.

Cách làm phẳng hostname này giúp chứng chỉ wildcard `*.mme.vn` tại Cloudflare phủ được cả trang đăng nhập và workspace.

## 1. Cloudflare DNS

Tạo một DNS record:

| Thuộc tính | Giá trị |
| --- | --- |
| Type | `A` |
| Name | `*` |
| IPv4 address | IP public của server Dokploy |
| Proxy status | Proxied (mây cam) |
| TTL | Auto |

Kết quả tương đương với `*.mme.vn -> IP server`.

Không dùng `*crm` hoặc `*crm.mme.vn` làm tên DNS wildcard. DNS wildcard phải dùng dấu `*` cho toàn bộ label. Nếu đã có record riêng cho `crm.mme.vn`, record riêng đó được ưu tiên hơn wildcard và phải trỏ về đúng server Dokploy.

## 2. Cloudflare SSL rule

Trong Cloudflare Page Rules hoặc rule SSL tương ứng, cấu hình:

```text
URL pattern: *crm.mme.vn/*
SSL mode: Flexible
Status: Enabled
```

Pattern này nhận cả:

- `crm.mme.vn/*` vì dấu `*` có thể khớp chuỗi rỗng.
- `glad-mustard-llama-crm.mme.vn/*` vì dấu `*` khớp phần `glad-mustard-llama-`.

Không cần dùng `*.mme.vn/*` cho SSL rule vì pattern đó rộng hơn cần thiết và có thể áp dụng Flexible cho các dịch vụ khác trong zone.

> `Flexible` nghĩa là trình duyệt kết nối HTTPS tới Cloudflare, còn Cloudflare kết nối HTTP tới origin. Khi chuyển sang `Full (strict)`, phải cấu hình certificate và router `websecure` tại Dokploy trước.

## 3. Dokploy Domains

Trong tab **Domains** của Compose deployment:

```text
Để trống — không thêm *.mme.vn và không cần thêm crm.mme.vn
```

Không nhập `*.mme.vn` trong Dokploy UI. Domain UI có thể sinh rule dạng `Host(`*.mme.vn`)`, coi dấu `*` là ký tự literal và không khớp hostname thật. Kết quả thường là Traefik trả `404 page not found`.

Routing được khai báo trực tiếp trong [`docker-compose.yml`](./docker-compose.yml):

```yaml
labels:
  - "traefik.enable=true"
  - 'traefik.http.routers.mmetwenty-flat-crm.rule=Host(`crm.mme.vn`) || HostRegexp(`^[a-z0-9-]+-crm\.mme\.vn$`)'
  - "traefik.http.routers.mmetwenty-flat-crm.entrypoints=web"
  - "traefik.http.routers.mmetwenty-flat-crm.priority=100"
  - "traefik.http.routers.mmetwenty-flat-crm.service=mmetwenty-flat-crm"
  - "traefik.http.services.mmetwenty-flat-crm.loadbalancer.server.port=3000"
```

Router chỉ nhận đúng hai nhóm:

1. `crm.mme.vn`
2. Hostname kết thúc bằng `-crm.mme.vn`

Nó không bắt toàn bộ `*.mme.vn`, nên có thể chạy song song với CRM hoặc ứng dụng khác trên cùng Dokploy.

## 4. Environment của Dokploy

Đặt các biến sau trong phần Environment của Compose deployment:

```env
SERVER_URL=https://mme.vn
IS_MULTIWORKSPACE_ENABLED=true
DEFAULT_SUBDOMAIN=crm
IS_WORKSPACE_CREATION_LIMITED_TO_SERVER_ADMINS=true
```

Không đặt `SERVER_URL=https://crm.mme.vn`. Với multi-workspace, Twenty tự nối `DEFAULT_SUBDOMAIN` hoặc workspace subdomain vào hostname của `SERVER_URL`.

File mẫu đầy đủ nằm tại [`.env.dokploy`](./.env.dokploy).

## 5. Deploy

Sau khi cập nhật Environment hoặc domain routing:

1. Xóa domain cũ khỏi tab Domains của Dokploy.
2. Lưu Environment.
3. Redeploy Compose để Dokploy nạp lại Traefik labels.
4. Đợi `server`, `worker`, `db` và `redis` healthy.
5. Mở `https://crm.mme.vn` và đăng ký hoặc đăng nhập lại.

Khi Twenty tạo workspace mới, không cần thêm DNS record hay domain Dokploy. Ví dụ workspace `glad-mustard-llama-crm` sẽ tự dùng:

```text
https://glad-mustard-llama-crm.mme.vn
```

Cloudflare wildcard DNS đưa request về server, còn `HostRegexp` của Traefik chuyển request tới đúng container Twenty.

### Reset dữ liệu bằng volume version

Compose hiện dùng `twenty-db-data-v8` và `server-local-data-v8`. Việc đổi tên volume từ một version cũ sang version mới làm Dokploy tạo database và local storage sạch ở lần deploy tiếp theo.

Không tăng version volume khi chỉ muốn redeploy code, vì Twenty sẽ khởi động như một installation mới. Volume cũ vẫn còn trên server cho đến khi được xóa thủ công và có thể dùng để rollback bằng cách khôi phục tên volume cũ trong Compose.

## 6. Workspace có nhiều hậu tố `-crm`

Bản cũ dùng một lệnh `sed` không idempotent nên mỗi lần container restart có thể nối thêm `-crm`:

```text
glad-mustard-llama-crm
glad-mustard-llama-crm-crm
```

Startup command hiện tại đã kiểm tra và chuẩn hóa patch để không nối lặp ở các lần restart sau. Thay đổi này chỉ sửa cách tạo workspace mới; nó không tự đổi subdomain đã lưu trong database.

Với workspace cũ, đổi subdomain trong **Settings -> Domains** từ `...-crm-crm` thành `...-crm`, sau đó yêu cầu một login link mới.

## 7. Kiểm tra và xử lý lỗi

Kiểm tra DNS:

```bash
dig +short crm.mme.vn
dig +short test-crm.mme.vn
```

Kiểm tra HTTP:

```bash
curl -I https://crm.mme.vn/
curl -I https://test-crm.mme.vn/
```

Nếu nhận `404 page not found` dạng text thuần, request thường đã qua Cloudflare nhưng Traefik chưa khớp router. Kiểm tra lại:

- Tab Domains của Dokploy đã để trống.
- Compose đã redeploy sau khi thêm labels.
- Service `server` đã nối vào `dokploy-network`.
- Router dùng entrypoint `web` khi Cloudflare đang ở Flexible.
- Hostname workspace thật sự kết thúc bằng `-crm.mme.vn`.

Nếu DNS không resolve, kiểm tra record `A` tên `*` và đảm bảo không có record cụ thể khác ghi đè hostname cần dùng.

## Tài liệu tham khảo

- [Cloudflare: Wildcard DNS records](https://developers.cloudflare.com/dns/manage-dns-records/reference/wildcard-dns-records/)
- [Cloudflare: Wildcard matching in Page Rules](https://developers.cloudflare.com/rules/page-rules/reference/wildcard-matching/)
- [Dokploy: Docker Compose domains](https://docs.dokploy.com/docs/core/docker-compose/domains)
- [Twenty: Multi-workspace setup](https://docs.twenty.com/developers/self-host/capabilities/setup)
