# Hướng dẫn thiết lập môi trường Pentest với Docker trên macOS

Hướng dẫn này cung cấp các bước để xóa toàn bộ container, image, và mạng Docker hiện có trên macOS, sau đó cài đặt lại môi trường pentest với Kali Linux và Metasploitable2 trong mạng `loint-pentest`. Ngoài ra, hướng dẫn cũng bao gồm cách cài đặt Wireshark trên macOS để phân tích lưu lượng mạng, và tùy chọn sử dụng Edgeshark và packetflix-handler để đơn giản hóa việc bắt gói tin.

## Yêu cầu
- **macOS**: Đã cài đặt Docker Desktop (phiên bản mới nhất).
- **Homebrew**: Được cài đặt để cài Wireshark và Docker Compose.
- **Quyền admin**: Một số lệnh yêu cầu quyền `sudo`.
- **Kết nối mạng**: Đảm bảo macOS có kết nối Internet để tải image và công cụ.

## 1. Xóa toàn bộ container, image và mạng Docker

Để bắt đầu lại từ đầu và tránh lỗi phát sinh, bạn cần xóa sạch các tài nguyên Docker hiện có.

### 1.1. Dừng và xóa tất cả container
1. **Liệt kê các container**:
   ```bash
   docker ps -a
   ```

2. **Dừng tất cả container đang chạy**:
   ```bash
   docker stop $(docker ps -q)
   ```

3. **Xóa tất cả container**:
   ```bash
   docker rm -f $(docker ps -a -q)
   ```

### 1.2. Xóa tất cả image
1. **Liệt kê các image**:
   ```bash
   docker images -a
   ```

2. **Xóa tất cả image**:
   ```bash
   docker rmi -f $(docker images -q)
   ```

   **Lưu ý**: Nếu chỉ muốn xóa image cụ thể:
   ```bash
   docker rmi kalilinux/kali-rolling
   docker rmi tleemcjr/metasploitable2
   ```

### 1.3. Xóa mạng Docker
1. **Liệt kê các mạng**:
   ```bash
   docker network ls
   ```

2. **Xóa mạng `loint-pentest`**:
   ```bash
   docker network rm loint-pentest
   ```

3. **Xóa tất cả mạng không sử dụng** (tùy chọn):
   ```bash
   docker network prune
   ```

### 1.4. Xóa volume và dữ liệu Docker (tùy chọn)
Để xóa hoàn toàn dữ liệu Docker (bao gồm volume và cache):
```bash
docker system prune -a --volumes
```

**Cảnh báo**: Lệnh này xóa toàn bộ dữ liệu Docker, bao gồm volume. Chỉ sử dụng nếu muốn làm sạch hoàn toàn.

### 1.5. Kiểm tra lại
- Xác nhận không còn container:
  ```bash
  docker ps -a
  ```
- Xác nhận không còn image:
  ```bash
  docker images -a
  ```
- Xác nhận không còn mạng `loint-pentest`:
  ```bash
  docker network ls
  ```

## 2. Cài đặt lại môi trường pentest

### 2.1. Tạo mạng Docker
Tạo mạng `loint-pentest`:
```bash
docker network create --driver bridge loint-pentest
```

### 2.2. Tải và chạy container Kali
1. **Tải image Kali Linux**:
   ```bash
   docker pull kalilinux/kali-rolling
   ```

2. **Chạy container Kali**:
   ```bash
   docker run -it --name kali --network loint-pentest --privileged kalilinux/kali-rolling /bin/bash
   ```

   - `--privileged`: Cấp quyền cao để chạy các công cụ như `tcpdump`.
   - `--network loint-pentest`: Kết nối container với mạng tùy chỉnh.

3. **Cài công cụ trong Kali** (tùy chọn):
   Trong container Kali:
   ```bash
   apt update && apt install tcpdump nmap metasploit-framework -y
   ```

### 2.3. Tải và chạy container Metasploitable2
1. **Tải image Metasploitable2**:
   ```bash
   docker pull tleemcjr/metasploitable2
   ```

2. **Chạy container Metasploitable2**:
   ```bash
   docker run -it --name metasploitable2 --network loint-pentest tleemcjr/metasploitable2 /bin/bash
   ```

3. **Kiểm tra dịch vụ**:
   Trong container Metasploitable2:
   ```bash
   service --status-all
   ```

### 2.4. Kiểm tra kết nối mạng
1. Trong container Kali, kiểm tra IP của Metasploitable2:
   ```bash
   docker network inspect loint-pentest
   ```
   Ghi lại IP của Metasploitable2 (ví dụ: `172.18.0.2`).

2. Thử ping từ Kali:
   ```bash
   ping 172.18.0.2
   ```

## 3. Cài đặt Wireshark trên macOS

### 3.1. Cài Wireshark
Cài Wireshark bằng Homebrew:
```bash
brew install wireshark
```
Hoặc tải từ [https://www.wireshark.org/download.html](https://www.wireshark.org/download.html) và cài đặt.

### 3.2. Cấp quyền cho Wireshark
```bash
sudo chmod -R o+rw /dev/bpf*
```

### 3.3. Bắt gói tin bằng tcpdump trong Kali
Vì Docker Desktop trên macOS chạy trong VM, sử dụng `tcpdump` trong container Kali là cách hiệu quả để bắt gói tin:

1. Trong container Kali:
   ```bash
   tcpdump -i eth0 -w capture.pcap
   ```

2. Tạo lưu lượng mạng (ví dụ):
   ```bash
   nmap 172.18.0.2
   ```

3. Sao chép tệp `.pcap` sang macOS:
   ```bash
   docker cp kali:/capture.pcap ~/capture.pcap
   ```

4. Mở tệp trong Wireshark:
   ```bash
   open -a Wireshark ~/capture.pcap
   ```

## 4. Cài đặt Edgeshark và packetflix-handler (tùy chọn)

Edgeshark giúp đơn giản hóa việc bắt gói tin từ container thông qua giao diện web. Packetflix-handler là một AppleScript để xử lý URL `packetflix://` trên macOS, tích hợp Edgeshark với Wireshark.

### 4.1. Cài Docker Compose
```bash
brew install docker-compose
```

### 4.2. Chạy Edgeshark
```bash
wget -q --no-cache -O - https://github.com/siemens/edgeshark/raw/main/deployments/wget/docker-compose.yaml | docker compose -f - up
```

Truy cập giao diện web tại `http://localhost:5001`.

### 4.3. Cài plugin cshargextcap
1. Tải plugin từ [https://github.com/siemens/cshargextcap/releases](https://github.com/siemens/cshargextcap/releases) (chọn phiên bản cho macOS arm64/amd64).
2. Giải nén và sao chép:
   ```bash
   tar -xzf cshargextcap-<version>-darwin-<arch>.tar.gz
   sudo mv cshargextcap /Applications/Wireshark.app/Contents/MacOS/extcap/
   ```

### 4.4. Cài packetflix-handler
1. Tải và cài đặt:
   ```bash
   mkdir -p /tmp/pflix-handler && cd /tmp/pflix-handler
   curl -sLO https://github.com/srl-labs/containerlab/files/14278951/packetflix-handler.zip
   unzip packetflix-handler.zip
   sudo mv packetflix-handler.app /Applications
   sudo xattr -r -d com.apple.quarantine /Applications/packetflix-handler.app
   ```

2. **Kiểm tra quyền**:
   - Mở **System Preferences > Security & Privacy > Privacy > Automation**, thêm `packetflix-handler.app` để điều khiển Wireshark.
   - Nếu macOS yêu cầu cho phép trong **Security & Privacy > General**, nhấp “Allow” sau khi chạy Edgeshark.

### 4.5. Sử dụng Edgeshark
- Truy cập `http://localhost:5001`, chọn giao diện `eth0` của container Kali hoặc Metasploitable2, nhấp vào nút “fin” để bắt gói tin trong Wireshark.

## 5. Phân tích gói tin trong Wireshark
- Mở tệp `.pcap` hoặc bắt gói tin trực tiếp từ Edgeshark.
- Sử dụng bộ lọc:
  - `ip.addr == 172.18.0.2`: Lọc lưu lượng từ Metasploitable2.
  - `tcp.port == 21`: Lọc lưu lượng FTP.
  - `icmp`: Lọc gói tin ICMP.

## 6. Lưu ý
- **Phiên bản Wireshark**: Sử dụng Wireshark 4.4.1 hoặc mới hơn để tránh lỗi với plugin `cshargextcap`.
- **Bảo mật**: Không kết nối mạng `loint-pentest` với Internet để tránh rủi ro từ Metasploitable2.
- **Tài nguyên**: Đảm bảo Docker Desktop có đủ RAM và CPU (Preferences > Resources).
- **Sao lưu**: Lưu tệp `.pcap` để phân tích sau.

## 7. Xử lý sự cố
- **Packetflix-handler không hoạt động**: Kiểm tra quyền trong **System Preferences > Security & Privacy > Privacy > Automation**. Chạy lại `open /Applications/packetflix-handler.app`.
- **Wireshark không bắt được gói tin**: Sử dụng `tcpdump` trong Kali và chuyển tệp `.pcap` sang macOS.
- **Edgeshark không hiển thị container**: Kiểm tra log:
  ```bash
  docker logs <edgeshark-container-id>
  ```