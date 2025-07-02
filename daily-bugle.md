**Daily Bugle -- TryHackMe Write‑up**

**Tóm tắt:**\
• **Mục tiêu:** thu thập *user* & *root flag* trên máy Daily Bugle.\
• **Độ khó:** *Easy* (Theo THM).\
• **Hệ điều hành:** Linux (CentOS).\
• **Flags:** 27a260fe3cba712cfdeb1c86d80442e,
eec3d53292b1821868266858d7fa6f79.

## 1. Reconnaissance & Enumeration

nmap -sC -sV -p- \<ip\>\
![](D:\HuynhLong_RE\writeUp\media/media/image1.png){width="6.5in"
height="4.256944444444445in"}

Thực hiện quét cổng và kiểm tra dịch vụ đang mở phát hiện:

- **Lý do full‑port scan:** nhiều box giấu service ở ngoài top 1000, vì
  vậy luôn quét -p-

- Kết quả quét phát hiện các cổng sau đang mở:

  - **22/tcp:** Dịch vụ SSH, chạy phiên bản OpenSSH 7.4 (protocol 2.0).

  - **80/tcp:** Dịch vụ HTTP, chạy Apache 2.4.6 (CentOS), ứng dụng sử
    dụng Joomla CMS, PHP 5.6.40.

  - **3306/tcp:** Dịch vụ cơ sở dữ liệu MySQL (MariaDB).

- Ngoài ra, Nmap cũng thu thập được file robots.txt chưa được ẩn, có
  chứa nhiều mục cấm truy cập nhưng vẫn có thể kiểm tra được.

> Thử đọc robots.txt:
>
> ![](D:\HuynhLong_RE\writeUp\media/media/image2.png){width="6.5in"
> height="6.8909722222222225in"}

- /administrator/, /bin/, /cache/, /components/, /language/,
  /libraries/, /logs/, /modules/, /plugins/, /tmp/\...

- Đặc biệt thư mục /administrator/ có thể là cổng vào trang quản trị
  Joomla.

Truy cập thử http://\<ip\>/administrator:

![](D:\HuynhLong_RE\writeUp\media/media/image3.png){width="6.5in"
height="5.1625in"}\
\
Xuất hiện trang đăng nhập vào trang quản trị joomla

Thử xác định phiên bản của joomla\
Đây thường là đường dẫn mặc định trên joomla\
![](D:\HuynhLong_RE\writeUp\media/media/image4.png){width="6.5in"
height="4.029166666666667in"}

Xác định được phiên bản joomla mà trang web đang dùng là 3.7.0\
\
\
Thử tìm kiếm các CVE/ các lỗ hổng có thể khai thác trên phiên bản này\
Sử dụng Searchsploit để tìm kiếm các lỗ hổng của Joomla 3.7.0:

![](D:\HuynhLong_RE\writeUp\media/media/image5.png){width="6.5in"
height="2.7756944444444445in"}

phát hiện lỗ hổng SQL Injection 42033 (CVE-2017-8917) liên quan đến
com_fields

![](D:\HuynhLong_RE\writeUp\media/media/image6.png){width="6.5in"
height="3.2583333333333333in"}

![](D:\HuynhLong_RE\writeUp\media/media/image7.png){width="6.5in"
height="3.265972222222222in"}

Sử dụng exploit-db để thu thập thông tin về lỗ hổng này và hướng khai
thác.

##  2. Initial Access -- SQL Injection on Joomla 3.7.0

- Phiên bản **Joomla 3.7.0** dính **CVE‑2017‑8917** (SQLi trong
  *com_fields*). Lỗi cho phép UNION‑based truy vấn DB ➜ dump credential.

Hướng dẫn khai thác bằng sqlmap nhưng cách đó khá chậm. Tôi thử tìm kiếm
về CVE-2017-8917\
\
![](D:\HuynhLong_RE\writeUp\media/media/image8.png){width="6.5in"
height="2.1868055555555554in"}\
\
Tìm kiếm được 1 số tool để khai thác lỗ hổng này.\
Dùng script joomblaH từ GitHub để khai thác\
![](D:\HuynhLong_RE\writeUp\media/media/image9.png){width="6.5in"
height="2.6534722222222222in"}\
\
Kết quả thu được người dùng:\
Found user \[\'811\', \'Super User\', \'jonah\',
\'jonah@tryhackme.com\',
\'\$2y\$10\$0veO/JSFh4389Lluc4Xya.dfy2MF.bZhz0jVMw.V.d3p12kBtZutm\',
\'\', \'\'\]\
![](D:\HuynhLong_RE\writeUp\media/media/image10.png){width="6.5in"
height="2.9659722222222222in"}

- Đây là người dùng 'jonah' là \'Super User\'

- Crack hash → **spiderman123**. Đăng nhập /administrator với quyền
  *Super User*.

![](D:\HuynhLong_RE\writeUp\media/media/image11.png){width="6.5in"
height="3.26875in"}

Sử dụng thông tin đăng nhập thu được để vào trang quản trị Joomla:

- URL: http://\<IP\>/administrator

- Username: jonah

- Password: spiderman123

Truy cập thành công với quyền **Super User**.

## 3. Post‑Auth RCE

Từ quyền quản trị, tôi chèn mã reverse shell PHP từ Pentestmonkey vào
file template error.php của Protostar. Thực hiện như sau:

Truy cập Extensions/templates/templates

![](D:\HuynhLong_RE\writeUp\media/media/image12.png){width="6.5in"
height="3.359722222222222in"}

Chọn templates

![](D:\HuynhLong_RE\writeUp\media/media/image13.png){width="6.5in"
height="3.3194444444444446in"}

**Payload**\
Chúng ta dùng *php‑reverse‑shell* của **pentestmonkey** (≈ 200 dòng).
Khi file chạy, hàm fsockopen() mở kết nối TCP ra **\$ip:\$port** (bạn
sửa thành IP & port máy tấn công). Tiếp đó script:

1.  proc_open(\'/bin/sh -i\') sinh shell tương tác.

2.  Đặt mọi stream thành **non‑blocking** và dùng stream_select() để:\
    • chuyển dữ liệu **attacker → STDIN** shell\
    • chuyển **STDOUT/STDERR → attacker**\
    ⇒ Bạn nhận **/bin/sh -i** chạy dưới quyền **apache**.

![](D:\HuynhLong_RE\writeUp\media/media/image14.png){width="6.5in"
height="3.2604166666666665in"}

**Triển khai & kích hoạt**

**Cách A -- sửa template hiện hữu**

1.  Vào **Extensions → Templates → Protostar → Edit → error.php**.

2.  Chèn payload cuối file, đổi \$ip, \$port.

3.  Mở http://\<TARGET_IP\>/templates/protostar/error.php ➜ payload thực
    thi, shell call‑home.

![](D:\HuynhLong_RE\writeUp\media/media/image15.png){width="6.5in"
height="3.2708333333333335in"}

![](D:\HuynhLong_RE\writeUp\media/media/image16.png){width="4.521464348206474in"
height="0.4271434820647419in"}

Trên máy attacker thiết lập lắng nghe tại port 9921

Truy cập để kích hoạt reverse shell vừa nhúng nào error.php, shell này
thực hiện mở port reverse\
![](D:\HuynhLong_RE\writeUp\media/media/image17.png){width="6.5in"
height="1.53125in"}

**Cách B -- upload template mới/php mới** (ít log hơn, khó bị phát hiện
hơn). Upload ZIP/file php chứa payload, sau đó truy cập file .php để
kích hoạt.\
\
Cách làm tương tự nhưng ta sẽ thực hiện upload/tạo file shell thay vì cố
gắn chỉnh sửa

![](D:\HuynhLong_RE\writeUp\media/media/image18.png){width="6.5in"
height="3.8243055555555556in"}![](D:\HuynhLong_RE\writeUp\media/media/image19.png){width="6.5in"
height="2.439583333333333in"}

![](D:\HuynhLong_RE\writeUp\media/media/image20.png){width="6.5in"
height="1.1354166666666667in"}

## 4. Privilege Escalation Privilege Escalation

![](D:\HuynhLong_RE\writeUp\media/media/image21.png){width="6.5in"
height="1.6291666666666667in"}

![](D:\HuynhLong_RE\writeUp\media/media/image22.png){width="6.5in"
height="2.740972222222222in"}

Với lệnh whoami xác định được user hiện tại là apache, với quyền rất hạn
chế.

### 4.1 Pivot sang jjameson

![](D:\HuynhLong_RE\writeUp\media/media/image23.png){width="6.5in"
height="1.961111111111111in"}

Dùng lệnh ls -lah xác định được user jjameson

![](D:\HuynhLong_RE\writeUp\media/media/image24.png){width="6.5in"
height="3.551388888888889in"}

- Vì đang là user apache và user apache thì thường dùng để chạy dịch vụ
  web nên tôi đoán nó có quyền trong thư mục **/var/www/html.**

- Truy cập vào đó và dùng lệnh ls -lah, xác nhận rằng user apache có
  quyền xem file configuration.php

![](D:\HuynhLong_RE\writeUp\media/media/image25.png){width="6.5in"
height="3.592361111111111in"}Đọc configuration.php phát hiện biến
**password**, rất có thể user **jjameson** sử dụng password này, và lúc
đầu recon phát hiện trên server có mở port **ssh**

Giờ thử ssh vào server bằng user jjameson và với password vừa tìm được\
\
![](D:\HuynhLong_RE\writeUp\media/media/image26.png){width="6.5in"
height="2.3340277777777776in"}\
\
![](D:\HuynhLong_RE\writeUp\media/media/image27.png){width="5.094460848643919in"
height="0.8334492563429571in"}\
\
Kết quả truy cập ssh vào server thành công với user jjameson.

Đọc file trong thư mục jjameson lúc nãy tìm được thì được user flag
27a260fe3cba712cfdeb1c86d80442e

### 4.2 Root qua yum sudo

Kiểm tra quyền sudo:

sudo -l

Phát hiện người dùng jjameson được phép chạy lệnh yum với quyền root mà
không cần mật khẩu. Dựa vào thông tin từ GTFOBins, chúng tôi khai thác
như sau:

![](D:\HuynhLong_RE\writeUp\media/media/image28.png){width="6.5in"
height="1.945138888888889in"}

**Vì sao có thể leo thang?**

- NOPASSWD: /usr/bin/yum cho phép **jjameson** chạy **yum** với UID 0
  không cần mật khẩu.

- Khi cài gói, **yum → rpm** thực thi script %post (và đọc file cấu
  hình) **dưới quyền root**.

https://gtfobins.github.io/gtfobins/yum/#sudo\
![](D:\HuynhLong_RE\writeUp\media/media/image29.png){width="6.442224409448819in"
height="5.350463692038495in"}

**Tóm tắt bước**

  -----------------------------------------------------------------------
    Bước    Mục đích
  --------- -------------------------------------------------------------
      1     Tạo thư mục tạm---không cần ghi hệ thống

      2     Tùy biến *yum.conf* để pluginpath = thư mục tạm

      3     *y.conf* khai báo plugin \"y\" được bật

      4     *y.py* là plugin; init_hook() gọi /bin/sh

      5     Chạy yum với cấu hình tùy ý → shell root
  -----------------------------------------------------------------------

![](D:\HuynhLong_RE\writeUp\media/media/image30.png){width="6.5in"
height="1.2243055555555555in"}\
cat /root/root.txt

eec3d53292b1821868266858d7fa6f79

## 5. Flags

  -----------------------------------------------------------------------
  Flag                                     Path
  ---------------------------------------- ------------------------------
  27a260fe3cba712cfdeb1c86d80442e          /home/jjameson/user.txt

  eec3d53292b1821868266858d7fa6f79         /root/root.txt
  -----------------------------------------------------------------------

## 6. Lessons Learned / Mitigation

- **Joomla 3.7.0** ➜ vá lên ≥ 3.7.1.

- Không cho **yum**, **apt**, v.v. chạy với *NOPASSWD* trong *sudoers*.

- Ẩn hoặc giới hạn truy cập /robots.txt trong production.

- Tách mật khẩu giữa ứng dụng & hệ thống; bật **Fail2ban** cho SSH.
