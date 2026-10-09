# DangLH-Levis

Giải pháp quản lý thời trang trên **Odoo 19**: website bán quần áo tham khảo giao diện và chức năng của [levi.com](https://www.levi.com), tích hợp bán hàng tại quầy (POS), thanh toán, quản lý kho, CRM và đào tạo nội bộ.

Đây là dự án mẫu để tìm hiểu quy trình nghiệp vụ trên Odoo. Trọng tâm: **Quản lý kho, POS, Tra cứu sản phẩm, Thanh toán**.

Repo này chứa **bản backup database** và hướng dẫn khôi phục. Toàn bộ cấu hình và dữ liệu nằm trong database, không có module tùy chỉnh.

## Chức năng chính

| Nhóm | Nội dung |
|---|---|
| Website / Shop | Trang giới thiệu, danh mục Áo / Quần / Phụ kiện, sản phẩm mẫu có biến thể size, màu, giá, ảnh, mã vạch |
| Thanh toán online | Giỏ hàng, checkout, payment provider (chế độ test), chuyển khoản dự phòng, đơn hàng trừ kho |
| POS | Quầy bán hàng gắn kho, mở/đóng phiên, bán hàng, hóa đơn, đổi/trả, thanh toán tại quầy |
| Tồn kho | Quản lý tồn kho theo sản phẩm, tra cứu từng sản phẩm |
| CRM | Form Đăng ký tư vấn và Contact Us tạo Lead, pipeline Mới → Tư vấn → Báo giá → Chốt |
| Khách hàng thân thiết | Chương trình tích điểm áp dụng cho POS |
| Nội dung | Blog/News, Events, About Us, Contact Us, menu kiểu levi.com |
| eLearning | Khóa học nội bộ mẫu "Quy trình kho và dùng POS" |

## Thông tin kỹ thuật

| Mục | Giá trị |
|---|---|
| Phiên bản | Odoo 19 |
| Cơ sở dữ liệu | PostgreSQL |
| Tên database | `lehaidang-db` |
| Chạy bằng | Docker |
| Tài khoản demo | `admin` / `admin` (chỉ dùng cho môi trường mẫu) |

## Nội dung repo

```
DangLH-Levis/
├── backup/        # Bản backup database (zip, gồm cả filestore)
└── README.md
```

Nếu file backup có dung lượng lớn, file được đính kèm ở mục **Releases** của repo thay vì nằm trong thư mục `backup/`.

## Yêu cầu

- Docker và Docker Compose
- Quyền truy cập repo (repo ở chế độ private)

## Bước 1: Dựng môi trường Odoo 19

Tạo một thư mục mới trên máy, tạo file `docker-compose.yml` với nội dung sau:

```yaml
services:
  db:
    image: postgres:16
    container_name: db
    environment:
      POSTGRES_DB: postgres
      POSTGRES_USER: odoo
      POSTGRES_PASSWORD: odoo
    volumes:
      - db-data:/var/lib/postgresql/data

  odoo:
    image: odoo:19.0
    container_name: odoo
    depends_on:
      - db
    ports:
      - "8069:8069"
    environment:
      HOST: db
      USER: odoo
      PASSWORD: odoo
    volumes:
      - odoo-data:/var/lib/odoo

volumes:
  db-data:
  odoo-data:
```

Khởi động:

```bash
docker compose up -d
```

Odoo chạy tại: http://localhost:8069

## Bước 2: Khôi phục database

Tải file backup (ví dụ `lehaidang-db_2026-10-09.zip`) từ repo này về máy.

### Cách A: Qua giao diện Odoo (khuyên dùng)

File zip đã gồm cả database và filestore (ảnh sản phẩm, tệp đính kèm).

1. Mở http://localhost:8069/web/database/manager
2. Chọn **Restore Database**
3. Nhập **Master Password** (mặc định là `admin` nếu chưa cấu hình `admin_passwd`)
4. Chọn file zip, đặt tên database là `lehaidang-db`
5. Nếu được hỏi, chọn **This database might have been moved**, rồi bấm **Continue**

### Cách B: Dòng lệnh (khi backup là file `.dump` và `filestore.tar.gz`)

```bash
# 1. Tạo database và nạp dữ liệu
docker exec -i db createdb -U odoo lehaidang-db
docker exec -i db pg_restore -U odoo -d lehaidang-db < lehaidang-db.dump

# 2. Giải nén và chép filestore vào container Odoo
tar -xzf filestore.tar.gz
docker cp filestore odoo:/var/lib/odoo/filestore/lehaidang-db
docker exec -u root odoo chown -R odoo:odoo /var/lib/odoo/filestore/lehaidang-db

# 3. Khởi động lại Odoo
docker restart odoo
```

> Tên database có dấu gạch ngang (`lehaidang-db`). Khi dùng trong câu lệnh SQL cần đặt trong dấu ngoặc kép: `"lehaidang-db"`.

## Bước 3: Đăng nhập và kiểm tra

Mở http://localhost:8069, chọn database `lehaidang-db`, đăng nhập bằng `admin` / `admin`.

- [ ] Đăng nhập được bằng tài khoản admin
- [ ] Website hiển thị đúng, trang Shop có sản phẩm kèm ảnh
- [ ] Tồn kho hiển thị số lượng từng sản phẩm
- [ ] Mở phiên POS, bán thử một đơn, tồn kho giảm tương ứng
- [ ] Đặt thử một đơn trên website, đơn trừ kho
- [ ] Form Đăng ký tư vấn tạo Lead trong CRM
- [ ] Khóa eLearning nội bộ xem được

## Tạo bản backup mới

**Cách nhanh** (file zip gồm cả filestore):

```bash
curl -X POST \
  -F "master_pwd=<admin_passwd>" \
  -F "name=lehaidang-db" \
  -F "backup_format=zip" \
  -o lehaidang-db_$(date +%F).zip \
  http://localhost:8069/web/database/backup
```

**Cách dòng lệnh:**

```bash
docker exec -t db pg_dump -U odoo -Fc lehaidang-db > lehaidang-db.dump
docker cp odoo:/var/lib/odoo/filestore/lehaidang-db ./filestore
tar -czf filestore.tar.gz filestore
```

File trên 25 MB không upload được qua giao diện web GitHub, hãy đính kèm vào **Releases** (tối đa 2 GB mỗi file).

## Lưu ý bảo mật

- Tài khoản `admin` / `admin` và Master Password mặc định chỉ phù hợp môi trường mẫu. Đổi trước khi triển khai thực tế.
- Không đưa mật khẩu thật, token hay file cấu hình chứa `admin_passwd` lên repo.
- Repo ở chế độ private; chỉ cấp quyền cho thành viên cần thiết.

## Liên hệ

Dự án thực hiện bởi DangLH, nhóm Phase1-team.
