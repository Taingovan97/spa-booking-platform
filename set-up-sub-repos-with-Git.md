Câu hỏi này **rất đúng “chỗ đau” của Git submodule** 👍
Mình trả lời ngắn gọn trước, rồi đưa **workflow chuẩn nên dùng cho team**.

---

## Cách ĐÚNG để pull đầy đủ source khi dùng submodule

### ✅ Cách 1 (khuyến nghị – dùng mọi lúc)

```bash
git clone --recurse-submodules <REPO_CHA_URL>
```

👉 Kết quả:

* Clone repo cha
* Tự động clone **toàn bộ submodule**
* Checkout **đúng commit** mà repo cha đang pin

---

### ✅ Cách 2 (đã clone repo cha rồi)

```bash
git submodule update --init --recursive
```

👉 Dùng khi:

* Clone repo cha **quên `--recurse-submodules`**
* Pull code mới mà có submodule mới

---

## Khi repo cha update submodule (bump commit)

Sau này khi bạn pull code:

```bash
git pull
git submodule update --recursive
```

Hoặc gộp 1 lệnh:

```bash
git pull --recurse-submodules
```

---

## Workflow chuẩn cho team (rất nên ghi vào README)

### 🚀 Lần đầu setup project

```bash
git clone --recurse-submodules git@github.com:org/spa-platform.git
cd spa-platform
```

### 🔄 Mỗi lần pull code mới

```bash
git pull --recurse-submodules
```

Hoặc an toàn tuyệt đối:

```bash
git pull
git submodule update --init --recursive
```

---

## Nếu submodule trỏ theo branch (develop/main)

Nếu trong `.gitmodules` có:

```ini
[submodule "spa-booking-service"]
  branch = develop
```

Thì có thể dùng:

```bash
git submodule update --remote
git submodule update --remote --recursive
```

⚠️ **Lưu ý quan trọng**
Lệnh này:

* Checkout **latest commit của branch**
* Có thể **KHÁC** commit repo cha đang pin
* → Chỉ nên dùng khi **chủ động update version**

---

## Cảnh báo rất quan trọng (tránh lỗi “thiếu code”)

❌ Chỉ chạy:

```bash
git clone <repo-cha>
```

→ Kết quả:

```
spa-booking-service/
  (empty directory or detached HEAD)
```

Dev mới rất hay dính lỗi này 😅

---

## Checklist ngắn gọn cho README repo cha

````md
## Setup

```bash
git clone --recurse-submodules <repo>
````

## Pull latest

```bash
git pull --recurse-submodules
```

## If submodules are missing

```bash
git submodule update --init --recursive
```

```

---

## Tóm tắt 1 dòng (chuẩn “lead dev”)

> **Repo cha KHÔNG tự pull code submodule — dev bắt buộc dùng `--recurse-submodules` hoặc `git submodule update`.**

---

Nếu bạn muốn, mình có thể:
- Viết **README chuẩn cho repo cha** (microservices + submodule)
- Đề xuất **CI/CD strategy** khi dùng submodule (lock version → deploy)
- Tạo **script bootstrap** (`setup.sh` / `setup.ps1`) để dev mới chạy 1 lệnh là xong

Chỉ cần nói 👍
```
