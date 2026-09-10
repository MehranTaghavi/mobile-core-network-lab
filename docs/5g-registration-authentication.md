# تحلیل فرآیند Registration و Authentication در شبکه 5G

بررسی و تشریح گام‌به‌گام فرآیند احراز هویت (5G-AKA) و تشکیل نشست PDU در یک شبکه 5G Standalone، بر اساس ترافیک واقعی Capture‌شده از اتصال یک UE به شبکه.

## روش کار

- شبکه هسته 5G (5GC) راه‌اندازی و آنتن (gNB) به آن متصل شد؛ فایل pcap مربوط به ترافیک شبکه در حین اتصال UE استخراج شد.
- تمام تحلیل‌های این سند بر پایه‌ی همین pcap واقعی و بررسی بسته‌ها در Wireshark انجام شده است.
- ارتباط رادیویی بین UE و gNB (لایه RRC) در این pcap موجود نیست، چون خارج از محدوده Capture سطح شبکه هسته است. برای توضیح این بخش صرفاً از نمودارهای مرجع [eventhelix.com](https://www.eventhelix.com) استفاده و بدون داده واقعی، صرفاً برای تکمیل تصویر کلی فرآیند اشاره شده است.
- تمرکز اصلی این سند بر پیام‌های **NGAP/NAS** (بین gNB و AMF) و **HTTP/2 مبتنی بر SBI** (بین Network Functionهای هسته) است.

## معماری و عناصر شبکه (Network Functions)

| NF | نقش | آدرس مشاهده‌شده در Capture |
|---|---|---|
| AMF | مدیریت Mobility و ارتباط NAS با UE | — |
| SCP | واسط مسیریابی درخواست‌های بین NFها (Service Communication Proxy) | — |
| NRF | کشف سرویس‌ها (Service Discovery) | 172.22.0.12 |
| AUSF | احراز هویت کاربر | 172.22.0.11:7777 |
| UDM | مدیریت داده‌های اشتراک کاربر | 172.22.0.13:7777 |
| UDR | ذخیره‌سازی داده (پشت UDM/PCF، مرتبط با MongoDB) | 172.22.0.14:7777 |
| PCF | سیاست‌گذاری QoS و Policy کاربر | — |
| SMF | مدیریت نشست PDU | — |
| UPF | مسیر داده کاربر (User Plane) | — |
| BSF | نگاشت Binding بین PCF و نشست کاربر | 172.22.0.29:7777 |

> نکته مهم: چون HTTP/2 (پروتکل ارتباطی SBI) ذاتاً **Stateless** است، بسیاری از اطلاعات (مثل مقادیر رمزنگاری یا شناسه‌های نشست) در چند پیام متوالی تکرار می‌شوند تا هر NF بتواند فرآیند را بدون وابستگی به وضعیت قبلی پردازش کند.

---

## ۱. آغاز Registration و دریافت هویت کاربر (SUCI)

- UE با ارسال یک `Registration Request` از طریق gNB به AMF (پروتکل NGAP، حامل پیام NAS) فرآیند ثبت‌نام را آغاز می‌کند. این پیام شامل اطلاعات موقعیت جغرافیایی، 5G-GUTI، TAI و ظرفیت‌های UE است.
- از آنجا که تمام ارتباطات AMF–UE از طریق پروتکل NAS و با واسطه gNB انجام می‌شود، هر پیام NGAP حامل یک بار NAS است.
- AMF با یک `Identity Request` هویت واقعی کاربر را درخواست می‌کند. UE با ارسال **SUCI** (شناسه رمزنگاری‌شده‌ی جهانی که IMSI واقعی را پنهان نگه می‌دارد) پاسخ می‌دهد.

## ۲. کشف AUSF و آغاز احراز هویت (Nausf_UEAuthentication)

- AMF برای احراز هویت کاربر باید با سرویس **Nausf_UEAuthentication** از AUSF صحبت کند، اما ابتدا باید آدرس آن را از **NRF** استعلام کند.
- این استعلام (Discovery) از طریق SCP انجام می‌شود: SCP از NRF می‌خواهد سرویس مدنظر را پیدا کند؛ NRF آدرس و پورت AUSF (`172.22.0.11:7777`) را بازمی‌گرداند.
- از این پس، AMF از طریق SCP و با آدرس مشخص‌شده با AUSF ارتباط برقرار می‌کند. مقدار SUCI کاربر نیز در این درخواست ارسال می‌شود.

## ۳. زنجیره دریافت بردار احراز هویت: AUSF → UDM → UDR

برای تولید بردار احراز هویت 5G-AKA، AUSF باید با UDM هماهنگ شود و UDM نیز داده‌های امنیتی را از UDR (که به یک پایگاه‌داده MongoDB متصل است) دریافت می‌کند:

1. **AUSF → UDM:** AUSF سرویس `Nudm-UEAuthentication` را (پس از کشف آدرس UDM از طریق NRF/SCP، آدرس `172.22.0.13:7777`) فراخوانی و SUCI و نام شبکه سرویس‌دهنده (`servingNetworkName`) را ارسال می‌کند.
2. **UDM → UDR:** UDM سرویس `Nudr-DataManagement` را از UDR (`172.22.0.14:7777`) درخواست می‌کند تا داده‌های امنیتی کاربر را دریافت کند. مقادیر کلیدی بازگردانده‌شده توسط UDR:
   ```
   encOpcKey: 22222222222222222222222222222222
   authenticationManagementField: 8000
   sqn: 000000021a2e
   ```
3. **UDM → AUSF:** UDM با استفاده از این داده‌ها، بردار احراز هویت 5G_HE_AKA را تولید و به AUSF بازمی‌گرداند:
   ```
   avType:    5G_HE_AKA
   rand:      7fef87e494279d5b277445423f7e21e7
   autn:      437942af283880009522c96ce411834c
   xresStar:  57c638364b5436a819d48549f71c009e
   kausf:     53d846c6a99bd65a698c32576ea9811099662ba588158880ea660756c2d83f5a
   imsi:      001010000000002
   ```
   (`xres*` امضای صحیحی است که کاربر باید با استفاده از کلید خودش روی `rand` تولید کند.)

## ۴. چالش-پاسخ (Challenge–Response) با UE

- مقادیر `rand`، `autn` و نسخه هش‌شده‌ی چالش (`hxresStar`) از طریق زنجیره AUSF → SCP → AMF به UE می‌رسد.
- AMF این مقادیر (به‌همراه `AMF-UE-NGAP-ID` و `RAN-UE-NGAP-ID`) را از طریق gNB و پروتکل NAS برای UE ارسال می‌کند.
- UE مقدار `RAND` را با کلید خود امضا کرده و پاسخ (`RES`) را بازمی‌گرداند:
  ```
  RES: 57c638364b5436a819d48549f71c009e
  ```
- این مقدار در مسیر معکوس (AMF → SCP → AUSF) به AUSF می‌رسد. AUSF با تطبیق `RES*` دریافتی با `hxresStar` قبلی، صحت هویت کاربر را تایید می‌کند (`PUT /nausf-auth/v1/ue-authentications/.../5g-aka-confirmation`).

## ۵. ثبت AMF و دریافت داده‌های اشتراک از UDM

پس از موفقیت احراز هویت، UDM باید AMF فعلی را به‌عنوان AMF سرویس‌دهنده کاربر ثبت کند و داده‌های اشتراک (Subscription Data) کاربر را در اختیار AMF بگذارد. این بخش شامل چند فراخوانی سرویس متوالی است:

- **`Nudm_UECM` (PUT):** AMF رکورد خودش را به‌عنوان AMF فعال کاربر در UDM ثبت می‌کند (`/nudm-uecm/v1/.../registrations/amf-3gpp-access`). UDM این ثبت را نیز در UDR ماندگار می‌کند.
- **`Nudm_SDM` (GET، چند بار):** AMF داده‌های اشتراک کاربر را از UDM می‌خواند — از جمله داده‌های Access & Mobility (`am-data`)، انتخاب SMF (`smf-select-data`) و اطلاعات SMF در Context فعلی (`ue-context-in-smf-data`). UDM خودش این داده‌ها را (در صورت نیاز) از UDR تامین می‌کند.
- **اشتراک در تغییرات (`sdm-subscriptions`, POST):** AMF برای اطلاع از تغییرات آینده در داده‌های کاربر، در UDM subscribe می‌کند تا در صورت تغییر، اعلان (`Nudm_SDM_Notification`) دریافت کند.

## ۶. سیاست دسترسی (AM Policy Control)

- AMF سرویس `Npcf-AMPolicyControl` را از PCF درخواست می‌کند (`POST /npcf-am-policy-control/v1/policies`) تا سیاست‌های مربوط به دسترسی و Mobility کاربر را دریافت کند.
- PCF برای این کار به UDR (`Nudr-DataRepository`, سرویس `policy-data`) مراجعه می‌کند تا اطلاعات مربوط به سیاست کاربر (شامل SNSSAI مجاز، محدودیت‌های AMBR و…) را بخواند و در پاسخ به AMF بازمی‌گرداند.
- پاسخ شامل یک شیء کامل JSON با اطلاعات SUPI، GPSI، نوع دسترسی (`3GPP_ACCESS`)، PEI، موقعیت مکانی (nrLocation شامل TAC و NCGI)، منطقه زمانی، PLMN سرویس‌دهنده، نرخ‌های AMBR کاربر (uplink/downlink) و SNSSAI مجاز است.

## ۷. برقراری امنیت NAS و بروزرسانی موقعیت مکانی

- AMF کلیدهای امنیتی (`SecurityKey`) و توانمندی‌های امنیتی UE (`UESecurityCapabilities`) را برای برقراری ارتباط رمزنگاری‌شده NAS به UE ارسال می‌کند (پیام `InitialContextSetupRequest`، شامل GUAMI، Masked IMEISV و NSSAI مجاز).
- UE با ارسال یک پیام تاییدیه، دریافت این اطلاعات را اعلام می‌کند.
- در ادامه، UE اطلاعات موقعیت مکانی خود (شامل `nrCellIdentity` و `Tracking Area Code`) را به AMF ارسال می‌کند تا در سیستم بروزرسانی شود. AMF با `DownlinkNASTransport` پاسخ می‌دهد.

---

## ۸. برقراری نشست PDU (PDU Session Establishment)

با تکمیل Registration، UE درخواست یک نشست داده (PDU Session) می‌دهد. این بخش پیچیده‌ترین زنجیره تعامل بین NFهاست:

1. **AMF → SMF (`Nsmf-PDUSession`, از طریق SCP):** AMF درخواست ایجاد نشست را برای SMF ارسال می‌کند؛ شامل اطلاعات کاربر (IMSI، IMEISV، MSISDN)، شناسه نشست (`pduSessionId`) و اطلاعات شبکه (AMF-ID، PLMN-ID، PCF-ID، و DNN=`internet`).
2. **SMF → NRF/UDM (کشف و دریافت داده اشتراک SM):** SMF از طریق NRF سرویس `nudm-sdm` را در UDM پیدا و داده‌های Session Management اشتراک کاربر (پروفایل‌های QoS شامل `5qi`، `priorityLevel`، ARP، نرخ‌های `uplink`/`downlink`) را دریافت می‌کند. UDM این داده‌ها را از UDR (پشت‌صحنه، از طریق MongoDB) می‌خواند.
3. **SMF → PCF (`Npcf-SMPolicyControl`):** SMF سیاست مدیریت نشست را از PCF درخواست می‌کند. PCF نیز:
   - سرویس `Nbsf-Management` را برای ثبت Binding نشست فراخوانی می‌کند (آدرس BSF از NRF کشف می‌شود: `172.22.0.29:7777`).
   - اطلاعات کاربر (IMSI، MSISDN، IP، DNN) را در BSF ثبت می‌کند.
   - سیاست نهایی (شامل QoS/AMBR کاربر) را به SMF بازمی‌گرداند.
4. **SMF → UPF (پروتکل PFCP):** SMF یک `PFCP Session Establishment Request` برای UPF ارسال می‌کند تا مسیر داده (User Plane) کاربر ایجاد شود؛ UPF با `PFCP Session Establishment Response` تایید می‌کند.
5. **SMF → AMF → UE:** SMF نتیجه (`PDU Session Establishment Accept`) را به AMF و از آنجا (به‌همراه `AggregateMaximumBitRate` برای uplink و downlink) از طریق gNB به UE می‌رساند.
6. **ثبت SMF در UDM:** SMF رکورد خودش را به‌عنوان SMF فعال این نشست، در UDM (سرویس `nudm-uecm`، مسیر `smf-registrations`) ثبت می‌کند.

## ۹. بروزرسانی QoS نشست (PDU Session Modification)

- برای اعمال کامل پارامترهای QoS، نشست ایجادشده یک بار دیگر بین SMF و UPF (پروتکل PFCP، `Session Modification Request/Response`) بروزرسانی می‌شود.
- سپس SMF اطلاعات نهایی نشست (شامل `pduSessionId`) را در UDM ثبت می‌کند (زنجیره مشابه بخش ۸، بار دیگر با تایید از UDR).
- در پایان این مرحله، AMF نیز نتیجه (`204 No Content`) را دریافت می‌کند که نشان‌دهنده تکمیل موفق بروزرسانی است.

## ۱۰. تایید نهایی: برقراری اتصال داده

- با تکمیل احراز هویت و نشست، ترافیک واقعی کاربر (از طریق UPF) آغاز می‌شود.
- به‌عنوان تایید نهایی عملکرد صحیح شبکه، درخواست Resolve نام دامنه `google.com` مشاهده و ترافیک QUIC/TCP/GTP کاربر به سمت DNS عمومی (`8.8.8.8`, `8.8.4.4`) capture شد که نشان‌دهنده اتصال کامل و موفق کاربر به اینترنت از طریق شبکه راه‌اندازی‌شده است.

---

## جمع‌بندی فنی

این تحلیل، چرخه کامل یک اتصال 5G Standalone را — از Registration اولیه، احراز هویت 5G-AKA، دریافت سیاست‌ها و داده‌های اشتراک، تا برقراری نشست PDU و اعتبارسنجی نهایی اتصال اینترنت — به‌صورت عملی و مبتنی بر بسته‌های واقعی شبکه پوشش می‌دهد و معماری Service-Based Architecture (SBA) شامل تعامل AMF، SCP، NRF، AUSF، UDM، UDR، PCF، SMF، UPF و BSF را به‌طور کامل نشان می‌دهد.
