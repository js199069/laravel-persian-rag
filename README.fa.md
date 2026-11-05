<div dir="rtl">

# Laravel Persian RAG

[![tests](https://github.com/js199069/laravel-persian-rag/actions/workflows/tests.yml/badge.svg)](https://github.com/js199069/laravel-persian-rag/actions/workflows/tests.yml)
[![Latest Version](https://img.shields.io/packagist/v/js199069/laravel-persian-rag.svg)](https://packagist.org/packages/js199069/laravel-persian-rag)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

**پاسخ به سؤال‌ها از روی اسناد فارسی سازمان، داخل برنامه‌ی لاراولی خودتان؛ با ارجاع به سند و صفحه، کنترل دسترسی بر اساس نقش، و بدون جواب ساختگی وقتی جواب در اسناد نیست.**

[English](README.md)

</div>

```php
$answer = app(Rag::class)->answer('یک واحد درس نظری چند جلسه است؟', role: $user->rag_role);

$answer['answer'];   // "یک واحد درس نظری طی ۱۶ جلسه یک ساعتی آموزش داده می‌شود [2]."
$answer['sources'];  // [2] => doc-04، «آیین‌نامه آموزشی ...»، صفحه‌ی ۸
$answer['refused'];  // false — وقتی جواب در اسناد نیست، true
```

<div dir="rtl">

- **اسناد واقعی فارسی، همان‌طور که هستند:** PDF متنی، PDF اسکن‌شده (با OCR)، DOCX؛ حتی با فونت خراب، شکل‌های نمایشی عربی، و لیگاچر «لا»ی برعکس.
- **جست‌وجوی ترکیبی** روی PostgreSQL: pgvector و جست‌وجوی متنی، ترکیب‌شده با RRF.
- **کنترل دسترسی داخل خود کوئری SQL**، نه در پرامپت. دانشجو هرگز سند مخصوص هیئت علمی را نمی‌بیند، هر طور که سؤال را بپرسد.
- **جواب با ارجاع** به سند و صفحه، امتناع دولایه، و تشخیص ارجاع به منبعی که وجود ندارد.
- **مستقل از ارائه‌دهنده:** embedding محلی (Ollama)؛ پاسخ از هر سرویس سازگار با OpenAI: مدل محلی، سرور GPU خودتان، یا سرویس‌های واسط داخلی. تعویض با یک متغیر محیطی.
- **سنجیده، نه ادعا:** دستورهای ارزیابی، ۴۱ تست خودکار (از جمله تست دسترسی روی PostgreSQL واقعی)، و CI روی PHP 8.3/8.4 و PostgreSQL 17/18.

---

## نتایج

**اسناد واقعی:** ۳۱ آیین‌نامه‌ی عمومی یک دانشگاه بزرگ ایرانی (۲۸ PDF و ۳ DOCX، حدود ۲۰۰ صفحه، ۵ گروه مخاطب). **۵۵٪ این اسناد فقط تصویر اسکن‌شده بودند، بدون هیچ لایه‌ی متنی.** خود اسناد منتشر نمی‌شوند.

| | |
|---|---|
| سند درست در رتبه‌ی اول / در ۵ نتیجه‌ی اول | ۹۵٪ / ۱۰۰٪ |
| صفحه‌ی درست | ۹۳٪ |
| جواب با کلمه‌ی کلیدی درست | ۱۰ از ۱۱ |
| ارجاع به سند درست | ۱۹ از ۲۰ |
| امتناع درست در سؤال‌های بی‌جواب | ۵ از ۵ |
| ارجاع به منبع ناموجود / نشت دسترسی | ۰ / ۰ |

**اسناد نمونه:** شش سند ساختگی در [`examples/sample-corpus`](examples/sample-corpus) که خودتان می‌توانید آزمایش کنید: سند درست در رتبه‌ی اول ۱۲ از ۱۲، جواب درست ۱۲ از ۱۲ (شامل سه شکلِ یک سؤال، و دو سؤال که فقط سند اسکن‌شده جوابشان را دارد)، امتناع درست ۳ از ۳، نشت صفر.

embedding با `bge-m3` روی CPU یک لپ‌تاپ ۲۰۱۶ (بدون GPU)؛ پاسخ با مدل Qwen 35B-A3B از طریق یک سرویس سازگار با OpenAI. جست‌وجو زیر نیم ثانیه، جواب کامل ۲ تا ۹ ثانیه.

> مدل محلی ۳ میلیاردی (Qwen2.5-3B) هم برای پاسخ امتحان شد: جست‌وجو برای همه‌ی سؤال‌ها سند درست را پیدا کرده بود، ولی مدل فقط برای ۱ سؤال از ۱۱ جواب درست داد. **جست‌وجو روی لپ‌تاپ خوب کار می‌کند؛ تولید پاسخ به مدل بزرگ‌تری نیاز دارد.**

جزئیات بیشتر در [نسخه‌ی انگلیسی](README.md#results).

---

## پیش‌نیازها

- PHP 8.3 به بالا با اکستنشن‌های `intl`، `mbstring`، `zip` و `pdo_pgsql`
- لاراول ۱۳
- PostgreSQL 17 یا 18 با [pgvector](https://github.com/pgvector/pgvector)
- [Ollama](https://ollama.com) با مدل `bge-m3` (برای embedding؛ **هنگام جست‌وجو باید روشن باشد**)
- یک سرویس سازگار با OpenAI برای پاسخ
- برای استخراج: [Poppler](https://poppler.freedesktop.org) (`pdftotext` و `pdftoppm`) و [Tesseract](https://github.com/tesseract-ocr/tesseract) با داده‌ی زبان فارسی (`fas`)

---

## نصب

**۱. نصب پکیج**

</div>

```bash
composer require js199069/laravel-persian-rag
```

<div dir="rtl">

**۲. فعال کردن pgvector:** یک بار، با کاربر superuser پستگرس، روی دیتابیس برنامه:

</div>

```sql
CREATE EXTENSION vector;
```

<div dir="rtl">

اگر این کار انجام نشده باشد، migration همین را با پیام روشن می‌گوید.

**۳. انتشار فایل‌های config**

</div>

```bash
php artisan vendor:publish --tag=persian-rag-config
php artisan vendor:publish --tag=ai-config
```

<div dir="rtl">

> از `--tag=ai-config` استفاده کنید، نه `--tag=ai`. دومی migrationهای خود laravel/ai را هم منتشر می‌کند، که این پکیج لازمشان ندارد.

**۴. تعریف دو provider** در `config/ai.php`، داخل `'providers' => [`:

</div>

```php
'ollama_local' => [
    'driver' => 'openai-compatible',
    'url' => env('OLLAMA_URL'),
    'models' => [
        'text' => ['default' => env('OLLAMA_CHAT_MODEL')],
        'embeddings' => ['default' => env('OLLAMA_EMBED_MODEL')],
    ],
],

'answers' => [
    'driver' => 'openai-compatible',
    'url' => env('RAG_ANSWER_URL'),
    'key' => env('RAG_ANSWER_KEY'),
    'models' => [
        'text' => ['default' => env('RAG_ANSWER_MODEL')],
    ],
],
```

<div dir="rtl">

> در `config/ai.php` از قبل یک provider به نام `ollama` هست. **از آن استفاده نکنید:** آدرس سفارشی نمی‌پذیرد. درایور `openai-compatible` با آدرس `/v1` اولاما کار می‌کند.

**۵. متغیرهای محیطی** (`.env`)

</div>

```ini
OLLAMA_URL=http://localhost:11434/v1
OLLAMA_CHAT_MODEL=qwen2.5:3b
OLLAMA_EMBED_MODEL=bge-m3

RAG_ANSWER_URL=https://your-openai-compatible-endpoint/v1
RAG_ANSWER_KEY=...
RAG_ANSWER_MODEL=...

RAG_LLM_PROVIDER=answers
```

<div dir="rtl">

`RAG_LLM_PROVIDER` نام provider است که جواب‌ها را می‌سازد. **اگر تنظیمش نکنید، پیش‌فرض `ollama_local` است**، یعنی مدل کوچک محلی، که برای پاسخ کافی نیست. سرویس‌های واسط داخلی که API سازگار با OpenAI دارند، با همین روش وصل می‌شوند.

**۶. migration و بررسی سلامت**

</div>

```bash
ollama pull bge-m3
php artisan migrate
php artisan config:clear
php artisan rag:health --llm
```

<div dir="rtl">

`rag:health` همه‌ی وابستگی‌ها را بررسی می‌کند و برای هر مشکل می‌گوید چه کار کنید: دیتابیس، pgvector، embedding، مدل پاسخ، ابزارهای OCR، و نقش‌ها.

---

## شروع سریع با اسناد نمونه

اسناد نمونه در مخزن GitHub هستند، نه در بسته‌ی Composer. مخزن را clone کنید (یا پوشه‌ی `examples/sample-corpus` را دانلود کنید)، بعد در برنامه‌ی لاراولی‌تان:

</div>

```bash
mkdir -p storage/app/corpus
cp /path/to/laravel-persian-rag/examples/sample-corpus/{*.pdf,*.docx,manifest.csv} storage/app/corpus/
cp /path/to/laravel-persian-rag/examples/sample-corpus/eval.csv storage/app/eval.csv

php artisan rag:extract
php artisan rag:quality
php artisan rag:refine
php artisan rag:index --fresh

php artisan rag:ask "یک واحد درس نظری چند جلسه است؟" --role=bachelor
php artisan rag:eval-answers
```

<div dir="rtl">

اسناد نمونه از نقش‌های پیش‌فرض پکیج استفاده می‌کنند، پس هیچ تنظیمی لازم نیست.

---

## ایندکس اسناد خودتان

فایل‌ها را در `storage/app/corpus/` بگذارید، همراه با یک `manifest.csv`:

</div>

```csv
id,file,original_name,audience,title
doc-01,doc-01.pdf,آیین‌نامه آموزشی,all_students,آیین‌نامه آموزشی دوره کارشناسی
doc-02,doc-02.pdf,آیین‌نامه تخلفات,faculty,آیین‌نامه رسیدگی به تخلفات اعضای هیئت علمی
```

<div dir="rtl">

- **`audience`** تعیین می‌کند چه کسی سند را می‌بیند.
- **`title`** اختیاری است ولی مهم: همراه هر تکه‌ی سند embed می‌شود. در ارزیابی ما، یک عنوان توصیفی یک سؤال را از نتیجه‌ی غلط به رتبه‌ی اول رساند. نام‌هایی مثل `scan003.pdf` به جست‌وجو آسیب می‌زنند.
- manifest می‌تواند UTF-8 یا Windows-1256 باشد، با یا بدون BOM، با جداکننده‌ی «,» یا «;» (اکسل).

بعد به ترتیب:

| دستور | کار |
|---|---|
| `rag:extract` | استخراج متن؛ برای PDFهای بدون لایه‌ی متنی، OCR خودکار |
| `rag:quality` | ساختن واژه‌نامه از خود اسناد و امتیاز کیفیت هر سند |
| `rag:refine` | انتخاب بهترین روش (متن یا OCR) برای هر صفحه؛ حذف محافظه‌کارانه‌ی نویز OCR |
| `rag:index` | تکه‌تکه کردن، embedding، ذخیره در PostgreSQL (`--fresh` برای ساخت از نو) |
| `rag:reindex-text` | بازسازی فقط ستون جست‌وجوی متنی (چند ثانیه، بدون embedding دوباره) |

**روی CPU زمان می‌برد:** OCR حدود ۳ تا ۶ ثانیه برای هر صفحه، و embedding حدود ۱٫۷ ثانیه برای هر تکه. ۲۰۰ صفحه روی یک لپ‌تاپ ۲۰۱۶ حدود نیم ساعت طول کشید.

> وقتی واژه‌نامه کمتر از ۴۰۰ کلمه دارد (مجموعه‌ی کوچک)، `rag:refine` حذف نویز را انجام نمی‌دهد، چون واژه‌نامه‌ی کوچک کلمه‌های واقعی را نویز می‌بیند. یک هشدار می‌بینید؛ این رفتار عمدی است.

---

## استفاده در کد

</div>

```php
use Js199069\PersianRag\Rag;

public function __construct(private Rag $rag) {}

// فقط جست‌وجو
$found = $this->rag->search($question, role: 'bachelor', k: 5);

// جواب کامل با ارجاع
$answer = $this->rag->answer($question, role: $request->user()->rag_role);

// برای وب: اول منبع‌ها، بعد استریم جواب
$prepared = $this->rag->prepare($question, role: $request->user()->rag_role);
// $prepared['sources'] بلافاصله آماده است (زیر یک ثانیه)
abort_if($prepared['refused'], 422);
return $this->rag->stream($prepared)->usingVercelDataProtocol();
```

<div dir="rtl">

**نقش باید از کاربرِ واردشده خوانده شود**، هرگز از ورودی درخواست؛ وگرنه هر کسی می‌تواند خودش را «هیئت علمی» معرفی کند. نقش ناشناخته `InvalidArgumentException` می‌دهد. `role: null` یعنی بدون فیلتر، و فقط برای ابزارهای مدیریتی است.

از خط فرمان: `php artisan rag:search` و `php artisan rag:ask` (هر دو `--role` و `--file` با یک سؤال در هر خط را می‌پذیرند).

یک دموی کامل وب، با استریم و ارجاع‌های قابل کلیک: [laravel-persian-rag-demo](https://github.com/js199069/laravel-persian-rag-demo)

---

## کنترل دسترسی

هر سند یک `audience` دارد؛ هر نقش فهرستی از مخاطب‌هایی که اجازه دارد ببیند. در `config/persian-rag.php`:

</div>

```php
'roles' => [
    'bachelor' => ['bachelor', 'all_students'],
    'phd'      => ['phd', 'all_students'],
    'faculty'  => ['faculty', 'phd', 'bachelor', 'all_students'],
    'staff'    => ['staff', 'phd', 'bachelor', 'all_students'],
],
```

<div dir="rtl">

**این نقش‌های پیش‌فرض، مثالی از یک دانشگاه‌اند.** با نقش‌های سازمان خودتان جایگزینشان کنید. `audience_labels` (نام خوانای هر مخاطب، که همراه هر تکه embed می‌شود) را هم به‌روز کنید، و بعد ایندکس را با `rag:index --fresh` از نو بسازید.

سندی که مخاطبش در هیچ نقشی نیست، برای **هیچ‌کس** دیده نمی‌شود. `rag:index` و `rag:health` هر دو درباره‌اش هشدار می‌دهند.

---

## تنظیم آستانه‌ی «جواب نده»

امتناع در دو لایه انجام می‌شود:

۱. **جست‌وجو:** اگر بیشترین شباهت کمتر از `min_similarity` باشد (پیش‌فرض `0.50`)، مدل اصلاً صدا زده نمی‌شود. این لایه فقط سؤال‌های **کاملاً بی‌ربط** را می‌گیرد.
۲. **مدل:** منبع‌ها را می‌خواند و وقتی جواب در آن‌ها نیست، می‌گوید. **این دفاع اصلی است.**

شباهت به‌تنهایی سؤال جواب‌دار را از بی‌جواب جدا نمی‌کند: در ارزیابی ما، سؤال‌های بی‌جوابِ هم‌موضوع تا ۰٫۶۵۰ شباهت گرفتند، در حالی که سؤال‌های جواب‌دار از ۰٫۵۹۹ شروع می‌شدند. **آستانه را روی داده‌ی خودتان تنظیم کنید:**

</div>

```bash
php artisan rag:eval
```

<div dir="rtl">

این دستور آستانه‌ی فعلی، تعداد سؤال‌های جواب‌داری که اشتباهاً رد می‌کند، و جدولی از مقدارهای دیگر را نشان می‌دهد. آستانه را طوری بگذارید که «false refusals» صفر باشد. آستانه‌ی ۰٫۵۵ که روی یک مجموعه تنظیم شده بود، روی مجموعه‌ی دیگری یک جواب درست را رد کرد؛ برای همین پیش‌فرض ۰٫۵۰ است.

---

## ارزیابی

سؤال‌ها را در `storage/app/eval.csv` بنویسید:

</div>

```csv
id,question,role,expected_doc,expected_page,must_not_doc,expected_answer
1,یک واحد درس نظری چند جلسه است؟,bachelor,doc-04,8,,16|۱۶
2,حقوق هیئت علمی چه روزی پرداخت می‌شود؟,faculty,,,,
3,مصرف مواد مخدر توسط عضو هیئت علمی چه تخلفی است؟,bachelor,,,doc-08,
```

<div dir="rtl">

- `expected_doc` خالی ← سؤال بی‌جواب (باید رد شود).
- `must_not_doc` پر ← تست دسترسی (آن سند هرگز نباید بیاید).
- چند سند درست: `doc-04|doc-12`؛ چند کلمه‌ی کلیدی قابل قبول: `16|۱۶`.

`rag:eval` جست‌وجو را می‌سنجد (قطعی، بدون مدل زبانی)؛ `rag:eval-answers` جواب‌ها را (بعد از هر سؤال ذخیره می‌کند و اگر یکی شکست خورد، ادامه می‌دهد).

**یک سؤال را به چند شکل بپرسید.** یک باگ رتبه‌بندی در همین پکیج، تا وقتی شکل کوتاه‌تری از یک سؤال موجود پرسیده نشد، دیده نمی‌شد.

---

## چرا این معماری؟

هر تصمیم زیر از چیزی آمده که اول شکست خورد. توضیح کامل در [نسخه‌ی انگلیسی](README.md#why-this-architecture).

- **OCR اختیاری نیست:** ۱۶ سند از ۳۱ سند یک دانشگاه بزرگ اصلاً لایه‌ی متنی نداشتند.
- **حذف نویز محافظه‌کارانه است:** یک فیلتر سخت‌گیر متن واقعی را حذف کرد («تهیه یا مصرف مواد مخدر»). قاعده: حذف اشتباه متن واقعی بدتر از نگه داشتن نویز است.
- **جست‌وجوی ترکیبی، با تضمین بهترین تکه‌ی معنایی:** سقف «تعداد تکه از هر سند» یک بار مرتبط‌ترین تکه را حذف کرد. حالا تکه‌ای که آستانه بر اساسش تصمیم می‌گیرد، همیشه به مدل می‌رسد.
- **دسترسی در SQL، نه در پرامپت:** مدل را می‌شود فریب داد، `WHERE` را نه. تست‌های دسترسی با خاموش کردن عمدی فیلتر آزموده شدند؛ دقیقاً همان سه تستی که باید شکست می‌خوردند، شکست خوردند.
- **embedding محلی، مدل پاسخ قابل تعویض:** اسناد کامل از سرور خارج نمی‌شوند؛ فقط ۳ تکه‌ی پیدا شده برای مدل پاسخ فرستاده می‌شود، که آن هم می‌تواند محلی باشد.
- **تغییرها سنجیده می‌شوند، و اگر کمک نکنند برگردانده می‌شوند:** جدا کردن رقم از کلمه («16جلسه») منطقی به نظر می‌رسید، ولی دقت رتبه‌ی اول را از ۹۵٪ به ۹۰٪ رساند؛ برگردانده شد.

---

## محدودیت‌ها

- رتبه‌بندی جست‌وجوی متنی، وزن کلمه‌های کمیاب را درست نمی‌فهمد.
- ساختار جدول در PDF اسکن‌شده از بین می‌رود.
- کلمه‌ی لاتین وسط خط فارسی ممکن است در `pdftotext` جابه‌جا شود و در OCR (با `fas` تنها) خراب شود.
- واژه‌نامه‌ی کیفیت به اندازه‌ی مجموعه بستگی دارد؛ در مجموعه‌ی کوچک، حذف نویز انجام نمی‌شود.
- **Ollama باید برای جست‌وجو روشن باشد**، چون خود سؤال هم محلی embed می‌شود.
- پاسخ‌های استریم‌شده در خطای سرویس دوباره تلاش نمی‌کنند (پاسخ‌های عادی دو بار تلاش می‌کنند).
- `rag:quality` و `rag:search` هنوز مسیر پیش‌فرض اسناد (`storage/app/corpus`) را فرض می‌کنند.
- مجموعه‌های ارزیابی کوچک‌اند (۳۰ و ۱۸ سؤال): عددها جهت را نشان می‌دهند، نه دقت قطعی.

---

## تست‌ها

</div>

```bash
composer install
cp phpunit.xml.dist phpunit.xml   # رمز RAG_TEST_DB_PASSWORD را تنظیم کنید
vendor/bin/phpunit
```

<div dir="rtl">

تست‌های یکپارچه یک دیتابیس PostgreSQL با pgvector لازم دارند که **نامش به `_test` ختم شود**. این تست‌ها جدول‌ها را پاک می‌کنند، و روی هر دیتابیس دیگری از اجرا خودداری می‌کنند.

---

## مجوز

MIT. فایل [LICENSE](LICENSE) را ببینید.

</div>