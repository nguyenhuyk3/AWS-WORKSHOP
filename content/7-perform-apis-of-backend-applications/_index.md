---
title : "Thực hiện các API của các ứng dụng Backend"
date : "`r Sys.Date()`"
weight : 7
chapter : false
pre : " <b> 7 </b> "
---

**Nội dung:**

- [Tổng quan](#tổng-quan)
- [Thực hiện đăng kí tài khoản người dùng](#thực-hiện-đăng-kí-tài-khoản-người-dùng)
- [Thực hiện thêm phim](#thực-hiện-thêm-phim)
- [Thực hiện đặt vé xem phim](#thực-hiện-đặt-vé-xem-phim)
- [Tổng kết](#tổng-kết)

---

### Tổng quan

Sau khi thực hiện xong các bước trên thì trong phần này chúng sẽ thực hiện gọi các APIs với tư cách là 1 máy khách.

---

### Thực hiện đăng kí tài khoản người dùng

Thực hiện đăng kí người dùng với email

![alt text](1-registration.png)

Chúng ta có thể thấy rằng sau khi người đăng kí với email **huykimcuong5@gmail.com** thì ở **Mail service** nhận được message từ **Kafka** và thực hiện việc gửi mail với mã OTP. 

![alt text](2-mail_service_receiving_message.png)
![alt text](3-mail_service_sending_mail.png)

Hoàn thành các bước tiếp theo

![alt text](4-completing_registration.png)

Có thể thấy chúng ta đã thực hiện 3 APIs để hoàn thành việc đăng kí.

![alt text](5-completing-registration_with_3_apis.jpg)

---

### Thực hiện thêm phim

Đăng nhập tài khoản với quyền admin.

![alt text](7_logging_in_with_admin_account.png)

Thêm phim với quyền admin.

![alt text](10_adding_film.png)

Khi thực hiện thêm phim thì ảnh và trailer của phim sẽ được tải lên S3 và S3 sẽ đưa sự kiện Put Object vào SNS và SQS sẽ nhận message từ SNS. Và upload service sẽ nhận message từ SQS.

![alt text](11_upload_service_getting_message_from_sqs.jpg)

Và upload service sẽ ghi message vào **Kafka** với những thông tin cần thiết để cho product service cập nhập URL của ảnh và trailer vào database.

![alt text](12_product_service_getting_message_from_kafka_to_upadte_information.jpg)

Và ta truy cập vào database để xem những thông tin về ảnh và trailer của phim.

![alt text](13_updating_information_in_database.png)

Cũng tương tự như với FAB (Food And Beverage)

![alt text](14_adding_fab.png)

Ở product service thì sẽ nhận message từ upload service khi chúng ta tải ảnh lên S3 thành công.

![alt text](15_upadting_fab_information_for_fab.png)

### Thực hiện đặt vé xem phim

Trước tiên chúng ta sẽ phải thêm showtime mới. Và chúng ta sẽ thực hiện 1 API gọi là Realease showtime. Sau đó chúng ta và database kiểm tra ghế cho showtime đó.

![alt text](16_adding_new_showtime.png)
![alt text](17_checking_seats.png)
![alt text](18_checking_seats.png)

Kiểm tra ghế có showtime id = 3.

![alt text](19_getting_seats_with_showtime_id.png)

Chúng ta sẽ thực hiện API để đặt ghế với showtime id = 3. Sau khi đặt ghế thành công thì sẽ được trả order id về.

![alt text](20_booking_seat.png)

Sau khi gọi API tạo payment URL với order id vừa được tạo khi thực hiện API tạo order.

![alt text](21_creating_payment_url.png)

Sau khi thanh toán thành công thì payment service trả về thông tin của vé xem phim.

![alt text](22_creating_ticket_information.png)

Và chúng ta sẽ cùng kiểm tra các topic mà chúng ta đã tạo thủ công với tên lần lượt là: **bmt_payment.public.outboxes**, **bmt_order.public.outboxes**.

![alt text](23_payment_outbox.png)
![alt text](24_order_outbox.png)

Như có thể thấy trong ảnh thì trong cả 2 topic thì đều có những message, chính những message này đã được những service khác tiêu thụ và xử lí theo nội dung của message.

---

## Tổng kết

Trong phần này, chúng ta đã đóng vai trò là một máy khách để thực hiện toàn bộ luồng sử dụng của hệ thống backend, từ việc đăng ký tài khoản người dùng đến đặt vé xem phim và xử lý thanh toán. Qua từng bước, chúng ta đã thấy rõ:

- Luồng đăng ký tài khoản được thực hiện thông qua chuỗi các API liên quan đến xác thực và xác minh OTP. Sau khi người dùng đăng ký, Mail Service đã nhận message từ Kafka và gửi OTP đến email thông qua Amazon SES.
- Luồng thêm phim với quyền admin bao gồm việc tải ảnh và trailer lên Amazon S3, nơi sự kiện PutObject kích hoạt SNS → SQS → Upload Service. Sau đó, Upload Service gửi message vào Kafka để Product Service cập nhật URL hình ảnh và video vào cơ sở dữ liệu.
- Luồng đặt vé xem phim bắt đầu từ việc tạo showtime, kiểm tra ghế theo showtime_id, thực hiện đặt vé, tạo đơn hàng và nhận URL thanh toán. Sau khi thanh toán thành công, hệ thống đã tạo vé xem phim và lưu thông tin liên quan.
- Cơ chế giao tiếp giữa các service sử dụng Kafka và SQS giúp hệ thống hoạt động theo mô hình event-driven, đảm bảo sự tách biệt giữa các thành phần và khả năng mở rộng linh hoạt.
- Cuối cùng, chúng ta cũng đã xác nhận rằng các message được ghi vào các Kafka topic như bmt_payment.public.outboxes và bmt_order.public.outboxes đã được các service tiêu thụ thành công để xử lý các nghiệp vụ liên quan.

Phần thực hành này giúp ta hiểu rõ hơn về cách hệ thống backend vận hành trong môi trường phân tán, sử dụng các dịch vụ như Kafka, S3, SQS và API Gateway một cách hiệu quả và đồng bộ.