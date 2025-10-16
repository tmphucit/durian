# Hướng Dẫn Build và Chạy Game Server JXSJ

## Giới Thiệu Game

**JXSJ (剑侠世界 - JX World - Kiếm Thế)**

- Game MMORPG 3D võ hiệp
- Engine 3D với đồ họa nâng cao
- Hệ thống môn phái đa dạng
- Chiến trường quy mô lớn (Castle War, Domain Battle)
- Kiến trúc client-server phức tạp

**Lưu ý:** Đây KHÔNG phải JX1 (Võ Lâm Truyền Kỳ 1 - 2D) mà là phiên bản nâng cao hơn với đồ họa 3D và gameplay phức tạp hơn.

## Tổng Quan Kiến Trúc

**Cấu trúc thư mục chính:**

- `/base/` - Thư viện cơ sở và header files (engine, network, common)
- `/devenv/` - Môi trường phát triển (Lua 5.0, MySQL, Oracle, zlib)
- `/gameworld/` - Mã nguồn chính (client, server, tools)
- `/JX/` - Thư mục runtime/deployment (chứa các file exe đã build và config)
- `/kgc/` - Các component bổ sung (text processing, XML, encryption)
- `/tool/` - Các công cụ hỗ trợ (biztool, devtool)

**Các server components:**

1. **Goddess** - Server quản lý account/role (port 5001)
2. **LogServer** - Server ghi log (port 5100)
3. **Bishop** - Gateway server kết nối payment system (port 5632)
4. **GameCenter** - Server quản lý trung tâm (port 5135)
5. **GameServer** - Server game world chính (port 6041)

## Phần 1: Yêu Cầu Hệ Thống và Cài Đặt

### 1.1 Windows Build Environment

**Yêu cầu phần mềm:**

- **Visual Studio 2005/2008/2010** (KHÔNG PHẢI VS6!) - Để build C++ code
- Visual Basic 6.0 - Cho UI tools và config tools (optional)
- MySQL Server 5.x hoặc 8.0 - Database cho game data
- DirectX 9 SDK - Có sẵn trong `/kgc/directx9/`

**LƯU Ý QUAN TRỌNG về Visual Studio:**

❌ **Visual Studio 6.0 KHÔNG thể mở file `.sln`**

- VS6 sử dụng file `.dsw` (workspace) và `.dsp` (project)
- File `.sln` và `.vcproj` là format của VS 2005+

✅ **Giải pháp:**

- Codebase này đã được upgrade lên **Visual Studio 2005 hoặc mới hơn**
- File `gameworld/gameworld.sln` cần VS 2005/2008/2010 để mở
- Tất cả file `.vcproj` cần VS 2005+ để build

**Cài đặt Visual Studio (2005/2008/2010):**

1. Download Visual Studio 2008 Express (miễn phí) hoặc Professional
2. Cài đặt với C++ support
3. Nếu có lỗi khi mở .sln, có thể cần upgrade solution:

   - Right-click .sln → "Convert" hoặc cho VS tự động convert

**Về password MySQL - QUAN TRỌNG:**

Password trong các file config được mã hóa bằng **thuật toán tự tạo** `SimplyEncryptPassword/SimplyDecryptPassword`:

- **Thuật toán**: Substitution cipher với random key (định nghĩa trong `/base/include/misc/passgen.h`)
- **Đặc điểm**: 
  - Mã hóa luôn tạo ra chuỗi 32 ký tự (chỉ chứa a-z, A-Z, 0-9, _)
  - Ví dụ: `_eHJKA_UKMEMeNvR__ZOCLQjYUQjWQPE` (32 chars)
  - Mỗi lần mã hóa cho kết quả khác nhau (random)
  - Giải mã luôn cho ra password gốc

**Cách xử lý password:**

1. **Option 1: Giải mã password hiện tại** (khó - cần build tool)

   - Code giải mã trong `base/include/misc/passgen.h` (function `SimplyDecryptPassword`)
   - Bạn cần build một tool nhỏ để giải mã

2. **Option 2: Thay password mới (KHUYẾN NGHỊ cho test):**

**Bước 1:** Comment out hoặc bypass encryption trong code:

   - File: `gameworld/server/gamecenter/gc_server/kgc_config.cpp` line 58
   - Thay: `!SimplyDecryptPassword(DBParam.szPassword, szPassword)`
   - Thành: `strcpy(DBParam.szPassword, szPassword) != NULL` (hoặc tương tự)

**Bước 2:** Đặt password plain text trong `gc_config.ini`:

   ```ini
   [DatabaseServer]
   Password=yourpassword
   ```

**Bước 3:** Làm tương tự cho:

   - `/gameworld/server/logserver/klogserver.cpp` line 619
   - `/gameworld/server/spreadersys/kg_spreader_cfg.cpp` line 50

3. **Option 3: Tạo password mã hóa mới**

   - Viết tool nhỏ gọi `SimplyEncryptPassword()` để tạo encrypted string từ plain password
   - Code mã hóa không có trong source (chỉ có giải mã)
   - Cần reverse engineer hoặc viết lại dựa theo thuật toán

**Lưu ý thêm:**

- Có một thuật toán mã hóa khác: `EDOneTimePad_Encipher/Decipher` (trong `/base/include/engine/edonetimepad.h`)
- Thuật toán này dùng cho GameMaster Central và Client (không dùng cho database config)

**Cài đặt MySQL:**

1. Tải và cài MySQL Server 5.x (hoặc 8.0 - xem lưu ý bên dưới)
2. Thiết lập root password
3. Tạo databases: `91jx_gamecenter`, `91jx_statlog`
4. Import SQL schemas từ `/JX/SQL/`

**Lưu ý về MySQL Version:**

- Code gốc được viết cho MySQL 5.x
- **MySQL 8.0 có thể dùng được** nhưng cần lưu ý:
  - Authentication method: MySQL 8.0 mặc định dùng `caching_sha2_password`, nhưng client cũ có thể chỉ hỗ trợ `mysql_native_password`
  - Nếu gặp lỗi authentication, cần đổi sang native password:
    ```sql
    ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'your_password';
    ```

  - SQL syntax có thể có khác biệt nhỏ, nhưng thường tương thích ngược

### 1.2 Linux Build Environment

**Yêu cầu phần mềm (CentOS/RedHat):**

```bash
# Cài đặt Development Tools
yum groupinstall "Development Tools"
yum install gcc gcc-c++ make

# Cài đặt dependencies
yum install mysql-devel
yum install lua-devel
yum install zlib-devel
yum install libuuid-devel

# Các thư viện 32-bit (nếu cần build 32-bit)
yum install glibc-devel.i686
yum install libstdc++-devel.i686
```

**Cài đặt MySQL:**

```bash
yum install mysql-server
service mysqld start
mysql_secure_installation
```

## Phần 2: Build Source Code

### 2.1 Build trên Windows

**Bước 1: Mở Solution File**

```
gameworld/gameworld.sln
```

**Bước 2: Build từng project theo thứ tự:**

1. **Build base libraries** (đã có sẵn compiled trong `/base/lib/`)

2. **Build jxex_lib:**

   - Mở `gameworld/jxex_lib/jxex_lib.vcproj`
   - Chọn configuration: Debug hoặc Release
   - Build → Build jxex_lib
   - Output: `gameworld/lib/debug/jxex_lib.lib`

3. **Build coregroup:**

   - Mở `gameworld/coregroup/coregroup.sln`
   - Build các projects: core_framework, core_logic, core_scene, core_api, core_extension
   - Output: `gameworld/lib/debug/core_*.lib`

4. **Build gc_core:**

   - Mở `gameworld/gc_core/gc_core.vcproj`
   - Build cả client và server versions
   - Output: `gameworld/lib/debug/gc_core_*.lib`

5. **Build servers:**

   - GameServer: `gameworld/server/gameserver/gameserver.vcproj`
   - GameCenter: `gameworld/server/gamecenter/gc_server/gc_server.vcproj`
   - KG_Angel: `gameworld/server/kg_angel/kg_angel.vcproj`
   - Bishop: `gameworld/server/bishop/bishop.vcproj`
   - Goddess: `gameworld/server/goddess/goddess.vcproj`
   - LogServer: `gameworld/server/logserver/logserver.vcproj`

**Bước 3: Integrate (Copy files to deployment folder)**

```cmd
cd gameworld\product
integrate.bat debug
```

hoặc

```cmd
integrate.bat release
```

Lệnh này sẽ:

- Tạo cấu trúc thư mục trong `gameworld/product/debug/` hoặc `release/`
- Copy tất cả DLLs, EXEs, và configs cần thiết
- Copy các thư viện phụ thuộc (engine.dll, lua5.dll, libmysql.dll, etc.)

### 2.2 Build trên Linux

**Bước 1: Generate Makefiles**

```bash
cd gameworld
chmod +x makeprj_build.sh
./makeprj_build.sh
```

Script này sử dụng tool `makeprj` để chuyển đổi các file `.vcproj` (Windows) thành makefile format cho Linux.

**Bước 2: Build từng component**

```bash
# Build XML library
cd ../kgc/xml
make

# Build encryption library
cd ../encryptlib
make

# Build jxex_lib
cd ../../gameworld/jxex_lib
make

# Build tất cả servers
cd ..
make debug
# hoặc
make release
```

**Bước 3: Integrate**

```bash
cd product
./integrate.sh debug
# hoặc
./integrate.sh release
```

**Output locations:**

- Debug: `gameworld/product/debug/server/`
- Release: `gameworld/product/release/server/`

## Phần 3: Cấu Hình Database

### 3.1 Tạo Databases

```sql
-- Kết nối MySQL
mysql -u root -p

-- Tạo databases
CREATE DATABASE 91jx_gamecenter CHARACTER SET utf8;
CREATE DATABASE 91jx_statlog CHARACTER SET utf8;

-- Import schemas
USE 91jx_gamecenter;
SOURCE /path/to/JXSJ/JX/SQL/jxaccount.sql;

-- Tạo user (optional, để bảo mật hơn)
CREATE USER 'jxadmin'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON 91jx_gamecenter.* TO 'jxadmin'@'localhost';
GRANT ALL PRIVILEGES ON 91jx_statlog.* TO 'jxadmin'@'localhost';
FLUSH PRIVILEGES;
```

### 3.2 Cấu Hình Connection Strings

**File: `/JX/gamecenter/gc_config.ini`**

```ini
[DatabaseServer]
Server=127.0.0.1
Port=3306
UserName=root
Password=YOUR_PASSWORD_HERE
Database=91jx_gamecenter

[StatLogDatabase]
Server=127.0.0.1
Port=3306
UserName=root
Password=YOUR_PASSWORD_HERE
Database=91jx_statlog
```

**File: `/JX/bishop/bishop.ini`**

```ini
[Paysys]
IPAddress=127.0.0.1
Port=8000
UserName=91jx
Password=1234
```

**File: `/JX/goddess/kg_goddess.ini`**

```ini
[DatabaseServer]
Server=127.0.0.1
Port=3306
UserName=root
Password=YOUR_PASSWORD_HERE
Database=91jx_gamecenter
```

## Phần 4: Cấu Hình Network và Ports

### 4.1 Cấu hình IP và Ports

**Mặc định (localhost development):**

- Goddess: 127.0.0.1:5001
- LogServer: 127.0.0.1:5100
- Bishop: 127.0.0.1:5632
- GameCenter: 127.0.0.1:5135
- GameServer Gateway: 127.0.0.1:5633
- GameServer World: 127.0.0.1:6041

**File: `/JX/gameserver/servercfg.ini`**

```ini
[Gateway]
Ip=127.0.0.1
Port=5633

[Gamecenter]
Ip=127.0.0.1
Port=5135

[GameServer]
ServerID=1
InIp=127.0.0.1
OutIp=127.0.0.1
Port=6041
```

### 4.2 Firewall Configuration (Production)

**Windows:**

```cmd
netsh advfirewall firewall add rule name="JX Goddess" dir=in action=allow protocol=TCP localport=5001
netsh advfirewall firewall add rule name="JX LogServer" dir=in action=allow protocol=TCP localport=5100
netsh advfirewall firewall add rule name="JX Bishop" dir=in action=allow protocol=TCP localport=5632
netsh advfirewall firewall add rule name="JX GameCenter" dir=in action=allow protocol=TCP localport=5135
netsh advfirewall firewall add rule name="JX Gateway" dir=in action=allow protocol=TCP localport=5633
netsh advfirewall firewall add rule name="JX GameServer" dir=in action=allow protocol=TCP localport=6041
```

**Linux:**

```bash
firewall-cmd --permanent --add-port=5001/tcp
firewall-cmd --permanent --add-port=5100/tcp
firewall-cmd --permanent --add-port=5632/tcp
firewall-cmd --permanent --add-port=5135/tcp
firewall-cmd --permanent --add-port=5633/tcp
firewall-cmd --permanent --add-port=6041/tcp
firewall-cmd --reload
```

## Phần 5: Chạy Game Server

### 5.1 Khởi động thủ công (Manual Start)

**Thứ tự khởi động quan trọng:**

**Windows:**

```cmd
cd /d C:\path\to\JXSJ\JX

# 1. Start Goddess (Account/Role server)
cd goddess
start KG_GoddessD.exe

# 2. Start LogServer
cd ..\logserver
start kg_cslogserverd.exe

# 3. Start Bishop (Gateway to payment)
cd ..\bishop
start KG_BishopD.exe

# 4. Start GameCenter
cd ..\gamecenter
start gamecenterd.exe gm

# 5. Start GameServer
cd ..\gameserver
start gameserverd.exe
```

**Linux:**

```bash
cd /path/to/JXSJ/JX

# 1. Start Goddess
cd goddess
./kg_goddessd_linux &

# 2. Start LogServer
cd ../logserver
./kg_cslogserverd_linux &

# 3. Start Bishop
cd ../bishop
./kg_bishopd_linux &

# 4. Start GameCenter
cd ../gamecenter
./gamecenterd_linux gm &

# 5. Start GameServer
cd ../gameserver
./gameserverd_linux &
```

### 5.2 Khởi động tự động (Automatic Start)

**Windows:**

```cmd
cd /d C:\path\to\JXSJ\JX
server_start.exe
```

**File config: `/JX/server_start.ini`**

```ini
[config]
GODDESS_ININame=kg_goddess.ini
LOGSERVER_ININame=kg_cslogserver.ini
BISHOP_ININame=bishop.ini
RELAY_ININame=gc_config.ini
GAMESERVER_ININame=servercfg.ini

[goddess]
IsChoice=1
PATH=.\goddess\kg_goddessd.exe
TIME=2

[LOGSERVER]
IsChoice=1
PATH=.\logserver\kg_cslogserverd.exe
TIME=1

[bishop]
IsChoice=1
PATH=.\bishop\KG_BishopD.exe
TIME=1

[gc_config GAME_CENTER]
IsChoice=1
PATH=.\gamecenter\gamecenterd.exe gm
TIME=0

[servercfg]
IsChoice=1
PATH=.\gameserver\gameserverd.exe
TIME=0
```

**Tham số:**

- `IsChoice=1` - Bật server này (0 = tắt)
- `TIME` - Delay (giây) trước khi start server tiếp theo

### 5.3 Kiểm tra Server đang chạy

**Windows:**

```cmd
netstat -ano | findstr "5001 5100 5632 5135 5633 6041"
tasklist | findstr "goddess bishop logserver gamecenter gameserver"
```

**Linux:**

```bash
netstat -tlnp | grep -E "5001|5100|5632|5135|5633|6041"
ps aux | grep -E "goddess|bishop|logserver|gamecenter|gameserver"
```

## Phần 6: Xem Logs và Troubleshooting

### 6.1 Log Locations

```
/JX/goddess/logs/KG_Goddess/
/JX/bishop/logs/KG_Bishop/
/JX/logserver/Logs/KG_CSLogServer/
/JX/gamecenter/log/gamecenter/
/JX/gameserver/log/gameserver/
/JX/ErrLog/
```

### 6.2 Common Issues

**Problem: Server không start**

- Kiểm tra port đã bị sử dụng chưa
- Kiểm tra MySQL đã chạy chưa
- Xem log files để tìm error messages

**Problem: Database connection failed**

- Xác nhận MySQL service đang chạy
- Kiểm tra username/password trong config files
- Verify database đã được tạo

**Problem: Servers không kết nối với nhau**

- Kiểm tra IP/port configuration trong tất cả INI files
- Đảm bảo servers được start theo đúng thứ tự
- Check firewall rules

## Phần 7: Tạo Game World (Setup Game Content)

### 7.1 Script và Settings

**Game scripts location:** `/JX/gameserver/script/`

- Lua scripts cho quests, NPCs, items, skills
- Được load tự động khi GameServer khởi động

**Settings location:** `/JX/gameserver/setting/`

- Config files cho maps, missions, events, items
- Format: TXT và INI files

**GameCenter scripts:** `/JX/gamecenter/script/`

- Server-side scripts cho account management, auctions, mail, etc.

### 7.2 Cấu hình Maps và Worlds

**File: `/JX/gameserver/servercfg.ini`**

```ini
[WorldSet]
Count=9
World00=1
World01=2
World02=3
# ... add more map IDs
```

### 7.3 Tạo GM Account

**Direct database:**

```sql
USE 91jx_gamecenter;
-- Insert GM account (details depend on your schema)
INSERT INTO account (username, password, gm_level) VALUES ('admin', 'hashed_password', 99);
```

## Phần 8: Client Setup và Chạy Game

### 8.1 Build Client

**Project file:** `gameworld/client/gameclient/s3client.vcproj`

**Build trong Visual Studio 2005/2008/2010:**

1. Mở `gameworld/client/gameclient/s3client.vcproj`
2. Chọn configuration: Debug hoặc Release
3. Build → Build s3client
4. Output: 

   - Debug: `s3clientd.exe`
   - Release: `s3client.exe`

**Dependencies cần thiết:**

- `engine.dll` (từ `/base/product/`)
- `lua5.dll` (từ `/devenv/product/`)
- `represent2.dll` hoặc `represent3.dll` (2D hoặc 3D graphics)
- `sound.dll` (audio engine)
- `text.dll` (text rendering)
- Các DLL khác trong `/base/product/` và `/kgc/product/`

### 8.2 Cấu Hình Client

**File quan trọng nhất: `\user\serverlistdebug.ini` hoặc `\user\serverlist.dat`**

Client đọc danh sách Gateway servers từ file này. Bạn cần tạo file trong thư mục client.

**Tạo file `\user\serverlistdebug.ini`:**

```ini
[List]
RegionCount=1
AdviceRegion=1
Region_1=Test Server

[Region_1]
Count=1
1_Address=127.0.0.1:5633
1_Title=Local Test Server
1_State=畅通
1_GatewayName=TestGateway
```

**Giải thích:**

- `RegionCount`: Số lượng region (khu vực server)
- `Region_1`: Tên hiển thị của region
- `[Region_1]`: Section chứa danh sách servers trong region 1
- `Count`: Số lượng servers trong region này
- `1_Address`: IP:Port của Gateway server (format: IP:PORT)
- `1_Title`: Tên hiển thị của server
- `1_State`: Trạng thái server (畅通=thông suốt, 繁忙=bận, 维护=bảo trì)
- `1_GatewayName`: Tên gateway (dùng internal)

**Multiple servers example:**

```ini
[List]
RegionCount=2
AdviceRegion=1
Region_1=Viet Nam Servers
Region_2=Test Servers

[Region_1]
Count=2
1_Address=192.168.1.100:5633
1_Title=Server 1 - Thanh Long
1_State=畅通
1_GatewayName=S1
2_Address=192.168.1.101:5633
2_Title=Server 2 - Bach Ho
2_State=繁忙
2_GatewayName=S2

[Region_2]
Count=1
1_Address=127.0.0.1:5633
1_Title=Local Test
1_State=畅通
1_GatewayName=LocalTest
```

### 8.3 Cấu Trúc Thư Mục Client

**Thư mục client cần có:**

```
/client_folder/
  ├── s3client.exe (hoặc s3clientd.exe)
  ├── engine.dll
  ├── lua5.dll
  ├── represent2.dll hoặc represent3.dll
  ├── sound.dll
  ├── text.dll
  ├── dumper.dll
  ├── infoclient.dll
  ├── /user/
  │   └── serverlistdebug.ini (hoặc serverlist.dat)
  ├── /ui/ (UI resources)
  ├── /spr/ (sprite files)
  ├── /sound/ (sound files)
  └── /pak/ (game data packages)
```

### 8.4 Chạy Client

**Windows:**

```cmd
cd /d C:\path\to\client_folder
s3client.exe
```

hoặc Debug version:

```cmd
s3clientd.exe
```

**Luồng hoạt động của Client:**

1. Client khởi động, load engine và resources
2. Hiển thị màn hình login
3. Người chơi chọn Region và Server từ list (đọc từ `serverlistdebug.ini`)
4. Client kết nối đến Gateway server (port 5633)
5. Người chơi đăng nhập với account/password
6. Gateway xác thực và gửi GameServer IP/Port cho client
7. Client kết nối đến GameServer (port 6041)
8. Vào game world

### 8.5 Testing Client-Server Connection

**Để test connection:**

1. **Đảm bảo servers đang chạy:**
   ```cmd
   netstat -ano | findstr "5633"
   ```


Phải thấy port 5633 (Gateway) đang LISTENING

2. **Tạo `\user\serverlistdebug.ini` với IP đúng:**
   ```ini
   [List]
   RegionCount=1
   Region_1=Local
   
   [Region_1]
   Count=1
   1_Address=127.0.0.1:5633
   1_Title=Local Server
   1_GatewayName=Local
   ```

3. **Chạy client và check logs:**

   - Client logs thường ở `\log\` hoặc `\ErrLog\`
   - Xem connection errors

**Common Client Issues:**

- **"Cannot find server list"**: File `serverlistdebug.ini` không tồn tại hoặc sai path
- **"Connection failed"**: Gateway server không chạy hoặc IP/Port sai
- **"Missing DLL"**: Thiếu các DLL dependencies, cần copy từ `/base/product/`
- **Graphics error**: Thiếu `represent*.dll` hoặc DirectX 9 chưa cài

### 8.6 Client Configuration Files (Advanced)

**Các file config khác của client:**

- `\setting\client.ini` - Client settings (nếu có)
- `\ui\` - UI layout configs
- User preferences được lưu trong registry hoặc local files

**AutoLogin Plugin (Optional):**

Trong `/gameworld/client/plugin/autologin/` có plugin tự động đăng nhập cho testing.

**File:** `autologin示例.ini`

```ini
[AutoLogin]
Enable=1
Account=testuser
Password=testpass
```

## Phần 9: Maintenance và Updates

### 9.1 Backup

**Database:**

```bash
mysqldump -u root -p 91jx_gamecenter > backup_gamecenter.sql
mysqldump -u root -p 91jx_statlog > backup_statlog.sql
```

**Game files:**

```bash
tar -czf jx_backup_$(date +%Y%m%d).tar.gz /path/to/JX/
```

### 9.2 Restart Servers

**Graceful shutdown:**

1. Stop GameServer first (để players disconnect)
2. Stop GameCenter
3. Stop Bishop
4. Stop LogServer
5. Stop Goddess
6. Restart theo thứ tự ngược lại

## Phần 10: Advanced Topics

### 10.1 Multi-Server Setup

Để chạy nhiều GameServer instances:

- Copy `/JX/gameserver/` to separate directories
- Thay đổi `ServerID` và `Port` trong mỗi `servercfg.ini`
- Update GameCenter config để biết tất cả GameServers

### 10.2 Performance Tuning

**MySQL optimization:**

```ini
[mysqld]
max_connections=500
innodb_buffer_pool_size=1G
query_cache_size=64M
```

**Server overload settings:**

```ini
[Overload]
MaxPlayer=1000
Precision=0
```

## Tóm Tắt Quick Start

**Minimal setup để test:**

1. Install MSVC6, MySQL
2. Build projects: jxex_lib → coregroup → gc_core → servers
3. Run integrate.bat debug
4. Setup MySQL databases
5. Edit config files (IPs, ports, passwords)
6. Start servers: goddess → logserver → bishop → gamecenter → gameserver
7. Check logs for errors

**Quan trọng:**

- Luôn build theo thứ tự dependencies (libs trước, executables sau)
- Config database trước khi start servers
- Start servers theo đúng thứ tự
- Monitor logs để catch errors sớm
