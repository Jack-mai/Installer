# Triển khai Kubernetes HA bằng Ansible (Ubuntu 22.04)

## Mô hình

| Node        | IP           | Vai trò                                  |
|-------------|--------------|------------------------------------------|
| (VIP)       | 10.10.90.150 | API endpoint – keepalived, port 8443     |
| k8s-master1 | 10.10.90.141 | control-plane + etcd + HAProxy/keepalived|
| k8s-master2 | 10.10.90.142 | control-plane + etcd + HAProxy/keepalived|
| k8s-master3 | 10.10.90.143 | control-plane + etcd + HAProxy/keepalived|
| k8s-worker1 | 10.10.90.144 | worker                                   |
| k8s-worker2 | 10.10.90.145 | worker                                   |
| k8s-worker3 | 10.10.90.146 | worker                                   |

- Runtime: containerd (SystemdCgroup)
- Cài bằng kubeadm, repo chính thức `pkgs.k8s.io` (mặc định v1.36)
- CNI: Calico v3.32.2, Pod CIDR `10.244.0.0/16`, Service CIDR `10.96.0.0/12`
- etcd stacked (chạy trên 3 master), chịu được mất 1 master
- kubectl/HAProxy gọi API qua VIP `10.10.90.150:8443` → HAProxy → kube-apiserver `:6443` của 3 master

## Trước khi chạy

1. **VIP**: `10.10.90.150` phải là IP còn trống. Nếu dùng IP khác → sửa `api_vip` trong `group_vars/all.yml`.
2. Máy chạy Ansible SSH được vào cả 6 node bằng key, user có sudo
   (mặc định `ubuntu`, sửa trong `inventory/hosts.ini`).
3. Các node có Internet (tải gói từ pkgs.k8s.io, image từ registry.k8s.io, manifest Calico từ GitHub).
4. Mỗi node tối thiểu 2 CPU, 2GB RAM (khuyến nghị master 4GB), hostname/MAC/product_uuid khác nhau.
5. Nếu mạng ảo hóa (VMware/Proxmox/OpenStack) có chặn IP giả: cho phép VIP trên port của 3 master
   (OpenStack: `allowed-address-pairs`; VMware: bật Promiscuous/Forged transmits nếu cần).

## Chạy

```bash
# Trên máy điều khiển (Ubuntu)
sudo apt update && sudo apt install -y ansible
ansible-galaxy collection install -r requirements.yml

# Copy SSH key nếu chưa có
for i in 141 142 143 144 145 146; do ssh-copy-id ubuntu@10.10.90.$i; done

# Kiểm tra kết nối
ansible all -m ping

# Triển khai (~10-20 phút tùy mạng)
ansible-playbook site.yml
```

Nếu sudo cần mật khẩu: `ansible-playbook site.yml -K`

Playbook chạy lại nhiều lần được (idempotent): node nào đã init/join thì bỏ qua.

## Kiểm tra sau khi cài

```bash
ssh ubuntu@10.10.90.141
kubectl get nodes -o wide
kubectl get pods -A
kubectl -n kube-system get pods -l component=etcd
ip a | grep 10.10.90.150        # VIP đang nằm trên master nào
curl http://10.10.90.141:9000/stats   # trang thống kê HAProxy
```

Test HA: tắt master đang giữ VIP → VIP chuyển sang master khác trong vài giây, `kubectl` vẫn chạy.

## Thêm worker mới

Thêm dòng vào nhóm `[workers]` trong `inventory/hosts.ini`, rồi:

```bash
ansible-playbook site.yml --limit 'k8s-master1,k8s-worker4'
```

## Cài lại từ đầu

```bash
ansible-playbook reset.yml -e confirm_reset=yes
ansible-playbook site.yml
```

## Cấu trúc

```
ansible.cfg
inventory/hosts.ini          # danh sách node
group_vars/all.yml           # version, VIP, CIDR...
site.yml                     # playbook chính
reset.yml                    # xóa cluster
roles/
  common/                    # swap, kernel module, sysctl, containerd, kubeadm/kubelet/kubectl
  loadbalancer/              # HAProxy + keepalived (VIP) trên 3 master
  master_init/               # kubeadm init master1 + Calico + tạo lệnh join
  master_join/               # join master2, master3 (lần lượt)
  worker_join/               # join 3 worker
  kubeconfig/                # copy admin.conf cho root và user SSH
```

## Lỗi hay gặp

| Triệu chứng | Nguyên nhân / cách xử lý |
|---|---|
| `wait_for VIP` timeout | Sai card mạng → đặt `keepalived_interface` (vd `ens160`); hoặc VIP bị trùng/bị chặn |
| Join master báo lỗi certificate key | Key hết hạn sau 2 giờ → chạy lại `ansible-playbook site.yml` (tự tạo key mới) |
| Node `NotReady` lâu | `kubectl -n kube-system get pods -l k8s-app=calico-node`; kiểm tra node tải được image |
| Pod Calico lỗi BGP | Mở TCP 179 và IP-in-IP (protocol 4) giữa các node nếu có firewall ngoài |
