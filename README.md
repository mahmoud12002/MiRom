# MiROM — أداة تعديل رومات وكيرنيل شاومي (Linux / Termux)

أداة سطر أوامر بـ Python، بتشتغل **من غير روت**، ومعاها قائمة تفاعلية (`mirom menu`).
واجهة الأداة بالإنجليزي لأن التيرمنال بيعرض العربي مقطّع.

## ⚠️ اقرأ قبل التفليش
- لازم **البوتلودر يكون مفتوح**. متقفلوش أبدًا والروم معدّل.
- خد باك أب لبياناتك واحتفظ بالروم الأصلي (خصوصًا `boot.img` و `vbmeta.img`).
- اتأكد إن الروم لنفس الـ codename بتاع جهازك، وانتبه لحماية الـ anti-rollback: متنزلش لإصدار أقدم.
- أي قسم تعدّله هيفشل في Android Verified Boot، فلازم `--disable-vbmeta`. بعض الأجهزة محتاجة تمسح الداتا بعدها.

## التثبيت
```bash
unzip mirom.zip && cd mirom
bash install.sh
mirom doctor
```
**تيرمكس:** اشتغل جوه `$HOME` مش `/sdcard` (الذاكرة المشتركة مبتدعمش الروابط ولا الصلاحيات).
```bash
termux-setup-storage
cp ~/storage/downloads/rom.tgz ~/
```
المساحة: سيب فاضي حوالي 4 أضعاف حجم الروم بعد فكه.

## المدعوم
| النوع | الفتح | الإخراج |
|---|---|---|
| روم فاستبوت `.tgz` أو فولدر | ✔ | نفس الهيكل + `super.img` جديد |
| `payload.bin` أو OTA zip كامل | ✔ | فولدر صور + `flash.sh` |
| zip فيه `new.dat(.br)` | ✔ | zip غير موقّع (لريكفري مخصص) |
| `super.img` / sparse / صورة قسم | ✔ | ✔ |
| ext4 | قراءة وتعديل | يحافظ على الصلاحيات و SELinux |
| EROFS | قراءة | إعادة بناء (erofs-utils 1.7 أو أحدث) |
| boot / init_boot / recovery v0–v4، vendor_boot v3–v4 | ✔ | ✔ |

**غير مدعوم:** OTA التزايدي (incremental)، f2fs، أقسام super المقسومة على أكتر من جهاز، توقيع الرومات،
تعديل الـ APK/JAR (استخدم apktool)، وملفات الريكفري المشفرة. ولو المصدر `super` على هيئة sparsechunks
الأداة بتفكه، لكن التجميع في `pack` محتاج تعدّل سكربت التفليش بإيدك.

## سير العمل
```bash
mirom unpack -p myrom rom.tgz          # فك الروم في مشروع
mirom list -p myrom                    # الأقسام
mirom extract -p myrom system product  # فك ملفات الأقسام

# عدّل الملفات مباشرة في myrom/fs/<partition>/
mirom apps -p myrom                    # التطبيقات وأحجامها
mirom debloat -p myrom lists/debloat-example.txt --dry-run
mirom debloat -p myrom lists/debloat-example.txt
mirom prop -p myrom get ro.product.system.device
mirom prop -p myrom set persist.sys.example 1 --part system
mirom add-app -p myrom product app.apk MyApp

mirom build -p myrom system product --shrink
mirom pack  -p myrom -o output --disable-vbmeta --tgz
```
- **الصلاحيات:** محفوظة في `myrom/meta/<partition>.json`. الملف الجديد بياخد المالك و SELinux من الملفات المجاورة.
  القيمة `mode` في الملف مكتوبة بالنظام العشري (مثلاً `33188` = ملف `0644`).
- `--shrink` بيصغّر صورة ext4 لأقل حجم. من غيرها الأداة بتضيف مساحة فاضية (`--extra-mb`).
- لو ظهر `group ... overflows` يبقى الأقسام أكبر من سعة super: امسح تطبيقات أو استخدم `--shrink`.
- EROFS: `--compression lz4hc` هو الافتراضي، والحجم الناتج ممكن يختلف عن الأصلي.
- قبل تشغيل `flash.sh` (رومات payload) راجعه، لأنه بيشمل أقسام الفيرموير. امسح سطور اللي متغيرش.
- رومات فاستبوت: متستخدمش سكربت `flash_all_lock`.

## البوت والـ ramdisk
```bash
mirom boot unpack myrom/rom/<...>/images/vendor_boot.img -o vb
mirom fstab-noavb vb                   # شيل فحص AVB من fstab
# عدّل ملفات vb/ramdisk*/ أو سطر cmdline في vb/boot.json
mirom boot repack vb -o vendor_boot-new.img
mirom replace -p myrom vendor_boot vendor_boot-new.img
```
ملفات fstab اللي جوه أقسام `vendor` تتعدل بـ `mirom fstab-noavb myrom/fs/vendor_a`.

## الكيرنيل
```bash
mirom kernel info boot.img --config stock.config
mirom kernel replace boot.img Image -o new-boot.img
fastboot boot new-boot.img             # تجربة مؤقتة (لو جهازك بيدعمها)
```
**بناء كيرنيل من سورس شاومي** (على لينكس x86_64، مش مباشرة على تيرمكس):
```bash
# اسم الفرع من صفحة MiCode/Xiaomi_Kernel_OpenSource على GitHub، حسب جهازك
mirom kernel clone <branch> ksrc
mirom kernel build ksrc --defconfig <device_defconfig> --menuconfig
mirom kernel build ksrc --defconfig <device_defconfig> --enable CONFIG_KSM
```
محتاج clang و lld و llvm و cross binutils و bc و bison و flex و libssl-dev و libelf-dev.
الأمر ده بيغطي البناء القياسي بـ `make`. الأجهزة الأحدث (GKI 5.15 وما بعده) غالبًا بتتبني بـ Bazel/Kleaf
وسكربتات خاصة، ومش مغطاة هنا. ومش كل جهاز السورس بتاعه كامل.

**زيب AnyKernel3:**
```bash
git clone https://github.com/osm0sis/AnyKernel3
# عدّل BLOCK و IS_SLOT_DEVICE في anykernel.sh حسب جهازك
mirom kernel anykernel AnyKernel3 ksrc/out/arch/arm64/boot/Image --device <codename> -o kernel.zip
```

## أوامر منفردة
```bash
mirom info FILE
mirom payload ota.zip -o out --part boot
mirom sparse unpack super.img -o super.raw
mirom unsuper super.raw -o parts
mirom sdat system.transfer.list system.new.dat.br -o system.img
mirom vbmeta vbmeta.img -o vbmeta-disabled.img
```

## حالة الاختبار
```bash
python3 -m pytest tests      # محتاج pytest و e2fsprogs
```
- **متختبر على لينكس** بملفات تجريبية متولدة (مش روم شاومي حقيقي): sparse، super، payload، new.dat غير المضغوط،
  boot/vendor_boot، cpio، تعديل ext4 (مع `e2fsck`)، دورة روم فاستبوت كاملة، و LZ4 (مقارنة بمكتبة liblz4).
- **لسه متختبرش:** بناء EROFS وفك المضغوط منه (القارئ اتجرب على صورة مصنوعة يدويًا بس)، brotli، zstd،
  بناء الكيرنيل، تيرمكس، صور ext4 بخاصية `shared_blocks`، والتفليش على جهاز.

لو حصل خطأ: `MIROM_DEBUG=1 mirom ...` بيطلع التفاصيل.