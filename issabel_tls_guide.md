# راهنمای کامل فعال‌سازی TLS در Issabel
### برای اتصال امن تلفن‌های IP از طریق اینترنت

---

## 📋 فهرست مطالب

1. [مقدمه](#مقدمه)
2. [پیش‌نیازها](#پیش-نیازها)
3. [مرحله ۱: ساخت گواهی SSL](#مرحله-۱-ساخت-گواهی-ssl)
4. [مرحله ۲: پیکربندی Asterisk برای TLS](#مرحله-۲-پیکربندی-asterisk-برای-tls)
5. [مرحله ۳: تنظیمات NAT](#مرحله-۳-تنظیمات-nat)
6. [مرحله ۴: پیکربندی Firewall](#مرحله-۴-پیکربندی-firewall)
7. [مرحله ۵: پیکربندی Fail2Ban](#مرحله-۵-پیکربندی-fail2ban)
8. [مرحله ۶: پیکربندی Extension](#مرحله-۶-پیکربندی-extension)
9. [مرحله ۷: تنظیمات تلفن Yealink](#مرحله-۷-تنظیمات-تلفن-yealink)
10. [مرحله ۸: Port Forwarding در روتر](#مرحله-۸-port-forwarding-در-روتر)
11. [عیب‌یابی](#عیب-یابی)
12. [نکات امنیتی](#نکات-امنیتی)

---

## مقدمه

این آموزش به شما کمک می‌کند تا ارتباط تلفن‌های IP با سرور Issabel را از طریق اینترنت **رمزنگاری** کنید. با استفاده از TLS (Transport Layer Security):
- اطلاعات لاگین (username/password) رمزنگاری می‌شود
- سیگنال‌های SIP امن می‌شوند
- همراه با SRTP، صدای تماس نیز رمزنگاری می‌شود

---

## پیش‌نیازها

✅ سرور Issabel نصب شده (CentOS 7 یا بالاتر)  
✅ دسترسی SSH به سرور با کاربر root  
✅ آی‌پی عمومی ثابت یا دامنه (مثل: `voip.example.com`)  
✅ دسترسی به تنظیمات روتر برای Port Forwarding  
✅ تلفن IP از نوع Yealink (یا سایر برندها با پشتیبانی TLS)  

---

## مرحله ۱: ساخت گواهی SSL

### ۱.۱ اتصال به سرور

```bash
ssh root@IP_SERVER
```

### ۱.۲ ساخت پوشه برای گواهی‌ها

```bash
mkdir -p /etc/asterisk/keys
cd /etc/asterisk/keys
```

### ۱.۳ ساخت کلید خصوصی (Private Key)

```bash
openssl genrsa -out asterisk.key 2048
```

### ۱.۴ ساخت گواهی (Certificate) - اعتبار 10 سال

```bash
openssl req -new -x509 -days 3650 -key asterisk.key -out asterisk.crt
```

**هنگام اجرا این سوالات پرسیده می‌شود:**

```
Country Name (2 letter code): IR
State or Province Name: Tehran
Locality Name (eg, city): Tehran
Organization Name: Your Company Name
Organizational Unit Name: IT
Common Name (FQDN): voip.example.com  ← مهم: دامنه یا IP سرور
Email Address: admin@example.com
```

**نکته مهم:** در قسمت **Common Name** حتماً دامنه سرور (مثل `voip.example.com`) یا IP عمومی سرور را وارد کنید.

### ۱.۵ ترکیب کلید و گواهی

```bash
cat asterisk.key asterisk.crt > asterisk.pem
```

### ۱.۶ تنظیم دسترسی‌ها

```bash
chmod 600 /etc/asterisk/keys/*
chown asterisk:asterisk /etc/asterisk/keys/*
```

### ۱.۷ بررسی تاریخ انقضای گواهی

```bash
openssl x509 -in /etc/asterisk/keys/asterisk.crt -noout -dates
```

---

## مرحله ۲: پیکربندی Asterisk برای TLS

### ۲.۱ بکاپ از فایل پیکربندی

```bash
cp /etc/asterisk/sip_general_custom.conf /etc/asterisk/sip_general_custom.conf.backup
```

### ۲.۲ ویرایش فایل پیکربندی

```bash
nano /etc/asterisk/sip_general_custom.conf
```

### ۲.۳ اضافه کردن تنظیمات TLS

این خطوط را به **انتهای فایل** اضافه کنید:

```ini
tlsenable=yes
tlsbindaddr=0.0.0.0:5061
tlscertfile=/etc/asterisk/keys/asterisk.pem
tlsprivatekey=/etc/asterisk/keys/asterisk.key
tlscafile=/etc/asterisk/keys/asterisk.crt
tlscipher=ALL
tlsclientmethod=tlsv1
```

**توضیح پارامترها:**
- `tlsenable=yes` → فعال‌سازی TLS
- `tlsbindaddr=0.0.0.0:5061` → گوش دادن به پورت 5061 (استاندارد SIP TLS)
- `tlscertfile` → مسیر گواهی ترکیبی
- `tlsprivatekey` → مسیر کلید خصوصی
- `tlscafile` → مسیر گواهی CA
- `tlscipher=ALL` → استفاده از همه رمزنگاری‌های موجود

### ۲.۴ ذخیره فایل

در nano: `Ctrl+O` → Enter → `Ctrl+X`

### ۲.۵ ریلود کردن Asterisk

```bash
asterisk -rx "core reload"
```

### ۲.۶ بررسی فعال بودن TLS

```bash
asterisk -rx "sip show settings" | grep -i tls
```

**خروجی مورد انتظار:**
```
  TLS SIP Bindaddress:    0.0.0.0:5061
```

### ۲.۷ بررسی پورت 5061

```bash
netstat -tuln | grep 5061
```

**خروجی مورد انتظار:**
```
tcp        0      0 0.0.0.0:5061            0.0.0.0:*               LISTEN
```

---

## مرحله ۳: تنظیمات NAT

اگر سرور پشت NAT (روتر) قرار دارد و تلفن‌ها از اینترنت متصل می‌شوند:

### ۳.۱ پیدا کردن آی‌پی عمومی

```bash
curl ifconfig.me
```

مثال خروجی: `15.255.228.23`

### ۳.۲ ویرایش فایل پیکربندی

```bash
nano /etc/asterisk/sip_general_custom.conf
```

### ۳.۳ اضافه کردن تنظیمات NAT

این خطوط را اضافه کنید:

```ini
externip=15.255.228.23
localnet=192.168.88.0/255.255.255.0
nat=force_rport,comedia
```

**توضیح:**
- `externip` → آی‌پی عمومی سرور (یا دامنه مثل `voip.example.com`)
- `localnet` → شبکه داخلی (بسته به شبکه شما تغییر دهید)
- `nat=force_rport,comedia` → تنظیمات NAT traversal

### ۳.۴ ذخیره و ریلود

```bash
asterisk -rx "sip reload"
```

---

## مرحله ۴: پیکربندی Firewall

### ۴.۱ بررسی وضعیت Firewall

```bash
systemctl status firewalld
```

اگر غیرفعال بود:

```bash
systemctl start firewalld
systemctl enable firewalld
```

### ۴.۲ باز کردن پورت‌های لازم

```bash
firewall-cmd --permanent --add-port=5061/tcp
firewall-cmd --permanent --add-port=10000-20000/udp
firewall-cmd --permanent --add-port=80/tcp
firewall-cmd --permanent --add-port=443/tcp
firewall-cmd --reload
```

**توضیح پورت‌ها:**
- `5061/tcp` → SIP با TLS
- `10000-20000/udp` → RTP (صدا)
- `80/tcp` → دسترسی به وب (HTTP)
- `443/tcp` → دسترسی به وب (HTTPS)

### ۴.۳ بررسی فایروال

```bash
firewall-cmd --list-all
```

**خروجی مورد انتظار:**
```
ports: 80/tcp 443/tcp 5061/tcp 10000-20000/udp
```

---

## مرحله ۵: پیکربندی Fail2Ban

Fail2Ban از تلاش‌های ورود غیرمجاز جلوگیری می‌کند.

### ۵.۱ بررسی نصب بودن

```bash
systemctl status fail2ban
```

اگر نصب نبود:

```bash
yum install fail2ban -y
systemctl start fail2ban
systemctl enable fail2ban
```

### ۵.۲ پیکربندی برای Asterisk

```bash
nano /etc/fail2ban/jail.local
```

این محتوا را بنویسید:

```ini
[asterisk]
enabled = true
port = 5060,5061
filter = asterisk
logpath = /var/log/asterisk/full
maxretry = 5
bantime = 3600
findtime = 600
action = iptables-allports[name=ASTERISK]
```

**توضیح:**
- `maxretry = 5` → بعد از 5 تلاش ناموفق
- `bantime = 3600` → 1 ساعت بلاک می‌شود
- `findtime = 600` → در بازه 10 دقیقه

### ۵.۳ ریستارت Fail2Ban

```bash
systemctl restart fail2ban
```

### ۵.۴ بررسی فعال بودن

```bash
fail2ban-client status asterisk
```

---

## مرحله ۶: پیکربندی Extension

### ۶.۱ از رابط وب Issabel

1. وارد پنل Issabel شوید: `https://IP_SERVER`
2. برید به: **PBX** → **PBX Configuration**
3. از منوی چپ: **Extensions**
4. روی Extension مورد نظر کلیک کنید (مثلاً 102)
5. به قسمت **Device Options** بروید
6. **Add Custom Key/Value Pair** را کلیک کنید
7. وارد کنید:
   - **Key**: `transport`
   - **Value**: `tls`
8. روی **Submit** کلیک کنید
9. بالای صفحه دکمه قرمز **Apply Config** را بزنید

### ۶.۲ یا از خط فرمان

```bash
nano /etc/asterisk/sip_additional.conf
```

پیدا کنید بخش Extension (مثلاً `[102]`):

```ini
[102]
secret=رمز_قوی_16_کاراکتری
context=from-internal
type=friend
host=dynamic
transport=tls          ← این خط را اضافه کنید
```

ذخیره کنید و:

```bash
asterisk -rx "sip reload"
```

### ۶.۳ بررسی Extension

```bash
asterisk -rx "sip show peer 102" | grep -i transport
```

**خروجی مورد انتظار:**
```
  Prim.Transp. : TLS
  Allowed.Trsp : TLS
```

### ۶.۴ تنظیم رمز عبور قوی (خیلی مهم!)

```bash
nano /etc/asterisk/sip_additional.conf
```

در بخش Extension:

```ini
secret=A1b2C3d4E5f6G7h8  ← حداقل 16 کاراکتر، ترکیبی از حروف، اعداد و علائم
```

---

## مرحله ۷: تنظیمات تلفن Yealink

### ۷.۱ اتصال از شبکه داخلی

#### الف) دسترسی به رابط وب تلفن

1. آی‌پی تلفن را در مرورگر باز کنید: `http://192.168.1.X`
2. Username/Password پیش‌فرض: `admin/admin`

#### ب) پیکربندی Account

برید به: **Account** → **Register**

```
Line Active: Enabled
Label: Extension 102
Display Name: نام شما
Register Name: 102
User Name: 102
Password: رمز_Extension_از_Issabel
SIP Server 1: 192.168.1.4  (یا IP سرور Issabel)
Port: 5061
Transport: TLS                ← خیلی مهم
Outbound Status: Disabled
```

#### ج) تنظیمات SRTP (رمزنگاری صدا)

برید به: **Account** → **Advanced** یا **Security**

```
RTP Encryption (SRTP): Optional  ← یا Enabled
```

#### د) تنظیمات گواهی SSL

برید به: **Security** → **Server Certificates**

```
Trust Certificates: All  ← یا Custom CA را آپلود کنید
```

#### ه) ذخیره تنظیمات

روی **Save** کلیک کنید. تلفن خودکار ریستارت می‌شود.

---

### ۷.۲ اتصال از اینترنت (خارج از شبکه)

#### الف) Port Forwarding را انجام دهید (مرحله ۸)

#### ب) تنظیمات تلفن

همان تنظیمات بالا، فقط:

```
SIP Server 1: voip.example.com  (یا IP عمومی سرور)
Port: 5061
Transport: TLS
```

---

### ۷.۳ بررسی اتصال

از سرور:

```bash
asterisk -rx "sip show peers"
```

**خروجی مورد انتظار:**
```
102/102    192.168.1.28    D  N      A  5061     OK (4 ms)
```

یا:

```bash
asterisk -rx "sip show peer 102"
```

بررسی کنید:
```
Status       : OK (4 ms)
Reg. Contact : sip:102@192.168.1.28:12277;transport=TLS
Prim.Transp. : TLS
Encryption   : Yes
```

---

## مرحله ۸: Port Forwarding در روتر

برای اتصال تلفن از اینترنت، باید پورت‌ها را در روتر Forward کنید.

### ۸.۱ دسترسی به روتر

معمولاً: `http://192.168.1.1` یا `http://192.168.0.1`

### ۸.۲ پیدا کردن قسمت Port Forwarding

نام‌های مختلف:
- Port Forwarding
- Virtual Server
- NAT
- Port Mapping

### ۸.۳ اضافه کردن قوانین

#### قانون ۱: SIP با TLS

```
Service Name: SIP-TLS
External Port: 5061
Protocol: TCP
Internal IP: 192.168.1.4  (IP سرور Issabel در شبکه داخلی)
Internal Port: 5061
Enable: Yes
```

#### قانون ۲: RTP (صدا)

```
Service Name: RTP
External Port: 10000-20000
Protocol: UDP
Internal IP: 192.168.88.4
Internal Port: 10000-20000
Enable: Yes
```

### ۸.۴ ذخیره و ریستارت روتر

### ۸.۵ تست از بیرون

از یک سیستم خارج از شبکه (موبایل با 4G/5G):

1. برید به: https://www.yougetsignal.com/tools/open-ports/
2. وارد کنید:
   - IP: آی‌پی عمومی سرور
   - Port: `5061`
3. Check کنید

اگر نتیجه **Open** بود، موفق بودید! ✅

---

## عیب‌یابی

### ❌ مشکل: تلفن رجیستر نمی‌شود

#### بررسی ۱: TLS فعال است؟

```bash
asterisk -rx "sip show settings" | grep -i tls
```

#### بررسی ۲: پورت 5061 باز است؟

```bash
netstat -tuln | grep 5061
```

#### بررسی ۳: Firewall مشکل دارد؟

```bash
firewall-cmd --list-all
```

#### بررسی ۴: لاگ Asterisk

```bash
tail -f /var/log/asterisk/full
```

بعد از تلفن سعی کنید رجیستر شوید و خطاها را ببینید.

---

### ❌ مشکل: "Not Acceptable Here" هنگام تماس

**علت:** SRTP در تلفن فعال نیست.

**راه حل:**
- در تلفن: **RTP Encryption** را روی **Optional** یا **Enabled** بگذارید

---

### ❌ مشکل: "SSL Connection Failed"

**علت:** مشکل در گواهی SSL

**بررسی:**
```bash
openssl s_client -connect 127.0.0.1:5061
```

**راه حل:**
- دوباره گواهی را بسازید (مرحله ۱)
- مطمئن شوید **Common Name** درست است

---

### ❌ مشکل: از داخل شبکه وصل می‌شود، از بیرون نه

**بررسی:**
1. Port Forwarding درست انجام شده؟
2. Firewall روتر باز است؟
3. آی‌پی عمومی ثابت است یا پویا؟

**راه حل:**
- از سرویس Dynamic DNS استفاده کنید (مثل No-IP, DynDNS)

---

## نکات امنیتی

### 🔒 ۱. رمز عبور قوی

- حداقل **16 کاراکتر**
- ترکیبی از حروف بزرگ، کوچک، اعداد و علائم
- مثال: `A1b!C2d@E3f#G4h$`

### 🔒 ۲. محدود کردن دسترسی به IP خاص (اختیاری)

اگر IP ثابت دارید:

```bash
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="37.255.228.83" port protocol="tcp" port="5061" accept'
firewall-cmd --reload
```

### 🔒 ۳. استفاده از VPN

بهترین روش:
- یک سرور VPN راه‌اندازی کنید (OpenVPN, WireGuard)
- تلفن‌ها از طریق VPN به سرور وصل شوند
- نیازی به Port Forwarding نیست

### 🔒 ۴. بررسی دوره‌ای

#### چک کردن IP های بلاک شده:

```bash
fail2ban-client status asterisk
```

#### چک کردن تلفن‌های متصل:

```bash
asterisk -rx "sip show peers"
```

#### چک کردن تماس‌های فعال:

```bash
asterisk -rx "core show channels"
```

### 🔒 ۵. آپدیت منظم

```bash
yum update -y
```

### 🔒 ۶. بکاپ

از فایل‌های پیکربندی بکاپ بگیرید:

```bash
tar -czf issabel-backup-$(date +%Y%m%d).tar.gz \
  /etc/asterisk/sip_general_custom.conf \
  /etc/asterisk/sip_additional.conf \
  /etc/asterisk/keys/ \
  /etc/fail2ban/jail.local
```

---

## خلاصه دستورات مهم

### بررسی وضعیت

```bash
# TLS فعال است؟
asterisk -rx "sip show settings" | grep -i tls

# تلفن‌های متصل
asterisk -rx "sip show peers"

# جزئیات یک Extension
asterisk -rx "sip show peer 102"

# تماس‌های فعال
asterisk -rx "core show channels"

# وضعیت Fail2Ban
fail2ban-client status asterisk

# بررسی Firewall
firewall-cmd --list-all

# بررسی پورت‌ها
netstat -tuln | grep -E '5061|80|443'
```

### ریلود و ریستارت

```bash
# ریلود Asterisk
asterisk -rx "sip reload"
asterisk -rx "core reload"

# ریستارت کامل Asterisk
systemctl restart asterisk

# ریستارت Apache
systemctl restart httpd

# ریستارت Fail2Ban
systemctl restart fail2ban

# ریلود Firewall
firewall-cmd --reload
```

---

## چک‌لیست نهایی

قبل از استفاده در محیط واقعی:

- [ ] گواهی SSL ساخته شد و اعتبار دارد
- [ ] TLS در Asterisk فعال است (پورت 5061)
- [ ] تنظیمات NAT انجام شد
- [ ] Firewall پیکربندی شد (پورت‌ها باز هستند)
- [ ] Fail2Ban فعال است و کار می‌کند
- [ ] Extension با `transport=tls` تنظیم شد
- [ ] رمز عبور Extension قوی است (16+ کاراکتر)
- [ ] تلفن با TLS و SRTP پیکربندی شد
- [ ] Port Forwarding در روتر انجام شد
- [ ] تست از داخل شبکه: ✅
- [ ] تست از بیرون شبکه: ✅
- [ ] تماس آزمایشی موفق بود: ✅
- [ ] بکاپ از فایل‌ها گرفته شد

---

## منابع و لینک‌های مفید

- [Asterisk TLS Documentation](https://wiki.asterisk.org/wiki/display/AST/Secure+Calling)
- [Issabel Official Documentation](https://www.issabel.org/documentation/)
- [Yealink Phone Configuration Guide](http://support.yealink.com/)
- [Fail2Ban Documentation](https://www.fail2ban.org/)

---

## پشتیبانی

در صورت بروز مشکل:
1. لاگ‌های Asterisk را بررسی کنید: `tail -f /var/log/asterisk/full`
2. به انجمن‌های Issabel/Asterisk مراجعه کنید
3. از بخش عیب‌یابی این آموزش استفاده کنید

---

**تاریخ آخرین بروزرسانی:** اکتبر 2025  
**نسخه:** 1.0

**موفق باشید!** 🎉🔒
