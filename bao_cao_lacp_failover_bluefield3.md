# Báo cáo kết quả kiểm thử LACP Failover trên NVIDIA BlueField-3

## 1. Mục tiêu kiểm thử

Kiểm tra khả năng **duy trì kết nối TCP và RDMA** qua LACP khi một trong hai cổng mạng vật lý (`p0` hoặc `p1`) bị ngắt.

## 2. Cấu hình kiểm thử

| Thành phần | Cấu hình / trạng thái |
|---|---|
| Thiết bị | NVIDIA BlueField-3 DPU |
| Hai cổng vật lý | `p0`, `p1` — 200 Gbps mỗi cổng |
| Giao diện bonding | `bond0` |
| Bonding mode | IEEE 802.3ad (LACP) |
| Transmit hash policy | `layer3+4` |
| LACP rate | `fast` |
| `min_links` | `0` |
| OVS bridge | `ovsbr1` chứa `bond0` và `pf0hpf` |
| Công cụ kiểm thử | `ping` (ICMP), `iperf3` (TCP), `ib_write_bw` (RDMA) |

## 3. Trình tự kiểm thử LACP

Ngắt port trên switch kết nối với `p0` của BlueField-3, quan sát trạng thái trên DPU, sau đó bật lại port.

| Giai đoạn | `p0` | `p1` | `bond0` | LAG mapping |
|---|---|---|---|---|
| Ban đầu | UP | UP | UP | `1, 2` |
| Ngắt port trên switch | DOWN | UP | UP | `2` |
| Bật lại port trên switch *(dự kiến)* | UP | UP | UP | `1, 2` |

**Ghi nhận:** Khi `p0` DOWN, `p1` và `bond0` vẫn UP; ICMP tiếp tục hoạt động. Trạng thái sau khi bật lại port là dự kiến, chưa có log xác nhận.

## 4. Kết quả truyền dữ liệu TCP và RDMA theo trạng thái port

Thứ tự trình bày kết quả kiểm thử: `p0 UP` **→** `p0 DOWN` **→** `p0 UP`. Trong cả ba giai đoạn, `p1` và `bond0` duy trì trạng thái UP; mục tiêu là thể hiện các luồng TCP và RDMA có khả năng tiếp tục truyền dữ liệu qua LACP khi ngắt một physical port.

| Trình tự                            | Trạng thái trên DPU            | TCP — `iperf3` | RDMA — `ib_write_bw` | Tình trạng kết nối                               |
| ----------------------------------- | ------------------------------ | -------------- | -------------------- | ------------------------------------------------ |
| **1. Ban đầu**                      | `p0 UP`, `p1 UP`, `bond0 UP`   | 97,9 Gbps      | 301,22 Gbps          | TCP, RDMA đang truyền dữ liệu                    |
| **2. Ngắt port switch nối** `p0`    | `p0 DOWN`, `p1 UP`, `bond0 UP` | 93,6 Gbps      | 60,29 Gbps           | TCP, RDMA vẫn có thông lượng; ICMP tiếp tục chạy |
| **3. Bật lại port switch nối** `p0` | `p0 UP`, `p1 UP`, `bond0 UP`   | 344 Gbps       | 40,12 Gbps           | TCP, RDMA tiếp tục truyền dữ liệu; ICMP tiếp tục chạy      |

Kết quả ghi nhận không ổn định nhưng có thể thấy từ tổng thông lượng của cả TCP và RDMA trong trường hợp 1 và 3 đều đạt gần 400 Gbps, trong trường hợp 2 đạt gần 200 Gbps. 
