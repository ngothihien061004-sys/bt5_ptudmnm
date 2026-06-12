# BÀI TẬP VỀ NHÀ SỐ 5: 
1. DOCKER LÀ GÌ?
Docker là một nền tảng mã nguồn mở cho phép lập trình viên và quản trị viên hệ thống tự động hóa việc đóng gói, triển khai và chạy các ứng dụng bên trong các môi trường ảo hóa cô lập, nhẹ nhàng gọi là Container.

Để hiểu rõ Docker, cần phân biệt giữa Máy ảo truyền thống (Virtual Machine - VM) và Docker Container:

Máy ảo (VM): Chạy trên một phần mềm phân luồng phần cứng (Hypervisor). Mỗi VM bắt buộc phải cài đặt một Hệ điều hành khách hoàn chỉnh (Guest OS) cùng toàn bộ thư viện. Điều này khiến VM rất nặng (vài GB đến vài chục GB), khởi động lâu (vài phút) và ngốn nhiều RAM/CPU của máy thật.

Docker Container: Các container hoạt động bằng cách dùng chung nhân hệ điều hành (Kernel) của máy Host. Nó chỉ cô lập ở tầng ứng dụng nhờ vào các công nghệ cốt lõi của Linux (Namespaces để cô lập tài nguyên và Cgroups để giới hạn phần cứng). Nhờ đó, Container cực kỳ nhẹ (chỉ vài chục MB), khởi động trong vài giây và tiêu tốn rất ít tài nguyên phần cứng.

2. CÁC KEYWORD TRONG docker-compose.yml
Tệp docker-compose.yml viết bằng định dạng YAML, được sử dụng để định nghĩa và quản lý một hệ thống gồm nhiều Container hoạt động cùng nhau (Multi-container).

A. Từ khóa cấp cao nhất (Root Level)
services: Định nghĩa danh sách các container (dịch vụ) sẽ được khởi tạo trong hệ thống.

networks: Định nghĩa các mạng nội bộ ảo để các container kết nối và truyền thông tin bảo mật với nhau.

volumes: Định nghĩa các vùng lưu trữ dữ liệu vĩnh viễn, giúp dữ liệu không bị mất đi khi container bị xóa hoặc restart.
B. Ví dụ minh họa cấu trúc tệp docker-compose.yml tổng thể
```YAML
version: '3.8'

services:
  # Dịch vụ Cơ sở dữ liệu
  mariadb:
    image: mariadb:10.6
    container_name: mariadb_service
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: root_password
      MYSQL_DATABASE: monitor_db
    ports:
      - "3306:3306"
    volumes:
      - mariadb_data:/var/lib/mysql
    networks:
      - iot_net

  # Dịch vụ Backend API
  flask_api:
    build: ./backend
    container_name: flask_service
    restart: always
    ports:
      - "5000:5000"
    networks:
      - iot_net
    depends_on:
      - mariadb

# Khai báo tài nguyên dùng chung cấp Root
volumes:
  mariadb_data:

networks:
  iot_net:
    driver: bridge
```
3. ƯU ĐIỂM KHI TRIỂN KHAI ỨNG DỤNG SỬ DỤNG DOCKER
Tính nhất quán môi trường ("Build Once, Run Anywhere"): Triệt tiêu hoàn toàn lỗi kinh điển: "Code chạy ngon lành trên máy tôi nhưng lỗi trên máy chủ". Docker đóng gói toàn bộ mã nguồn, thư viện phụ thuộc và biến môi trường vào một Image duy nhất, đảm bảo ứng dụng chạy đồng nhất trên mọi máy tính.

Tiết kiệm tài nguyên phần cứng tối đa: Do cơ chế chia sẻ chung nhân hệ điều hành (Kernel) và loại bỏ được tầng Guest OS nặng nề, một máy chủ vật lý sử dụng Docker có thể chạy số lượng container gấp nhiều lần so với số lượng máy ảo VM.

Tốc độ triển khai cực nhanh: Việc khởi tạo, tắt, bật hay nâng cấp phiên bản một hệ thống phức tạp bao gồm nhiều dịch vụ chỉ diễn ra trong vài giây thông qua các lệnh đơn giản (docker compose up / down / restart), tăng tốc quy trình CI/CD.

Cô lập và bảo mật an toàn: Mỗi container hoạt động độc lập trong môi trường Sandbox của riêng mình. Nếu một container ứng dụng (ví dụ Nginx hay Flask) bị tấn công hoặc dính mã độc, hacker cũng bị khóa chặt bên trong container đó và không thể tự do xâm nhập sang hệ điều hành máy chủ thật.

Dễ dàng mở rộng (Scalability): Dễ dàng nhân bản (scale-up) một container dịch vụ lên thành nhiều thực thể để phân tải (Load Balancing) khi lưu lượng người dùng tăng đột biến.

4. QUY TRÌNH TRIỂN KHAI ĐIỀU KIỆN OFFLINE (MÁY CHỦ KHÔNG CÓ INTERNET)
Đây là bài toán thực tế cực kỳ phổ biến tại các doanh nghiệp lớn, ngân hàng hoặc cơ quan nhà nước nhằm bảo mật thông tin tối đa. Khi máy chủ thật hoàn toàn cách ly với Internet, bạn không thể dùng các lệnh tải trực tiếp (docker pull, apt-get install, npm install). Quy trình xử lý gồm 4 bước:

Bước 1: Đóng gói toàn bộ Docker Images tại máy Laptop cá nhân (Có Internet)
Trên máy laptop của bạn – nơi hệ thống ứng dụng đã được tạo và kiểm thử chạy thành công (OK), tiến hành quét và xuất tất cả các Image cấu thành nên ứng dụng ra thành các tệp nén vật lý (.tar hoặc .tar.gz).

```Bash
# Xuất các image hệ thống ra file vật lý
docker save -o mariadb_image.tar mariadb:10.6
docker save -o influxdb_image.tar influxdb:1.8-alpine
docker save -o nodered_image.tar nodered/node-red:latest
docker save -o grafana_image.tar grafana/grafana:latest
docker save -o nginx_image.tar nginx:alpine

# Đối với image tự build (Flask API), bạn commit hoặc save trực tiếp tên image local
docker save -o flask_image.tar bt5_complete-flask_api:latest
```
Bước 2: Chuẩn bị bộ cài đặt Docker Engine Offline cho Máy chủ thật
Vì máy chủ thật mới tinh chưa có Docker và không có mạng, bạn phải lên trang chủ của Docker tải sẵn các gói cài đặt Offline định dạng tương thích với hệ điều hành của máy chủ thật (ví dụ tệp .deb cho Ubuntu/Debian hoặc .rpm cho CentOS/RHEL) và lưu vào thiết bị lưu trữ di động.

Bước 3: Sao chép toàn bộ dữ liệu sang máy chủ vật lý
Sử dụng các thiết bị ngoại vi an toàn (USB, ổ cứng di động) hoặc kết nối dây mạng LAN nội bộ trực tiếp để sao chép các thành phần sau sang máy chủ thật:

Các tệp nén hình ảnh: *.tar (Đã tạo ở Bước 1).

Các gói cài đặt Docker Engine Offline (Đã chuẩn bị ở Bước 2).

Toàn bộ thư mục mã nguồn chứa cấu hình: gồm tệp docker-compose.yml, thư mục frontend/, backend/,...

Bước 4: Khởi chạy và kích hoạt hệ thống tại máy chủ thật
Đăng nhập vào Terminal của máy chủ thật và thực hiện chuỗi lệnh quản trị:

Cài đặt Docker Engine (Chế độ Offline):

```Bash
# Ví dụ trên Ubuntu: Cài đặt tất cả các gói .deb trong thư mục USB
sudo dpkg -i /path_to_usb_folder/*.deb
```
Nạp (Load) các tệp ảnh vào Docker Engine của máy chủ:

```Bash
# Nạp các file ảnh mà không tốn một byte băng thông internet nào
docker load -i mariadb_image.tar
docker load -i influxdb_image.tar
docker load -i nodered_image.tar
docker load -i grafana_image.tar
docker load -i nginx_image.tar
docker load -i flask_image.tar
```
(Gõ docker images để kiểm tra, bạn sẽ thấy toàn bộ danh sách image xuất hiện sẵn sàng).

Kích hoạt hệ thống: Di chuyển vào thư mục chứa dự án (nơi đặt tệp docker-compose.yml) và ra lệnh khởi chạy:

```Bash
cd ~/bt5_complete/
docker compose up -d
```
# 5. Thực hành áp dụng:
Bước 1: Tạo dự án và các cấu trúc tệp dữ liệu
<img width="702" height="66" alt="image" src="https://github.com/user-attachments/assets/bb0d79fa-5dd5-42ec-9ddb-4fab6f5df222" />
Tiến hành khởi tạo nội dung cho các tệp cấu hình tầng Backend Flask API:
1. backend/requirements.txt
   <img width="1103" height="616" alt="image" src="https://github.com/user-attachments/assets/cb9a940d-1410-4af4-8d3e-0cc143f6a75f" />
2. backend/Dockerfile:
   <img width="1112" height="631" alt="image" src="https://github.com/user-attachments/assets/f6f96ab4-bc61-4946-86af-df93b2823ad8" />
3. backend/app.py:
   <img width="1108" height="623" alt="image" src="https://github.com/user-attachments/assets/6b90c20c-98a6-4d00-906e-c64929edf7e4" />
Bước 2: Cấu hình các dịch vụ trong docker-compose.yml
<img width="1112" height="612" alt="image" src="https://github.com/user-attachments/assets/f473d83c-9df7-4736-ab4b-69dfab59cc0e" />
Bước 3: Khởi chạy cụm Docker Container
<img width="1097" height="547" alt="image" src="https://github.com/user-attachments/assets/055b9ddb-7720-4336-a205-d2506843eec6" />
Bước 4: Cấu hình Node-RED & Xử lý API Thời tiết:
<img width="1180" height="412" alt="image" src="https://github.com/user-attachments/assets/38dd73fe-0851-4c73-a80b-6cb6c8099a5d" />
    Bước 5: Cấu hình grafana để liên kết tới influx:
  <img width="1111" height="459" alt="image" src="https://github.com/user-attachments/assets/cf9d7256-051c-4c04-805e-249de7bc678f" />
<img width="962" height="544" alt="image" src="https://github.com/user-attachments/assets/a403b664-3823-446b-a7a5-383434f261d3" />

Bước 6: Cấu hình frontend kết nối tới flask api và iframe của grafana:
<img width="1107" height="622" alt="image" src="https://github.com/user-attachments/assets/c1b21bd4-6036-4cb0-9106-3540fbad37aa" />
<img width="1107" height="622" alt="image" src="https://github.com/user-attachments/assets/6f5ebbb0-72be-41de-b4f4-e3c00d9eade4" />
- Kết quả web: <img width="1353" height="989" alt="image" src="https://github.com/user-attachments/assets/ed018522-edcd-4fb9-8b21-38ce6bd2cef0" />
 Bước 7: Kết nối tới bot tele và gửi cảnh báo vào nhóm:
<img width="1111" height="814" alt="image" src="https://github.com/user-attachments/assets/d7b326ed-3175-4c76-93d9-00b92f7451fd" />
<img width="1432" height="937" alt="image" src="https://github.com/user-attachments/assets/8c7f2159-0635-4922-b1cf-3a88dd892a48" />

- Bước 8: Xóa và cài lại các dịch vụ:
  <img width="1096" height="131" alt="image" src="https://github.com/user-attachments/assets/ad1f53d0-a04c-473c-ac82-5e4caa1f1d2c" />
<img width="1920" height="661" alt="image" src="https://github.com/user-attachments/assets/882dc29b-1214-42ca-8bb9-3a52ab22e7dc" />
