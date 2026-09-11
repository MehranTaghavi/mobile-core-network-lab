# راه‌اندازی شبکه موبایل از صفر: از 4G RAN/Core تا VoLTE و 5G

پیاده‌سازی عملی یک شبکه موبایل کامل با استفاده از ابزارهای متن‌باز — از راه‌اندازی لایه دسترسی و هسته شبکه تا VoLTE و تحلیل 5G.

---

## مسئله

راه‌اندازی یک شبکه موبایل واقعی معمولاً پشت درهای بسته اپراتورها و با تجهیزات اختصاصی و گران‌قیمت انجام می‌شود؛ در نتیجه، دسترسی عملی برای یادگیری عمیق معماری شبکه‌های سلولار محدود است.

## راه‌حل

پروژه در سه فاز پیوسته پیش رفت، که هر فاز روی زیرساخت فاز قبل ساخته شد:

1. **راه‌اندازی شبکه 4G (RAN تا Core):** پیاده‌سازی EPC و eNB با استفاده از srsRAN و open5GS، و اتصال یک UE واقعی از طریق SDR.
2. **افزودن VoLTE با IMS:** اتصال یک لایه IMS مبتنی بر Kamailio به هسته شبکه برای برقراری تماس صوتی بین دو کاربر روی شبکه داده.
3. **گسترش به 5G:** تحلیل کامل فرآیندهای Registration و 5G-AKA Authentication، و برقراری PDU Session، بر اساس Capture واقعی ترافیک شبکه.

## معماری

```
UE  ──(radio / USRP B210)──►  eNB (srsRAN)  ──►  EPC (open5GS)  ──►  Internet
                                                      │
                                                      ▼
                                              IMS (Kamailio) ──► VoLTE Call
```

### اجزای اصلی

| جزء     | نقش                                                         | ابزار                                                        |
| ---------- | -------------------------------------------------------------- | ----------------------------------------------------------------- |
| eNB        | ایستگاه پایه، لایه دسترسی رادیویی | [srsRAN](https://docs.srsran.com/projects/4g/en/latest/index.html) |
| EPC        | هسته شبکه 4G (MME, HSS, SGW, PGW)                      | [open5GS](https://open5gs.org/open5gs/docs/)                       |
| SDR        | ارتباط رادیویی بین UE و eNB                   | USRP B210                                                         |
| IMS        | زیرساخت VoLTE                                           | Kamailio                                                          |
| Deployment | Build و پیکربندی خودکار سرویس‌ها        | [docker_open5gs](https://github.com/herlesupreeth/docker_open5gs)  |

---

## فاز ۱: راه‌اندازی شبکه 4G

### پیاده‌سازی

- EPC با استفاده از open5GS و eNB با استفاده از srsRAN راه‌اندازی شد.
- به‌جای Build دستی از Source، از اسکریپت‌های خودکار پروژه docker_open5gs برای Build و Deploy استفاده شد.
- در پیکربندی eNB، مدل SDR (USRP B210) تنظیم شد تا ارتباط رادیویی با UE برقرار شود.
- اطلاعات هویتی UE (IMSI، Key، OPC) در HSS ثبت شد.

### اعتبارسنجی

- اتصال UE به eNB و EPC برقرار و عملکرد اتصال به اینترنت روی UE بررسی شد.
- سیگنالینگ بین اجزای شبکه Capture شد تا فرآیندهای زیر تحلیل شوند:
  - Authentication (احراز هویت UE)
  - Session Establishment (تشکیل Session داده)
  - اتصال به اینترنت (PDN Connectivity)

📄 مستند کامل: [`docs/4g-network-signaling.md`](docs/4g-network-signaling.md)

> برای تحلیل عمیق‌تر پروتکل‌ها پیش از دسترسی به سخت‌افزار واقعی، از pcapهای عمومی موجود (مانند [نمونه open5GS](https://open5gs.org/open5gs/docs/tutorial/01-your-first-lte/)) نیز استفاده شده است.

---

## فاز ۲: VoLTE با IMS (Kamailio)

### پیاده‌سازی

- یک لایه IMS مبتنی بر **Kamailio** به هسته شبکه 4G فاز ۱ متصل شد.
- Deployment از طریق همان پروژه docker_open5gs انجام شد (که پشتیبانی از IMS را نیز فراهم می‌کند).
- هدف: برقراری یک تماس صوتی (VoLTE) کامل میان دو کاربر روی زیرساخت داده‌ای که در فاز ۱ ساخته شد.

📄 مستند کامل: [`docs/volte-ims-call-setup.md`](docs/volte-ims-call-setup.md)

---

## فاز ۳: تحلیل Registration و Authentication در شبکه 5G

### پیاده‌سازی و تحلیل

- شبکه هسته 5G راه‌اندازی و آنتن (gNB) به آن متصل شد؛ فایل pcap واقعی ترافیک شبکه در حین اتصال UE استخراج و در Wireshark تحلیل شد.
- فرآیند کامل **Registration**، **5G-AKA Authentication**، دریافت سیاست‌ها و داده‌های اشتراک از UDM/UDR/PCF، و **PDU Session Establishment** (شامل تخصیص IP و ایجاد GTP-U Tunnel) مرحله‌به‌مرحله بررسی شد.
- به‌عنوان تایید نهایی عملکرد شبکه، ترافیک داده واقعی کاربر (Resolve یک دامنه عمومی) capture و بررسی شد.

📄 مستند کامل: [`docs/5g-registration-authentication.md`](docs/5g-registration-authentication.md)

---

## نتایج و یافته‌ها

- شبکه 4G به‌طور کامل و end-to-end از UE تا اینترنت راه‌اندازی و تست شد؛ فرآیندهای Authentication، Session Establishment و PDN Connectivity در Traceهای واقعی تایید شدند.
- زیرساخت IMS برای VoLTE روی هسته 4G موجود پیاده‌سازی و جریان سیگنالینگ ثبت‌نام و برقراری تماس (SIP/IMS) مستند شد.
- فرآیند کامل Registration و 5G-AKA Authentication در شبکه 5G، بر اساس Capture واقعی، تحلیل و مستند شد؛ از کشف سرویس‌ها (NRF/SCP) و احراز هویت تا ایجاد Session داده.

## دانش و مهارت‌های به‌کاررفته

شبکه‌های موبایل (4G/LTE, EPC, IMS)، SDR و ارتباطات رادیویی، تحلیل سیگنالینگ و پروتکل‌های شبکه، Docker و ابزارهای Deployment، تحلیل ترافیک با Wireshark.

## 🛠️ پیش‌نیازها و اجرا

<<<<<<< HEAD
### پیش‌نیازها

- Docker و Docker Compose
- درایورهای UHD برای USRP B210
- Wireshark

### مراحل راه‌اندازی

1. کلون‌کردن پروژه `docker_open5gs`:

```bash
git clone https://github.com/herlesupreeth/docker_open5gs.git
cd docker_open5gs
```

2. کلون‌کردن این ریپازیتوری و ورود به آن:

```bash
git clone https://github.com/MehranTaghavi/mobile-core-network-lab.git
cd mobile-core-network-lab
```

3. اعمال کانفیگ‌های موجود در پوشه `configs/` روی محیط `docker_open5gs` (مسیر مقصد را متناسب با ساختار همان پروژه تنظیم کنید):

```bash
# از مسیر mobile-core-network-lab
cp -r configs/* ../docker_open5gs/
```

> در صورت نیاز، پارامترهای مربوط به `PLMN`، `TAC`، `APN`، `IMSI/Key/OPC` و IPها را با سناریوی خود هماهنگ کنید.

### شناسایی سخت‌افزار

برای تست شناسایی SDR (USRP B210):

```bash
uhd_find_devices
```

اگر دستگاه درست شناسایی شده باشد، اطلاعات B210 در خروجی نمایش داده می‌شود.

### اجرا و تست

1. اجرای سرویس‌های Open5GS/srsRAN:

```bash
cd ../docker_open5gs
docker compose up -d
```

2. بررسی وضعیت کانتینرها:

```bash
docker compose ps
```

3. اجرای Kamailio برای VoLTE (در صورت اجرا روی میزبان):

```bash
kamailio -f /etc/kamailio/kamailio.cfg -DD -E
```

اگر Kamailio به‌صورت کانتینری تعریف شده باشد، آن را از طریق سرویس مربوطه در `docker-compose` بالا بیاورید.

4. تحلیل ترافیک با Wireshark:

- اینترفیس‌های مرتبط با RAN/Core را Capture کنید.
- برای عیب‌یابی، پیام‌های `SIP`، `GTP`، `Diameter` و `NAS` را بررسی کنید.
=======
این پروژه روی سیستم‌عامل Ubuntu Linux پیاده‌سازی و تست شده است.

### پیش‌نیازهای نرم‌افزاری و سخت‌افزاری

- Ubuntu Linux با دسترسی Root
- `Docker` و `Docker Compose`
- درایورهای USRP (پکیج `uhd-host` / `uhd-utils`) برای ارتباط با SDR
- یک دستگاه SDR مدل **USRP B210** به‌همراه آنتن مناسب
- یک سیم‌کارت قابل‌برنامه‌ریزی (Programmable/Test SIM) با مقادیر IMSI، Key و OPC مطابق آنچه در HSS ثبت می‌شود
- یک گوشی موبایل واقعی به‌عنوان UE (برای تست اتصال داده و تماس VoLTE)

### راه‌اندازی هسته شبکه (open5GS و IMS)

برای استقرار سریع تمام توابع شبکه (MME, HSS, SGW, PGW و Kamailio) از کانتینرهای Docker پروژه [docker_open5gs](https://github.com/herlesupreeth/docker_open5gs) استفاده شد:

```bash
# کلون کردن مخزن
git clone https://github.com/herlesupreeth/docker_open5gs.git
cd docker_open5gs

# بیلد ایمیج‌ها (ممکن است زمان‌بر باشد)
docker-compose build

# اجرای هسته شبکه در پس‌زمینه
docker-compose up -d
```

> اتصال نهایی و تست عملی (ثبت‌نام UE، اتصال داده، و برقراری تماس VoLTE با استفاده از یک گوشی واقعی) به‌صورت عملی روی سخت‌افزار انجام شد؛ مستندسازی گام‌به‌گام این بخش (برخلاف تحلیل سیگنالینگ در `docs/`) ثبت نشده و این راهنما صرفاً چارچوب کلی راه‌اندازی محیط را پوشش می‌دهد.
>>>>>>> 8a70557 (docs: add prerequisites and execution section)

---

## نویسنده

**مهران تقوی افخم**
دانشجوی کارشناسی ارشد مهندسی کامپیوتر (معماری کامپیوتر)، دانشگاه صنعتی شریف

---

## ساختار ریپازیتوری

```
.
├── README.md
├── docs/
│   ├── project-statement.md               # صورت‌مسئله کامل پروژه
│   ├── 4g-network-signaling.md            # فاز ۱: تحلیل سیگنالینگ 4G
│   ├── volte-ims-call-setup.md            # فاز ۲: تحلیل VoLTE/IMS
│   └── 5g-registration-authentication.md  # فاز ۳: تحلیل Registration/Authentication در 5G
├── configs/                      # فایل‌های پیکربندی eNB / EPC / IMS
└── captures/                     # فایل‌های pcap تحلیل‌شده
```