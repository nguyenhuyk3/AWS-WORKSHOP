---
title : "Routing API với Kong Gateway"
date : "`r Sys.Date()`"
weight : 6
chapter : false
pre : " <b> 6 </b> "
---

**Nội dung:**

- [Tổng quan](#tổng-quan)
- [Các bước thực hiện](#các-bước-thực-hiện)
- [Tạo service](#tạo-service)
- [Tạo route](#tạo-route)
- [Tạo JWT plugin](#tạo-jwt-plugin)
- [Tạo Pre function plugin](#tạo-pre-function-plugin)
- [Tổng kết](#-tổng-kết)

### Tổng quan

Trong phần này chúng ta sẽ thực hiện 1 số api để định tuyến API với mục đích là để cho máy khách có thể gọi API tới các dịch vụ của chúng ta mà sẽ không gọi trực tiếp đến các dịch vụ nằm trong Private Applicaton Subnet. Đây cũng là 1 thiết kế phổ biến trong kiến trúc microservices.

---

### Các bước thực hiện

Để máy khách có thể gọi tới 1 api của ứng dụng backend của chúng ta thì chúng ta sẽ thực hiện những bước sau đây:

1. Tạo service
2. Tao route

Ngoài ra đối với những API cần xác thực JWT thì chúng ta sẽ tạo **JWT plugin** cho route này hay kiểm tra 1 số thông tin từ Token JWT như kiểm tra Role có phù hợp với hành động này hay không thì chúng ta sẽ tạo **Pre-function**.

### Tạo service

1. Chúng ta sẽ vào **Post man** và thực hiện API để tạo service
2. Chúng ta sẽ gọi api http://<PUBLIC_IP_OF_KONG_GATEWAY>:8001/services với phương thức **POST** (Tôi khuyên là chúng ta nên cấu hình **Security group** của **EC2** đang host **Kong gateway** được truy cập vào cổng 8001 với 1 số ip nhất định như chinh ip của máy mình):

```json
{
  "name": "<NAME_OF_SERVICE>",
  "protocol": "http",
  "host": "<PRIVATE_IP_OF_SERVICE>",
  "port": "<PORT_OF_SERVICE>", // Chúng ta sẽ sử dụng kiểu số ở đây
  "path": "PATH_OF_API",
  "retries": 5,
  "connect_timeout": 60000,
  "write_timeout": 60000,
  "read_timeout": 60000
}
```

---

### Tạo route

1. Chúng ta sẽ vào **Post man** và thực hiện API để tạo service
2. Chúng ta sẽ gọi api http://<PUBLIC_IP_OF_KONG_GATEWAY>:8001/routes với phương thức **POST**:

````json
{
  "name": "<NAME_OF_ROUTE>",
  "service": {
    "name": "<NAME_OF_SERVICE>"
  },
  "protocols": ["http"], // Chúng ta có thể chọn nhiều protocols khác như http, https, gRPC,...
  "methods": ["GET"], // Chúng ta có thể thêm nhiều methods khác như GET, POST, PATHC,...
  "paths": ["<PATH_OF_API_THAT_WE_WANT_CLIENT_USE>"],
  "strip_path": true,
  "preserve_host": false,
  "request_buffering": true,
  "response_buffering": true,
  "https_redirect_status_code": 426,
  "regex_priority": 0,
  "path_handling": "v0"
}
````

---

### Tạo JWT plugin

1. Trước khi tạo JWT thì chúng ta sẽ tạo 1 **Consummer** trước.

**Consummer là gì?**

Là đối tượng đại diện cho client truy cập API, như:

- Ứng dụng di động.
- Frontend web.
- Đối tác (partner system).
- Một người dùng cụ thể.

2. Tạo **Consummer** với API http://<PUBLIC_IP_OF_KONG_GATEWAY>:8001/consumers với phương **POST**:

````json
{
  "username": "jwt",
  "custom_id": "f0a221b9-d74f-44ac-93b6-50c8fd4259a7"
}
````

3. Tạo **JWT Plugin** với API http://<PUBLIC_IP_OF_KONG_GATEWAY>:8001/routes/<ID_OF_ROUTE>/plugins với phương thức **POST**. Và để lấy **<ID_OF_ROUTE>** chúng ta sẽ thực hiện API http://<PUBLIC_IP_OF_KONG_GATEWAY>:8001/routes với phương thức **GET**.

```json
{
    "name": "jwt",
    "config": {
      "run_on_preflight": true,
      "secret_is_base64": false,
      "key_claim_name": "iss",
      "claims_to_verify": ["exp"],
      "anonymous": null,
      "maximum_expiration": 0,
      "header_names": ["authorization"]
    },
    "enabled": true
}
```

---

### Tạo Pre function plugin

1. Chúng ta sẽ thực hiện API http://<PUBLIC_IP_OF_KONG_GATEWAY>:8001/routes/<ID_OF_ROUTE>/plugins.

```json
{
  "name": "pre-function",
  "config": {
    "access": [
      "-- Get JWT token from header Authorization\nlocal auth_header = kong.request.get_header(\"Authorization\")\nif not auth_header then\n  return kong.response.exit(401, { message = \"missing Authorization header\" })\nend\n\n-- Get token from \"Bearer <token>\"\nlocal token = auth_header:match(\"^Bearer%s+(.+)$\")\nif not token then\n  return kong.response.exit(401, { message = \"invalid Token format\" })\nend\n\n-- Split token into 3 parts: header.payload.signature\nlocal token_parts = {}\nfor part in token:gmatch(\"[^.]+\") do\n  table.insert(token_parts, part)\nend\n\nif #token_parts ~= 3 then\n  return kong.response.exit(401, { message = \"invalid JWT token structure\" })\nend\n\n-- Decode payload (middle part of JWT)\nlocal payload_str = ngx.decode_base64(token_parts[2])\nif not payload_str then\n  return kong.response.exit(401, { message = \"failed to decode JWT payload\" })\nend\n\n-- Extract \"role\" from JSON payload\nlocal role = payload_str:match('\"role\"%s*:%s*\"([^\"]+)\"')\nif not role then\n  return kong.response.exit(403, { message = \"missing role in token\" })\nend\n\n-- Check role, only allow customer\nif role ~= \"customer\" then\n  return kong.response.exit(403, { message = \"access denied: requires customer role\" })\nend\n\nreturn"
    ] // Đây là code viết bằng lua và field access là nơi chúng ta sẽ đặt những code lua cho những mục đích kiểm tra của chúng ta. Và code trên kiểm tra Token JWT có chứa role là customer hay không và gán nó vòa Header của APi.
  }
}
```
---

### Tổng kết

Trong phần này, chúng ta đã tìm hiểu cách sử dụng Kong Gateway để định tuyến và bảo vệ API trong môi trường kiến trúc microservices, cụ thể:

- Tạo Service: khai báo dịch vụ backend thực tế mà Kong sẽ định tuyến đến.
- Tạo Route: cấu hình đường dẫn API mà client sẽ gọi, và ánh xạ nó đến service tương ứng.
- Tạo Consumer: đại diện cho client hoặc hệ thống sử dụng API, dùng để gắn thông tin xác thực.
- Cấu hình JWT Plugin: đảm bảo chỉ client có JWT hợp lệ mới có thể truy cập API, giúp tăng cường bảo mật.
- Cấu hình Pre-function Plugin: viết logic Lua tùy chỉnh để kiểm tra thêm thông tin trong JWT (ví dụ: kiểm tra quyền truy cập thông qua trường role trong payload).

Những bước này giúp đảm bảo rằng:

- Hệ thống của bạn được định tuyến hợp lý.
- Các dịch vụ backend nằm trong Private Subnet vẫn có thể được truy cập một cách an toàn và kiểm soát được thông qua Kong Gateway.
-Bạn có thể dễ dàng kiểm soát, giới hạn và giám sát các request đến hệ thống.

👉 Đây là cách tiếp cận chuẩn và phổ biến trong việc triển khai các API Gateway cho hệ thống microservices hiện đại.
