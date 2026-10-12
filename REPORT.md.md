# BÁO CÁO KỸ THUẬT: PHÂN TÍCH KIẾN TRÚC CONTAINER VS VIRTUAL MACHINE
**Dự án:** Chuyển đổi hạ tầng từ VPS truyền thống sang Docker Container  
**Người thực hiện:** Kỹ sư Hệ thống / DevOps  

---

## 1. Bối cảnh & Lý do chuyển đổi hạ tầng
Mô hình triển khai trên Virtual Private Server (VPS/VM) tạo ra gánh nặng tài nguyên đáng kể do mỗi máy ảo phải chạy một hệ điều hành khách (Guest OS) độc lập cùng lớp ảo hóa Hypervisor. Nhằm tối ưu hóa chi phí phần cứng, đẩy nhanh chu kỳ phát hành sản phẩm (CI/CD) và hỗ trợ tự động mở rộng (auto-scaling) tức thì khi có biến động tải, doanh nghiệp quyết định chuyển dịch các dịch vụ vi mô (microservices) từ VPS sang nền tảng Docker Container.

---

## 2. Bảng so sánh chuyên sâu: Virtual Machine vs Container

| Tiêu chí | Virtual Machine (VPS / Hypervisor) | Container (Docker Engine) |
| :--- | :--- | :--- |
| **Dung lượng Image** | Lớn (vài GB đến hàng chục GB do chứa trọn vẹn Guest OS, kernel và thư viện). | Nhỏ (vài chục MB đến vài trăm MB; dùng base image tối giản như Alpine chỉ từ ~5 MB). |
| **Thời gian khởi động** | Chậm (tính bằng phút do phải nạp BIOS ảo, khởi động Kernel và các tiến trình init/systemd). | Nhanh (tính bằng mili-giây đến vài giây; bản chất chỉ là tạo tiến trình mới trên Host Kernel). |
| **Hao phí OS (OS Overhead)** | Cao (tiêu tốn RAM, CPU cố định cho Guest OS và chi phí chuyển đổi ngữ cảnh qua Hypervisor). | Gần như bằng 0 (chạy trực tiếp trên Host Kernel, chia sẻ tài nguyên động và không cần ảo hóa phần cứng). |
| **Mức độ cô lập (Isolation)** | Tuyệt đối (cô lập ở mức phần cứng ảo; lỗi Kernel ở một VM không ảnh hưởng đến Host và VM khác). | Tương đối (cô lập mức tiến trình qua kernel primitives; chia sẻ chung Host Kernel nên tiềm ẩn rủi ro nếu kernel bị khai thác). |
| **Khả năng mở rộng (Scalability)** | Kém linh hoạt (tạo mới và cấu hình mất nhiều thời gian, mật độ đóng gói instance/host thấp). | Rất cao (dễ dàng khởi tạo hàng trăm container trong vài giây, mật độ đóng gói tài nguyên tối đa). |

---

## 3. Cơ chế cốt lõi của Linux Kernel trong Containerization

Docker thực chất không ảo hóa phần cứng mà dựa vào hai tính năng cốt lõi của Linux Kernel để tạo môi trường thực thi cô lập:

### 3.1. Linux Namespaces (Cô lập không gian nhìn thấy)
Namespaces quy định những gì một tiến trình có thể **nhìn thấy** trên hệ thống, phân tách tài nguyên hệ thống giữa các container:
* **PID Namespace (Process ID):** Cô lập cây tiến trình. Bên trong container, tiến trình chính nhận PID = 1 và không thể thấy hoặc gửi tín hiệu (signal) tới các tiến trình của container khác hay Host OS.
* **NET Namespace (Network):** Cung cấp stack mạng độc lập gồm giao diện mạng ảo (`veth`), bảng định tuyến (routing table), bảng ARP, danh sách cổng mạng (ports) và quy tắc tường lửa (iptables) riêng biệt.
* **MNT Namespace (Mount Points):** Cô lập hệ thống tệp tin (filesystem). Container có gốc thư mục (`/`) riêng, tách biệt hoàn toàn với cây thư mục của máy chủ, ngăn chặn việc truy cập trái phép dữ liệu ngoài phạm vi được chỉ định.

### 3.2. Control Groups - cgroups (Giới hạn và giám sát tài nguyên)
Trong khi Namespaces quy định những gì container thấy, cgroups kiểm soát lượng tài nguyên mà container có thể **sử dụng**:
* **Giới hạn RAM (Memory):** Thiết lập trần sử dụng bộ nhớ (hard limit / soft limit). Khi container chạm ngưỡng RAM cho phép, cơ chế OOM (Out Of Memory) Killer sẽ can thiệp để bảo vệ tài nguyên của toàn bộ máy chủ và các dịch vụ lân cận.
* **Giới hạn CPU:** Kiểm soát thời gian chiếm dụng CPU thông qua cơ chế Shares (độ ưu tiên tương đối) hoặc Quota/Period (giới hạn cứng số core hoặc phần trăm xung nhịp mà container được phép tính toán trong một chu kỳ).

---

