# PINNING OF MAPS
Trong eBPF, bản đồ (maps) là các cấu trúc dữ liệu dùng để lưu trữ và chia sẻ dữ liệu giữa kernel space và user space, hoặc giữa các chương trình eBPF khác nhau.

Bình thường, eBPF map sẽ chỉ tồn tại trong bộ nhớ khi còn chương trình eBPF đang sử dụng nó. Khi chương trình eBPF bị unload (gỡ bỏ), các map sẽ bị huỷ.

Để giải quyết vấn đề này, bạn dùng pinning – tức là ghim map vào hệ thống file ảo (bpffs) để nó tồn tại độc lập và có thể được mở bởi chương trình khác bất cứ lúc nào.

- Chương trình `xdp_loader.c`: Tải chương trình XDP và tạo map và pin map vào /sys/fs/bpf/.
- Chương trình `xdp_stats.c`: Mở map đã được pin và đọc dữ liệu thống kê từ đó.
# 1. Một số định nghĩa cần lưu ý
## 1.1 bpf syscall wrappers
Trong dự án XDP/eBPF:
- Một chương trình có thể chia thành 2 phần: phần load chương trình eBPF vào kernel, và phần đọc dữ liệu thống kê từ map.
- Phần đọc thống kê `(xdp_stats.c)` chỉ cần thực hiện các thao tác như: mở map đã được pin, đọc key/value — không cần tải chương trình eBPF mới.
- Vì thế, không cần dùng toàn bộ libbpf với các tính năng như xử lý object files, section parsing...Thay vào đó, chỉ cần dùng các hàm bọc syscall cơ bản trong bpf.h, ví dụ như:

```
int bpf_map_get_fd_by_id(__u32 id);
int bpf_map_lookup_elem(int fd, const void *key, void *value);
```
## 1.2 Mounting the BPF file system
Cơ chế được sử dụng để chia sẻ các map BPF giữa các chương trình được gọi là pinning (ghim). Điều này có nghĩa là chúng ta sẽ tạo một tệp cho mỗi map trong một hệ thống tập tin đặc biệt được gắn tại đường dẫn /sys/fs/bpf/.

Lệnh mount cần dùng là:
```
mount -t bpf bpf /sys/fs/bpf/
```
## 1.3 Những lưu ý/phạm lỗi thường gặp khi sử dụng map đã pin
Ghim tất cả các map trong một bpf_object bằng libbpf có thể được thực hiện dễ dàng với hàm `bpf_object__pin_maps(bpf_object, pin_dir)`, mà chúng ta sử dụng trong file `xdp_loader`.

Để tránh việc trùng tên file, chúng ta tạo một thư mục con theo tên interface mà chúng ta đang tải chương trình BPF lên.

Khi bạn load chương trình, bạn thường pin các map vào thư mục như /sys/fs/bpf/<iface>/ để tái sử dụng.

## 1.4 Xoá các map khi tải lại chương trình XDP
Khi tải lại chương trình BPF XDP thông qua công cụ `xdp_loader`, các map đã được ghim trước đó sẽ không được tái sử dụng.

Nhớ khởi động lại `xdp_stats` để nó theo dõi đúng map mới (đúng FD).
## 1.5 Tái sử dụng map với thư viện libbpf
Thư viện libbpf có thể tái sử dụng và thay thế một map bằng một file descriptor đã có sẵn, thông qua hàm API của libbpf: `bpf_map__reuse_fd()`.

Các bước tái sử dụng map:
- Mở file descriptor của map đã ghim (pinned) bằng `bpf_obj_get()`.
- Mở object file eBPF chứa chương trình mới bằng `bpf_object__open()`.
- Tìm map trong object file bằng tên với `bpf_object__find_map_by_name()`.
- Gắn lại file descriptor cũ vào map mới bằng `bpf_map__reuse_fd()`.
- Load chương trình với `bpf_object__load()`.

```
int pinned_map_fd = bpf_obj_get("/sys/fs/bpf/veth0/xdp_stats_map");
struct bpf_object *obj = bpf_object__open(cfg.filename);
struct bpf_map    *map = bpf_object__find_map_by_name(obj, "xdp_stats_map");
bpf_map__reuse_fd(map, pinned_map_fd);
bpf_object__load(obj);
```
# 2. Bài tập
## 2.1 Bài tập 1: (xdp_stats.c) reload map file-descriptor
- **Đề bài:** Chương trình `xdp_stats.c` hiện tại không phát hiện khi `xdp_loader `load lại chương trình và tạo map mới, nên phải restart thủ công.

- **Giải pháp:** Tự động reload file descriptor (FD) của map trong `xdp_stats.c` mà không cần restart chương trình.

- **Ý tưởng:**
    - Phát hiện khi map đã bị thay đổi (vì bị `xdp_loader` tạo mới).
    - Tự động reload lại FD mới tương ứng với map mới đó.

- **Các bước thực hiện:**
## 2.2 Assignment 2: (xdp_loader.c) reuse pinned map
