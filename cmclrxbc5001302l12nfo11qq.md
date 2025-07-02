---
title: "Daily Bugle – TryHackMe Write‑up"
datePublished: Wed Jul 02 2025 09:48:11 GMT+0000 (Coordinated Universal Time)
cuid: cmclrxbc5001302l12nfo11qq
slug: daily-bugle-tryhackme-writeup

---

**Tóm tắt:**  
• **Mục tiêu:** thu thập *user* & *root flag* trên máy Daily Bugle.  
• **Độ khó:** *Easy* (Theo THM).  
• **Hệ điều hành:** Linux (CentOS).  
• **Flags:** 27a260fe3cba712cfdeb1c86d80442e, eec3d53292b1821868266858d7fa6f79.

## 1\. Reconnaissance & Enumeration

![image1.png](https://github.com/huynhlonghehe/Daily_Bugle/blob/main/media/image1.png?raw=true align="left")

Thực hiện quét cổng và kiểm tra dịch vụ đang mở phát hiện:

* **Lý do full-port scan:** nhiều box giấu service ở ngoài top 1000, vì vậy luôn quét -p-
    
* Kết quả quét phát hiện các cổng sau đang mở:
    
    * **22/tcp:** Dịch vụ SSH, chạy phiên bản OpenSSH 7.4 (protocol 2.0).
        
    * **80/tcp:** Dịch vụ HTTP, chạy Apache 2.4.6 (CentOS), ứng dụng sử dụng Joomla CMS, PHP 5.6.40.
        
    * **3306/tcp:** Dịch vụ cơ sở dữ liệu MySQL (MariaDB).
        
* Ngoài ra, Nmap cũng thu thập được file robots.txt chưa được ẩn, có chứa nhiều mục cấm truy cập nhưng vẫn có thể kiểm tra được.
    

Thử đọc robots.txt:

![image2.png](https://github.com/huynhlonghehe/Daily_Bugle/blob/main/media/image2.png?raw=true align="left")

/administrator/, /bin/, /cache/, /components/, /language/, /libraries/, /logs/, /modules/, /plugins/, /tmp/...

* Đặc biệt thư mục /administrator/ có thể là cổng vào trang quản trị Joomla.
    
* Truy cập thử: http://&lt;ip&gt;/administrator
    

![image3.png](https://github.com/huynhlonghehe/Daily_Bugle/blob/main/media/image3.png?raw=true align="left")

Xuất hiện trang đăng nhập vào trang quản trị joomla

Thử xác định phiên bản của joomla http://&lt;IP&gt;/administrator/manifests/files/joomla.xml  
Đây thường là đường dẫn mặc định trên joomla\\

![image4.png](https://github.com/huynhlonghehe/Daily_Bugle/blob/main/media/image4.png?raw=true align="left")

* Xác định được phiên bản joomla mà trang web đang dùng là 3.7.0
    
* Thử tìm kiếm các CVE/ các lỗ hổng có thể khai thác trên phiên bản này
    
* Sử dụng Searchsploit để tìm kiếm các lỗ hổng của Joomla 3.7.0:
    

![image5.png](https://github.com/huynhlonghehe/Daily_Bugle/blob/main/media/image5.png?raw=true align="left")

phát hiện lỗ hổng SQL Injection 42033 (CVE-2017-8917) liên quan đến com\_fields

![image6.png](https://github.com/huynhlonghehe/Daily_Bugle/blob/main/media/image6.png?raw=true align="left")

![image7.png](https://github.com/huynhlonghehe/Daily_Bugle/blob/main/media/image7.png?raw=true align="left")

Sử dụng exploit-db để thu thập thông tin về lỗ hổng này và hướng khai thác.

## 2\. Initial Access – SQL Injection on Joomla 3.7.0

* Phiên bản **Joomla 3.7.0** dính **CVE‑2017‑8917** (SQLi trong *com\_fields*). Lỗi cho phép UNION‑based truy vấn DB ➜ dump credential.
    

Hướng dẫn khai thác bằng sqlmap nhưng cách đó khá chậm. Tôi thử tìm kiếm về CVE-2017-8917

![image8.png](https://github.com/huynhlonghehe/Daily_Bugle/blob/main/media/image8.png?raw=true align="left")

Tìm kiếm được 1 số tool để khai thác lỗ hổng này.  
Dùng script `joomblaH` từ GitHub để khai thác

![image9.png](https://github.com/huynhlonghehe/Daily_Bugle/blob/main/media/image9.png?raw=true align="left")

Kết quả thu được người dùng:

* Found user \['811', 'Super User', 'jonah', '[jonah@tryhackme.com](mailto:jonah@tryhackme.com)','$2y$10$0veO/JSFh4389Lluc4Xya.dfy2MF.bZhz0jVMw.V.d3p12kBtZutm', '', ''\]
    

![image.png](https://github.com/huynhlonghehe/Daily_Bugle/blob/main/media/image.png?raw=true align="left")

* Đây là người dùng ‘jonah’ là 'Super User'
    
* Crack hash → **spiderman123**. Đăng nhập /administrator với quyền *Super User*.
    

## 3\. Post‑Auth RCE

Từ quyền quản trị, tôi chèn mã reverse shell PHP từ Pentestmonkey vào file error.php của template Protostar. Thực hiện như sau:

Truy cập Extensions/templates/templates

![image12.png](https://github.com/huynhlonghehe/Daily_Bugle/blob/main/media/image12.png?raw=true align="left")

Chọn templates muốn chèn/upload

![image13.png](https://github.com/huynhlonghehe/Daily_Bugle/blob/main/media/image13.png?raw=true align="left")

**Payload**  
Chúng ta dùng *php‑reverse‑shell* của **pentestmonkey** (≈ 200 dòng). Khi file chạy, hàm fsockopen() mở kết nối TCP ra **$ip:$port** (bạn sửa thành IP & port máy tấn công). Tiếp đó script:

1. proc\_open('/bin/sh -i') sinh shell tương tác.
    
2. Đặt mọi stream thành **non‑blocking** và dùng stream\_select() để:  
    • chuyển dữ liệu **attacker → STDIN** shell  
    • chuyển **STDOUT/STDERR → attacker**  
    ⇒ Bạn nhận **/bin/sh -i** chạy dưới quyền **apache**.
    

![image14.png](https://github.com/huynhlonghehe/Daily_Bugle/blob/main/media/image14.png?raw=true align="left")

**Triển khai & kích hoạt**

**Cách A – sửa template hiện hữu**

1. Vào **Extensions → Templates → Templates → Protostar → Edit → error.php**.
    
2. Chèn payload cuối file, đổi $ip, $port.
    
3. Mở http://&lt;TARGET\_IP&gt;/templates/protostar/error.php ➜ payload thực thi, shell call‑home.
    

![image15.png](https://github.com/huynhlonghehe/Daily_Bugle/blob/main/media/image15.png?raw=true align="left")

![image16.png](https://github.com/huynhlonghehe/Daily_Bugle/blob/main/media/image16.png?raw=true align="left")

* Trên máy attacker thiết lập lắng nghe tại port 9921
    
* Truy cập http://&lt;IP&gt;/templates/protostar/error.php để kích hoạt reverse shell vừa nhúng nào error.php, shell này thực hiện mở port reverse
    

![image17.png](https://github.com/huynhlonghehe/Daily_Bugle/blob/main/media/image17.png?raw=true align="left")

**Cách B – upload template mới/php mới** (ít log hơn, khó bị phát hiện hơn). Upload ZIP/file php chứa payload, sau đó truy cập file `.php` để kích hoạt.

\+ Cách làm tương tự nhưng ta sẽ thực hiện upload/tạo file shell thay vì cố gắn chỉnh sửa

![image18.png](https://github.com/huynhlonghehe/Daily_Bugle/blob/main/media/image18.png?raw=true align="left")

![image19.png](https://github.com/huynhlonghehe/Daily_Bugle/blob/main/media/image19.png?raw=true align="left")

![image20.png](https://github.com/huynhlonghehe/Daily_Bugle/blob/main/media/image20.png?raw=true align="left")

## 4\. Privilege Escalation Privilege Escalation

![image21.png](https://github.com/huynhlonghehe/Daily_Bugle/blob/main/media/image21.png?raw=true align="left")

![image22.png](https://github.com/huynhlonghehe/Daily_Bugle/blob/main/media/image22.png?raw=true align="left")

* Sau khi tải lên hoặc chèn reverse shell và kích hoạt, máy attacker đã nhận được kết nối từ reverse shell.
    
* Với lệnh whoami xác định được user hiện tại là apache, với quyền rất hạn chế.
    

### 4.1 Pivot sang jjameson

![image23.png](https://github.com/huynhlonghehe/Daily_Bugle/blob/main/media/image23.png?raw=true align="left")

* Để xác định các tài khoản user trên hệ thống, tôi liệt kê thư mục `/home`, nơi mặc định chứa các thư mục cá nhân của người dùng.
    
* Dùng lệnh ls -lah xác định được user jjameson
    

![image24.png](https://github.com/huynhlonghehe/Daily_Bugle/blob/main/media/image24.png?raw=true align="left")

* Vì đang là user apache và user apache thì thường dùng để chạy dịch vụ web nên tôi đoán nó có quyền trong thư mục **/var/www/html.**
    
* Truy cập vào đó và dùng lệnh ls -lah, xác nhận rằng user apache có quyền xem file configuration.php
    

![image25.png](https://github.com/huynhlonghehe/Daily_Bugle/blob/main/media/image25.png?raw=true align="left")

* Đọc configuration.php phát hiện biến **password**, rất có thể user **jjameson** sử dụng password này, và lúc đầu recon phát hiện trên server có mở port **ssh**
    
* Giờ thử ssh vào server bằng user jjameson và với password vừa tìm được
    

![image26.png](https://github.com/huynhlonghehe/Daily_Bugle/blob/main/media/image26.png?raw=true align="left")

![image27.png](https://github.com/huynhlonghehe/Daily_Bugle/blob/main/media/image27.png?raw=true align="left")

* Kết quả truy cập ssh vào server thành công với user jjameson.
    
* Đọc file trong thư mục jjameson lúc nãy tìm được thì được user flag 27a260fe3cba712cfdeb1c86d80442e
    

### 4.2 Root qua `yum` sudo

* Kiểm tra quyền sudo:
    
    sudo -l
    
* Phát hiện người dùng jjameson được phép chạy lệnh yum với quyền root mà không cần mật khẩu. Dựa vào thông tin từ **GTFOBins(**[https://gtfobins.github.io/gtfobins/yum/#sudo](https://gtfobins.github.io/gtfobins/yum/#sudo)**)**, chúng tôi khai thác như sau:
    

![image28.png](https://github.com/huynhlonghehe/Daily_Bugle/blob/main/media/image28.png?raw=true align="left")

**Vì sao có thể leo thang?**

* NOPASSWD: /usr/bin/yum cho phép **jjameson** chạy **yum** với UID 0 không cần mật khẩu.
    
* Khi cài gói, **yum → rpm** thực thi script %post (và đọc file cấu hình) **dưới quyền root**.
    

![image29.png](https://github.com/huynhlonghehe/Daily_Bugle/blob/main/media/image29.png?raw=true align="left")

**Tóm tắt bước**

| Bước | Mục đích |
| --- | --- |
| 1 | Tạo thư mục tạm—không cần ghi hệ thống |
| 2 | Tùy biến *yum.conf* để pluginpath = thư mục tạm |
| 3 | *y.conf* khai báo plugin "y" được bật |
| 4 | [*y.py*](http://y.py) là plugin; init\_hook() gọi /bin/sh |
| 5 | Chạy yum với cấu hình tùy ý → shell root |

![image30.png](https://github.com/huynhlonghehe/Daily_Bugle/blob/main/media/image30.png?raw=true align="left")

cat /root/root.txt

eec3d53292b1821868266858d7fa6f79

## 5\. Flags

| Flag | Path |
| --- | --- |
| `27a260fe3cba712cfdeb1c86d80442e` | `/home/jjameson/user.txt` |
| `eec3d53292b1821868266858d7fa6f79` | `/root/root.txt` |

## 6\. Lessons Learned / Mitigation

* **Joomla 3.7.0** ➜ vá lên ≥ 3.7.1.
    
* Không cho **yum**, **apt**, v.v. chạy với *NOPASSWD* trong *sudoers*.
    
* Ẩn hoặc giới hạn truy cập `/robots.txt` trong production.
    
* Tách mật khẩu giữa ứng dụng & hệ thống; bật **Fail2ban** cho SSH.