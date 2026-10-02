# Bài 3 - Cấu hình xác thực SSH và đẩy dự án lên GitHub

Môn: DevOps - Session 04
Học viên: Lê Toàn (GitHub: `letoan0902`)
Môi trường thực hiện: Windows 11, Git Bash, `git 2.49.0.windows.1`, `OpenSSH_9.9p2`

## 1. Mục tiêu

- Khởi tạo cặp khóa SSH bảo mật bằng thuật toán Ed25519 để xác thực kết nối từ xa.
- Cấu hình liên kết an toàn giữa repository cục bộ và GitHub.
- Đẩy mã nguồn cùng lịch sử commit lên GitHub bằng giao thức SSH (không dùng HTTPS).

Ràng buộc của đề bài: bắt buộc dùng SSH và thuật toán Ed25519.

Toàn bộ lệnh và output được lưu theo thứ tự trong file `terminal_log.txt` cùng thư mục.

## 2. Kiến thức nền

### 2.1. So sánh HTTPS và SSH khi làm việc với GitHub

| Tiêu chí | HTTPS | SSH |
| --- | --- | --- |
| Dạng URL | `https://github.com/user/repo.git` | `git@github.com:user/repo.git` |
| Cách xác thực | Tên đăng nhập và Personal Access Token (GitHub không còn chấp nhận mật khẩu tài khoản) | Cặp khóa bất đối xứng: khóa riêng ở máy, khóa công khai đăng ký trên GitHub |
| Nhập thông tin mỗi lần push | Phải nhập hoặc phụ thuộc credential helper | Không phải nhập (nếu dùng ssh-agent hoặc khóa không passphrase) |
| Bí mật có bị gửi qua mạng không | Token được gửi trong mỗi request | Khóa riêng không bao giờ rời khỏi máy, chỉ gửi chữ ký |
| Cổng mặc định | 443 | 22 |
| Rủi ro khi lộ | Token lộ là bị chiếm quyền theo phạm vi token | Lộ khóa riêng mới bị chiếm quyền; khóa công khai lộ không gây hại |
| Phù hợp | Môi trường chặn cổng 22, CI đơn giản | Lập trình viên làm việc hằng ngày, tự động hóa |

Trong bài này dùng SSH vì an toàn hơn (không truyền bí mật), thuận tiện (không hỏi mật khẩu) và là yêu cầu bắt buộc của đề bài.

### 2.2. Vì sao chọn Ed25519 thay vì RSA

| Tiêu chí | RSA | Ed25519 |
| --- | --- | --- |
| Cơ sở toán học | Phân tích thừa số nguyên tố | Đường cong Edwards (Curve25519) |
| Độ dài khóa để đạt mức an toàn ~128 bit | 3072 bit trở lên | 256 bit |
| Kích thước khóa công khai | Dài (khoảng 750 ký tự với RSA 4096) | Ngắn (68 ký tự phần mã hóa base64) |
| Tốc độ sinh khóa và ký | Chậm hơn | Nhanh hơn |
| Chống tấn công kênh kề, lỗi sinh số ngẫu nhiên khi ký | Phụ thuộc cài đặt | Thiết kế tất định, ít rủi ro hơn |
| Được GitHub hỗ trợ | Có | Có và được khuyến nghị |

Kết luận: Ed25519 cho khóa ngắn, nhanh, an toàn và là lựa chọn hiện đại được GitHub khuyến nghị.

## 3. Thực hiện

### Bước 1. Sinh cặp khóa Ed25519

Trước khi tạo, kiểm tra tên khóa chưa tồn tại để không ghi đè các khóa đang có (`id_ed25519`, `id_rsa`, ...). Khóa mới được đặt tên riêng là `id_ed25519_github`.

```bash
ssh-keygen -t ed25519 -C "trinhhanh261293@gmail.com" -f ~/.ssh/id_ed25519_github
```

Output:

```text
Generating public/private ed25519 key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /c/Users/gigabyte/.ssh/id_ed25519_github
Your public key has been saved in /c/Users/gigabyte/.ssh/id_ed25519_github.pub
The key fingerprint is:
SHA256:Tq1tFkaXhQUQ0JCEZ8oUpYEhizOaXlJz9DGf7oJTifA trinhhanh261293@gmail.com
The key's randomart image is:
+--[ED25519 256]--+
|  . .+oB==oo.+o  |
| . oo +oB o .o   |
|+ .+ +.= o. o    |
|.+. = + oo .     |
|o. . E oS.+      |
|. o   oo.+ .     |
| .   o .o.+      |
|      . .o       |
|                 |
+----[SHA256]-----+
```

Hai dòng `Enter passphrase` và `Enter same passphrase again` là bước hỏi cụm mật khẩu bảo vệ khóa riêng. Trong thực tế nên đặt passphrase; khi chạy để thực hành, thao tác này chỉ cần nhấn Enter hai lần.

Giải thích từng tham số:

| Tham số | Ý nghĩa |
| --- | --- |
| `ssh-keygen` | Công cụ của OpenSSH dùng để sinh, quản lý và chuyển đổi khóa |
| `-t ed25519` | Chọn thuật toán (type) Ed25519 theo yêu cầu đề bài |
| `-C "trinhhanh261293@gmail.com"` | Comment gắn vào cuối khóa công khai để nhận diện chủ sở hữu (thường là email) |
| `-f ~/.ssh/id_ed25519_github` | Đường dẫn và tên file khóa riêng; khóa công khai tự động thêm đuôi `.pub` |
| `-N ""` (tùy chọn) | Đặt passphrase rỗng không tương tác; bỏ tham số này để được hỏi passphrase |

Vị trí và quyền của hai file được tạo:

```bash
ls -la ~/.ssh/id_ed25519_github*
```

```text
-rw-r--r-- 1 gigabyte 197610 419 Oct  2 12:47 /c/Users/gigabyte/.ssh/id_ed25519_github
-rw-r--r-- 1 gigabyte 197610 107 Oct  2 12:47 /c/Users/gigabyte/.ssh/id_ed25519_github.pub
```

| File | Vị trí | Vai trò | Có được chia sẻ không |
| --- | --- | --- | --- |
| `id_ed25519_github` | `~/.ssh/` (Windows: `C:\Users\gigabyte\.ssh\`) | Khóa riêng tư (private key) | Tuyệt đối không |
| `id_ed25519_github.pub` | `~/.ssh/` | Khóa công khai (public key) | Có, đăng ký lên GitHub |

Về quyền file: trên Linux/macOS khóa riêng phải có quyền `600` (chỉ chủ sở hữu đọc/ghi), thư mục `~/.ssh` là `700`; nếu quyền quá rộng OpenSSH sẽ từ chối dùng khóa với cảnh báo `UNPROTECTED ... KEY FILE`. Trên Windows, Git Bash hiển thị quyền mô phỏng `-rw-r--r--`, việc bảo vệ thực tế dựa trên ACL của NTFS (thư mục profile của người dùng chỉ cho tài khoản đó truy cập).

### Bước 2. Xem khóa công khai và fingerprint

```bash
cat ~/.ssh/id_ed25519_github.pub
```

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIFAtfVq1rQWm1vWQQf8TSfNnzN3yW5A5hJ3nFN05zXDN trinhhanh261293@gmail.com
```

Cấu trúc một dòng khóa công khai: `<loại khóa> <dữ liệu khóa mã hóa base64> <comment>`.

```bash
ssh-keygen -lf ~/.ssh/id_ed25519_github.pub
```

```text
256 SHA256:Tq1tFkaXhQUQ0JCEZ8oUpYEhizOaXlJz9DGf7oJTifA trinhhanh261293@gmail.com (ED25519)
```

Fingerprint `SHA256:Tq1tFkaX...` là giá trị băm ngắn gọn đại diện cho khóa; GitHub hiển thị đúng chuỗi này trong trang SSH keys nên dùng để đối chiếu. Số `256` là độ dài khóa (bit), `(ED25519)` là loại khóa.

Bản sao khóa công khai (chỉ khóa công khai) được nộp kèm tại `id_ed25519_github.pub`.

### Bước 3. Thêm khóa công khai vào tài khoản GitHub

Cách 1: qua giao diện web (cách đã dùng)

1. Sao chép toàn bộ nội dung một dòng của file `.pub` (ví dụ `cat ~/.ssh/id_ed25519_github.pub | clip` trên Git Bash Windows, hoặc bôi đen rồi copy).
2. Đăng nhập GitHub, nhấn ảnh đại diện góc trên bên phải, chọn Settings.
3. Ở thanh bên trái, mục Access chọn SSH and GPG keys.
4. Nhấn nút New SSH key.
5. Điền:
   - Title: `Laptop Windows 11 - DevOps Session 4` (tên gợi nhớ máy đang dùng).
   - Key type: `Authentication Key` (dùng để xác thực; loại `Signing Key` chỉ dùng để ký commit).
   - Key: dán nội dung file `.pub`, bắt đầu bằng `ssh-ed25519 AAAA...`.
6. Nhấn Add SSH key, xác nhận mật khẩu hoặc 2FA nếu GitHub yêu cầu.
7. Kiểm tra dòng khóa mới xuất hiện với fingerprint `SHA256:Tq1tFkaXhQUQ0JCEZ8oUpYEhizOaXlJz9DGf7oJTifA` khớp với Bước 2.

Cách 2: qua GitHub CLI (phương án thay thế)

```bash
gh auth login
gh ssh-key add ~/.ssh/id_ed25519_github.pub --title "Laptop Windows 11 - DevOps Session 4"
gh ssh-key list
```

Lệnh `gh ssh-key add` cần token có quyền `admin:public_key`; nếu thiếu quyền, chạy `gh auth refresh -h github.com -s admin:public_key`.

### Bước 4. Cấu hình `~/.ssh/config` và ssh-agent

Tạo file `~/.ssh/config` để SSH luôn dùng đúng khóa khi kết nối tới GitHub:

```text
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_github
    IdentitiesOnly yes
```

| Dòng | Ý nghĩa |
| --- | --- |
| `Host github.com` | Áp dụng khối cấu hình khi kết nối tới `github.com` |
| `HostName github.com` | Tên máy chủ thật để kết nối |
| `User git` | GitHub luôn dùng tài khoản hệ thống `git`; danh tính thật được xác định bằng khóa |
| `IdentityFile` | Khóa riêng dùng để xác thực |
| `IdentitiesOnly yes` | Chỉ thử khóa nêu trên, không thử lần lượt mọi khóa khác trong `~/.ssh` hoặc trong agent |

Máy này có nhiều khóa (`id_ed25519`, `id_rsa`, `id_ed25519_ptit_s02`, ...), nên `IdentitiesOnly yes` giúp tránh việc server từ chối vì thử quá nhiều khóa (`Too many authentication failures`). Bản mẫu của khối này nằm ở `ssh_config.sample`.

Nạp khóa vào ssh-agent để không phải nhập lại passphrase:

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519_github
ssh-add -l
```

```text
Agent pid 240
Identity added: /c/Users/gigabyte/.ssh/id_ed25519_github (trinhhanh261293@gmail.com)
256 SHA256:Tq1tFkaXhQUQ0JCEZ8oUpYEhizOaXlJz9DGf7oJTifA trinhhanh261293@gmail.com (ED25519)
```

`ssh-agent` là tiến trình nền giữ khóa riêng đã giải mã trong bộ nhớ; `ssh-add` nạp khóa vào agent; `ssh-add -l` liệt kê fingerprint các khóa đang nạp.

### Bước 5. Kiểm tra kết nối SSH tới GitHub

```bash
ssh -T git@github.com
```

Lần kết nối đầu tiên, SSH chưa biết khóa máy chủ của GitHub nên hỏi xác nhận:

```text
The authenticity of host 'github.com (20.205.243.166)' can't be established.
ED25519 key fingerprint is SHA256:+DiY3wvvV6TuJJhbpZisF/zLDA0zPMSvHdkr4UvCOqU.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'github.com' (ED25519) to the list of known hosts.
Hi letoan0902! You've successfully authenticated, but GitHub does not provide shell access.
```

Quy trình đúng: so sánh fingerprint hiển thị với fingerprint ED25519 chính thức do GitHub công bố, đó là `SHA256:+DiY3wvvV6TuJJhbpZisF/zLDA0zPMSvHdkr4UvCOqU`. Khớp thì gõ `yes`; khi đó host key được lưu vào `~/.ssh/known_hosts` và các lần sau không hỏi lại. Nếu fingerprint khác, phải dừng lại vì có thể đang bị tấn công kẻ đứng giữa (man-in-the-middle).

Kết quả: dòng `Hi letoan0902! You've successfully authenticated, but GitHub does not provide shell access.` xác nhận GitHub đã nhận diện đúng tài khoản `letoan0902` qua khóa SSH. Phần "does not provide shell access" là bình thường, GitHub không cấp shell, chỉ phục vụ giao thức Git.

Chạy lại lần hai (không còn hỏi xác nhận host key):

```text
Hi letoan0902! You've successfully authenticated, but GitHub does not provide shell access.
```

### Bước 6. Tạo repository trên GitHub và liên kết remote

Tạo repository rỗng trên GitHub:

1. Nhấn dấu `+` góc trên bên phải, chọn New repository.
2. Repository name: `DevOps-Session4-bai3`.
3. Chọn Public (hoặc Private tùy yêu cầu nộp bài).
4. Không tick `Add a README file`, `.gitignore`, `license` để kho hoàn toàn rỗng, tránh xung đột lịch sử khi push lần đầu.
5. Nhấn Create repository, chọn tab SSH để lấy URL `git@github.com:letoan0902/DevOps-Session4-bai3.git`.

Tạo dự án cục bộ có sẵn lịch sử commit:

```bash
mkdir DevOps-Session4-bai3 && cd DevOps-Session4-bai3
git init -b main
git config user.name "Lê Toàn"
git config user.email "trinhhanh261293@gmail.com"
git config core.autocrlf false
```

```text
Initialized empty Git repository in D:/Rikkei/DevOps/Session4/_demo/bai3/DevOps-Session4-bai3/.git/
```

Nội dung hai file của dự án:

```text
README.md : tiêu đề và mô tả dự án demo
hello.sh  : script in lời chào
```

Tạo hai commit:

```bash
git add README.md
git commit -m "docs: thêm README mô tả dự án"
git add hello.sh
git commit -m "feat: thêm script hello.sh"
git log --oneline
```

```text
7941fd4 feat: thêm script hello.sh
ee0261a docs: thêm README mô tả dự án
```

Liên kết remote bằng URL SSH:

```bash
git remote add origin git@github.com:letoan0902/DevOps-Session4-bai3.git
git remote -v
```

```text
origin	git@github.com:letoan0902/DevOps-Session4-bai3.git (fetch)
origin	git@github.com:letoan0902/DevOps-Session4-bai3.git (push)
```

URL origin có dạng `git@github.com:username/repository.git`, đúng yêu cầu kiểm tra của đề bài. Có thể xem thêm cấu hình remote mà không cần kết nối mạng:

```bash
git remote show -n origin
```

```text
* remote origin
  Fetch URL: git@github.com:letoan0902/DevOps-Session4-bai3.git
  Push  URL: git@github.com:letoan0902/DevOps-Session4-bai3.git
  HEAD branch: (not queried)
  Local ref configured for 'git push' (status not queried):
    (matching) pushes to (matching)
```

Đổi remote từ HTTPS sang SSH (khi trước đó đã clone hoặc add bằng HTTPS):

```bash
git remote -v
# origin  https://github.com/letoan0902/DevOps-Session4-bai3.git (fetch)

git remote set-url origin git@github.com:letoan0902/DevOps-Session4-bai3.git
git remote -v
# origin  git@github.com:letoan0902/DevOps-Session4-bai3.git (fetch)
# origin  git@github.com:letoan0902/DevOps-Session4-bai3.git (push)
```

`git remote set-url` chỉ đổi URL, không mất lịch sử hay cấu hình nhánh.

### Bước 7. Đẩy mã nguồn lên GitHub bằng SSH

```bash
git push -u origin main
```

```text
Enumerating objects: 6, done.
Counting objects: 100% (6/6), done.
Delta compression using up to 8 threads
Compressing objects: 100% (4/4), done.
Writing objects: 100% (6/6), 649 bytes | 649.00 KiB/s, done.
Total 6 (delta 0), reused 0 (delta 0), pack-reused 0 (from 0)
To github.com:letoan0902/DevOps-Session4-bai3.git
 * [new branch]      main -> main
branch 'main' set up to track 'origin/main'.
```

Ý nghĩa: `-u` (`--set-upstream`) liên kết nhánh cục bộ `main` với `origin/main` nên các lần sau chỉ cần gõ `git push` hoặc `git pull`. Dòng `To github.com:letoan0902/...` cho thấy kết nối đi qua SSH.

Kiểm tra nhánh theo dõi và trạng thái đồng bộ:

```bash
git branch -vv
git status -sb
git ls-remote origin
```

```text
* main 7941fd4 [origin/main] feat: thêm script hello.sh

## main...origin/main

7941fd4c70319d315fb3a6fdc1743949c9928678	HEAD
7941fd4c70319d315fb3a6fdc1743949c9928678	refs/heads/main
```

Hash `7941fd4...` của remote trùng với commit mới nhất ở máy cục bộ, chứng tỏ mã nguồn và lịch sử commit đã lên GitHub đầy đủ. Mở URL web để xem hai commit trên giao diện GitHub.

## 4. URL kho lưu trữ

| Loại | URL |
| --- | --- |
| URL SSH (origin) | `git@github.com:letoan0902/DevOps-Session4-bai3.git` |
| URL web | https://github.com/letoan0902/DevOps-Session4-bai3 |

## 5. Sơ đồ luồng xác thực bằng khóa công khai

```text
   May cua hoc vien                                    May chu GitHub
 +--------------------+                            +----------------------+
 | Khoa rieng         |                            | Khoa cong khai da    |
 | id_ed25519_github  |                            | dang ky tren tai     |
 | (khong roi may)    |                            | khoan letoan0902     |
 +---------+----------+                            +-----------+----------+
           |                                                   |
           | 1. ssh/git ket noi toi github.com:22 (user git)   |
           |-------------------------------------------------->|
           |                                                   |
           | 2. Server gui host key; client doi chieu voi      |
           |    known_hosts (lan dau: hoi yes/no)              |
           |<--------------------------------------------------|
           |                                                   |
           | 3. Client de xuat khoa cong khai (fingerprint)    |
           |-------------------------------------------------->|
           |                                                   |
           |        4. Server tim khoa trong tai khoan,        |
           |           gui thu thach (du lieu ngau nhien)      |
           |<--------------------------------------------------|
           |                                                   |
           | 5. Client ky thu thach bang khoa rieng            |
           |    (ssh-agent ky ho neu da nap khoa)              |
           |-------------------------------------------------->|
           |                                                   |
           |        6. Server kiem tra chu ky bang khoa        |
           |           cong khai -> hop le                     |
           |<--------------------------------------------------|
           |   "Hi letoan0902! You've successfully             |
           |    authenticated..."                              |
           |                                                   |
           | 7. Kenh ma hoa mo ra -> git push / git pull       |
           |<=================================================>|
```

Điểm mấu chốt: khóa riêng không bao giờ được gửi đi, chỉ có chữ ký của thử thách được gửi; vì vậy lộ khóa công khai không ảnh hưởng đến an toàn.

## 6. Xử lý sự cố thường gặp

| Triệu chứng | Nguyên nhân | Cách xử lý |
| --- | --- | --- |
| `git@github.com: Permission denied (publickey).` | GitHub không có khóa công khai tương ứng; SSH dùng nhầm khóa; chưa nạp khóa vào agent | Kiểm tra khóa đã thêm ở Settings > SSH and GPG keys; chạy `ssh -vT git@github.com` xem khóa nào được thử; kiểm tra `IdentityFile` trong `~/.ssh/config`; chạy `ssh-add ~/.ssh/id_ed25519_github`; so fingerprint bằng `ssh-keygen -lf` |
| `WARNING: UNPROTECTED ... KEY FILE! Permissions ... are too open` | Quyền file khóa riêng quá rộng | Linux/macOS: `chmod 600 ~/.ssh/id_ed25519_github` và `chmod 700 ~/.ssh`; Windows: chỉ cho tài khoản của mình quyền đọc file (Properties > Security hoặc `icacls`) |
| Push vẫn hỏi username/password hoặc báo lỗi xác thực HTTPS | Remote `origin` đang là URL HTTPS | `git remote -v` để kiểm tra, rồi `git remote set-url origin git@github.com:letoan0902/DevOps-Session4-bai3.git` |
| `Host key verification failed.` | Host key của github.com thay đổi so với `known_hosts`, hoặc file `known_hosts` hỏng | Đối chiếu fingerprint với công bố chính thức của GitHub; nếu đúng thì xóa dòng cũ bằng `ssh-keygen -R github.com` rồi kết nối lại và gõ `yes`; nếu sai thì không kết nối |
| `Too many authentication failures` | Agent chứa nhiều khóa, server ngắt sau vài lần thử sai | Thêm `IdentitiesOnly yes` vào khối `Host github.com` |
| `Could not open a connection to your authentication agent` | Chưa chạy ssh-agent trong phiên shell hiện tại | `eval "$(ssh-agent -s)"` rồi `ssh-add` lại |
| `ERROR: Repository not found.` | Sai tên user hoặc repo, hoặc chưa tạo repo | Kiểm tra lại URL trong `git remote -v` và repo đã tồn tại trên GitHub |
| `! [rejected] main -> main (fetch first)` | Repo GitHub không rỗng (đã tạo kèm README) | `git pull --rebase origin main` rồi push lại, hoặc tạo repo rỗng |
| Cổng 22 bị chặn bởi mạng | Tường lửa công ty hoặc trường học | Dùng SSH qua cổng 443: `Host github.com` với `HostName ssh.github.com` và `Port 443` |

## 7. Lưu ý bảo mật

- Không bao giờ nộp, đẩy lên Git, gửi qua chat hay chụp màn hình khóa riêng (`id_ed25519_github`). Thư mục nộp bài chỉ chứa file `.pub`. File `.gitignore` ở thư mục gốc chặn `id_*` (trừ `*.pub`), `*.pem`, `*.ppk` để tránh vô tình commit khóa riêng.
- Nên đặt passphrase cho khóa riêng và dùng ssh-agent (hoặc Windows OpenSSH Agent, Keychain) để không phải gõ lại nhiều lần. Passphrase bảo vệ khóa khi máy bị mất hoặc bị sao chép dữ liệu.
- Mỗi máy dùng một khóa riêng, đặt Title khác nhau trên GitHub để dễ nhận biết và thu hồi từng máy.
- Khi nghi ngờ lộ khóa riêng: vào Settings > SSH and GPG keys, nhấn Delete ở khóa đó, sau đó sinh khóa mới và đăng ký lại. Khóa cũ trên GitHub bị xóa thì lập tức vô hiệu.
- Định kỳ rà soát danh sách khóa trên GitHub, xóa các khóa của máy không còn sử dụng.
- Luôn đối chiếu fingerprint host key của GitHub ở lần kết nối đầu tiên, không gõ `yes` một cách máy móc.

## 8. Đối chiếu yêu cầu đề bài

| Yêu cầu | Cách đáp ứng | Vị trí minh chứng |
| --- | --- | --- |
| Sinh khóa SSH bằng thuật toán Ed25519 | `ssh-keygen -t ed25519 -C "trinhhanh261293@gmail.com" -f ~/.ssh/id_ed25519_github` | Bước 1, mục 3 |
| Không dùng HTTPS, bắt buộc SSH | Remote origin dùng URL `git@github.com:...`, push qua SSH | Bước 6 và Bước 7 |
| Cấu hình liên kết an toàn giữa repo cục bộ và GitHub | Thêm khóa công khai vào GitHub, cấu hình `~/.ssh/config`, `git remote add origin` | Bước 3, 4, 6 |
| `ssh -T git@github.com` xác nhận danh tính | Thông báo `Hi letoan0902! You've successfully authenticated, but GitHub does not provide shell access.` | Bước 5 |
| `git remote -v` cho URL dạng `git@github.com:username/repository.git` | `origin git@github.com:letoan0902/DevOps-Session4-bai3.git` | Bước 6 |
| Đẩy mã nguồn và lịch sử commit lên GitHub | `git push -u origin main`, hai commit `ee0261a` và `7941fd4` | Bước 7 |
| README mô tả quá trình tạo khóa, liên kết remote | Tài liệu này | Mục 3 |
| Cung cấp URL kho GitHub | `git@github.com:letoan0902/DevOps-Session4-bai3.git` và `https://github.com/letoan0902/DevOps-Session4-bai3` | Mục 4 |
| Không nộp khóa private | Chỉ nộp `id_ed25519_github.pub`; `.gitignore` chặn khóa riêng | Mục 7 |

## 9. Danh sách file nộp

| File | Nội dung |
| --- | --- |
| `README.md` | Báo cáo này |
| `id_ed25519_github.pub` | Khóa công khai Ed25519 (không phải khóa riêng) |
| `ssh_config.sample` | Khối cấu hình `Host github.com` mẫu cho `~/.ssh/config` |
| `terminal_log.txt` | Toàn bộ lệnh và output theo thứ tự thực hiện |

## 10. Kết luận

Qua bài thực hành đã hoàn thành chuỗi công việc: sinh cặp khóa Ed25519 `id_ed25519_github` (fingerprint `SHA256:Tq1tFkaXhQUQ0JCEZ8oUpYEhizOaXlJz9DGf7oJTifA`), đăng ký khóa công khai lên tài khoản `letoan0902`, cấu hình `~/.ssh/config` cùng ssh-agent, xác nhận danh tính bằng `ssh -T git@github.com`, liên kết remote `origin` dạng SSH và đẩy thành công nhánh `main` cùng hai commit lên kho `DevOps-Session4-bai3`.

Kinh nghiệm rút ra: SSH với Ed25519 vừa an toàn (khóa riêng không rời máy) vừa tiện lợi (không phải nhập mật khẩu mỗi lần push); `IdentitiesOnly yes` rất hữu ích khi máy có nhiều khóa; kiểm tra bằng `ssh -T` và `git remote -v` trước khi push giúp phát hiện sớm lỗi xác thực và nhầm remote HTTPS.

URL kho lưu trữ: https://github.com/letoan0902/DevOps-Session4-bai3
