# Đề 8 – Streaming telemetry: gNMIc + Prometheus + Grafana

Project giữa kỳ môn Mạng thế hệ mới. Hệ thống thu thập telemetry từ hai router Nokia SR Linux qua gNMI, lưu vào Prometheus, hiển thị trên Grafana và cảnh báo khi băng thông vượt ngưỡng.

## Kiến trúc

```
[srl1] ──e1-1── [srl2]              router SR Linux, gNMI cổng 57400
   │
  e1-2
   │
 [h1]                               client netshoot, dùng tạo traffic

[srl1], [srl2] ──gNMI STREAM──▶ [gnmic] ──:9273/metrics──▶ [Prometheus] ──▶ [Grafana]
```

## Cấu trúc thư mục

```
de08-telemetry/
├── containerlab/telemetry.clab.yml        # topology: 2 router SR Linux + 1 client
├── gnmic/gnmic.yaml                       # subscription gNMI và output Prometheus
├── prometheus/prometheus.yml              # scrape gnmic:9273
├── grafana/
│   ├── provisioning/datasources/          # datasource Prometheus tự động
│   └── dashboards/telemetry-lab.json      # dashboard đã export
├── docker-compose.yml                     # gnmic, prometheus, grafana
├── evidence/                              # output lệnh và log minh chứng
└── README.md
```

## Yêu cầu

- Ubuntu 22.04 hoặc 24.04 (đã thử trên Ubuntu 26.04.1 LTS chạy bằng WSL2 trên Windows 10).
- RAM tối thiểu 8 GB, còn trống khoảng 20 GB ổ cứng.
- Docker Engine và Docker Compose plugin.
- containerlab 0.83.0 trở lên.

Trên Windows 10, cài WSL2 và Ubuntu trước (`wsl --install -d Ubuntu`), sau đó làm mọi bước bên dưới trong cửa sổ Ubuntu.

## Cài đặt môi trường

```bash
# Docker Engine
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
sudo systemctl enable --now docker

# containerlab
bash -c "$(curl -sL https://get.containerlab.dev)"
sudo usermod -aG clab_admins $USER
```

Đóng và mở lại cửa sổ Ubuntu để nhóm quyền có hiệu lực. Kiểm tra bằng `groups` (phải có `docker` và `clab_admins`).

## Chạy hệ thống

**Bước 1: Dựng lab**

```bash
cd ~/de08-telemetry
docker pull ghcr.io/nokia/srlinux:latest
sudo containerlab deploy -t containerlab/telemetry.clab.yml
sudo containerlab inspect -t containerlab/telemetry.clab.yml
```

Ghi lại IP của `srl1` và `srl2` trong bảng inspect. Lần deploy này IP là `srl1` 172.20.20.3 và `srl2` 172.20.20.5. IP do containerlab cấp tự động và có thể thay đổi sau mỗi lần deploy lại.

**Bước 2: Cấu hình cổng nối với client**

Gán IP cho client `h1`:

```bash
docker exec clab-telemetry-h1 ip addr add 10.1.1.2/24 dev eth1
docker exec clab-telemetry-h1 ip link set eth1 up
```

Cấu hình cổng `e1-2` trên `srl1` (vào CLI bằng `docker exec -it clab-telemetry-srl1 sr_cli`, gõ từng dòng, rồi `quit` để thoát):

```
enter candidate
set / interface ethernet-1/2 admin-state enable
set / interface ethernet-1/2 subinterface 0 type routed
set / interface ethernet-1/2 subinterface 0 admin-state enable
set / interface ethernet-1/2 subinterface 0 ipv4 admin-state enable
set / interface ethernet-1/2 subinterface 0 ipv4 address 10.1.1.1/24
set / network-instance default interface ethernet-1/2.0
commit now
```

Kiểm tra kết nối: `docker exec clab-telemetry-h1 ping -c 3 10.1.1.1`.

**Bước 3: Kiểm tra gNMI**

```bash
docker run --rm --network clab ghcr.io/openconfig/gnmic:latest \
  -a 172.20.20.3:57400 -u admin -p 'NokiaSrl1!' --skip-verify -e json_ietf \
  get --path /system/information/version
```

Kết quả là JSON chứa phiên bản SR Linux là đạt. Nếu IP đã đổi, thay IP tương ứng.

**Bước 4: Khởi động stack giám sát**

```bash
docker compose up -d
sleep 15
curl -s localhost:9273/metrics | grep octets | head -3
```

Phải có dòng chứa `in_octets` và `source="172.20.20.3:57400"`.

**Bước 5: Truy cập giao diện**
- Prometheus: http://localhost:9090 (xem `/targets`, target `gnmic` phải UP).
- Grafana: http://localhost:3000, đăng nhập `admin` / `admin123`.

Dashboard `Telemetry Lab` có thể nạp bằng cách import file `grafana/dashboards/telemetry-lab.json` trong Grafana (**Dashboards → New → Import → Upload JSON file**).

## Tạo traffic và kiểm tra cảnh báo

```bash
docker exec clab-telemetry-h1 ping -s 1400 -i 0.01 10.1.1.1
```

Lệnh này gửi khoảng 100 gói mỗi giây, mỗi gói 1400 byte, tương đương khoảng 1.1 Mb/s. Dừng bằng Ctrl+C.

**Lưu ý:** Ctrl+C trong `docker exec` có thể không dừng tiến trình ping bên trong container. Nếu băng thông vẫn tăng sau khi dừng, chạy:

```bash
docker exec clab-telemetry-h1 pkill ping
```

Rule cảnh báo `Băng thông vào vượt ngưỡng` (thư mục `Telemetry`) sẽ chuyển sang **Firing** khi băng thông vượt 500 kb/s, và về **Normal** khoảng một phút sau khi traffic dừng do cửa sổ `rate(...[1m])`.

## Kết quả thực nghiệm

| Trạng thái | Băng thông vào ethernet-1/2 (srl1) | CPU router |
|---|---|---|
| Nền | khoảng 60 b/s | khoảng 3–4 % |
| Đang ping 1400 byte | khoảng 1.15 Mb/s (Firing) | khoảng 20 % |
| Sau khi dừng ping | 0 b/s (Normal) | trở về mức nền |

Ảnh minh chứng và output lệnh nằm trong thư mục `evidence/`.

## Dọn dẹp

```bash
cd ~/de08-telemetry
docker compose down
sudo containerlab destroy -t containerlab/telemetry.clab.yml --cleanup
```

Thêm `-v` vào `docker compose down` nếu muốn xóa cả dữ liệu Prometheus và Grafana.

## Xử lý lỗi thường gặp

| Triệu chứng | Cách xử lý |
|---|---|
| `lookup clab-telemetry-srl1 ... no such host` | Dùng IP thay vì tên container trong `gnmic.yaml` và lệnh `docker run`. |
| gnmic báo `Unimplemented ... encoding` | Thêm `encoding: json_ietf` vào `gnmic.yaml`. |
| Target `gnmic` DOWN trên Prometheus | Kiểm tra IP trong `targets` của `gnmic.yaml`, rồi `docker compose restart gnmic`. |
| `/metrics` không có CPU | Kiểm tra đường dẫn `/platform/control[slot=A]/cpu[index=all]/total` trong `gnmic.yaml`. |
| `Permission denied` khi `git add` | Thư mục `containerlab/clab-telemetry/` được tạo bằng quyền root, đã được loại trừ trong `.gitignore`. |
| Băng thông không về 0 sau khi dừng ping | Chạy `docker exec clab-telemetry-h1 pkill ping`. |
| Lỗi quyền docker | Chạy `groups` kiểm tra có `docker`, nếu không có thì đóng và mở lại Ubuntu. |

## Tài liệu tham khảo

- gNMI specification (OpenConfig)
- Tài liệu gNMIc: gnmic.openconfig.net
- Tài liệu containerlab: containerlab.dev
- Tài liệu Nokia SR Linux (YANG và gNMI)
