# Bot kodi tahlili va Railway'ga joylash qo'llanmasi

## 1. Nima tuzatildi (`public/bot.php`)

| # | Muammo | Qayerda | Nega xavfli edi |
|---|--------|---------|------------------|
| 1 | **Tirnoqsiz array kalitlar** (`inline_keyboard=>`, `callback_data=>`, `chat_id=>`, `url=>`, `show_alert=>`, `from_chat_id=>`) — jami ~50 joyda | Butun fayl bo'ylab | PHP 8+ da bular "aniqlanmagan konstanta" deb hisoblanadi va **Fatal Error** beradi. Railway'da standart PHP 8.2/8.3 ishlatiladi — bot birinchi so'rovdayoq yiqiladi edi. |
| 2 | `bot(getMe)` | 18-qator | Xuddi shu sabab — `getMe` tirnoqsiz yozilgan edi |
| 3 | `'parse_mode'=>html` | 2 joyda | Xuddi shu sabab |
| 4 | `php://input` ikki marta o'qilishi | ~256-258 qator | `php://input` ni ikkinchi marta o'qishga urinish ba'zi serverlarda **bo'sh natija** qaytaradi → `$message` `null` bo'lib qoladi → `$message->message_id` chaqirilganda **"Attempt to read property on null"** xatosi bilan bot yiqiladi. Bu qator olib tashlandi. |
| 5 | `$step = file_get_contents("step/$from_id.step")` | 255-qator | `$from_id` bu yerda hali aniqlanmagan edi (pastda, 287-qatorda aniqlanadi) — bu qator PHP Warning chiqarardi va baribir keyinroq qayta yoziladi. Butunlay olib tashlandi. |
| 6 | `mkdir("user"); mkdir("set");` | ~370-qator | Papka mavjud bo'lsa ham har safar `mkdir` chaqirilar edi → har bir xabarga **Warning**. Endi `is_dir()` bilan tekshiriladi. |
| 7 | Bo'sh tugma matni `'text'=>""` | "❌ Yo'q" tugmasida, 2 joyda | Foydalanuvchiga bo'sh tugma ko'rinardi. `"❌ Yo'q"` matni qo'shildi. |
| 8 | Qattiq yozilgan tokenlar (`API_KEY`, `admin`, `simkey`, `channel`) | Fayl boshi | Endi `getenv()` orqali Railway Environment Variables'dan olinadi (pastga qarang). |
| 9 | Xatolar ekranga chiqishi | Fayl boshi | `display_errors` o'chirildi, `log_errors` yoqildi — production uchun xavfsiz. |

## 2. TUZATILMAGAN, lekin JIDDIY muammolar (qaror sizga bog'liq)

Bularni avtomatik tuzatmadim, chunki har biri **biznes-logikaga oid qaror** talab qiladi:

### a) SQL Injection — ~170 ta joyda
Deyarli barcha `mysqli_query($connect, "... WHERE id = $cid ...")` ko'rinishidagi so'rovlar foydalanuvchi kiritgan qiymatni **to'g'ridan-to'g'ri** SQL ichiga qo'yadi. Masalan:
```php
mysqli_query($connect,"SELECT * FROM `users` WHERE id = '$chat_id'");
```
Telegram `chat_id` odatda raqam bo'lgani uchun xavf past, lekin `$text`, `$data`, `$tx` kabi foydalanuvchi matni SQL ichiga tushadigan joylar (masalan promokod, admin xabar matni, va h.k.) bor va ular orqali **butun bazani o'chirish/o'g'irlash** mumkin.
**Tavsiya:** kamida foydalanuvchi matni ishtirok etadigan so'rovlarni `mysqli_real_escape_string()` yoki `mysqli_prepare()`/`bind_param()` ga o'tkazish kerak. Bu alohida, katta ish — xohlasangiz shu bilan ham yordam beraman.

### b) Ikki marta takrorlangan "➕ Yangi bot qo'shish" bloki
`bot_fixed.php`da ~4960—5364-qatorlar oralig'ida **bir xil funksionallik ikki marta** yozilgan (narxlari boshqacha: biri 45000, ikkinchisi 30000 so'm). `stripos($data,"botopen=")` sharti ikkalasida ham ishlaydi, ya'ni foydalanuvchi "✅ Tanlash" tugmasini bossa, **hisobidan pul ikki marta yechilishi** yoki bot ikki marta yaratilishga urinishi mumkin.
**Qaror kerak:** qaysi narx (45000 yoki 30000) to'g'ri? Bittasini o'chirib tashlashim mumkin.

### c) "Reseller / domen ochish" funksiyasi umuman ishlamaydi
Shu blok ichida chaqirilayotgan `url_query()`, `generatemysql()`, `plusmysql()` funksiyalari va `$isp_user`, `$acc` o'zgaruvchilari **faylning hech qayerida aniqlanmagan**. Bu — asl muallifning **shaxsiy hosting-panelga** (`ispsystem.sysdc.uz`) ulanadigan, xususiy infratuzilmaga bog'liq qism bo'lib, u sizga baribir ishlamaydi (server, login-parol sizda yo'q).
**Tavsiya:** agar botni faqat SMM-xizmat sotish uchun ishlatsangiz, shu "🤖 SMM Bot / boshqa botlar ochish" bo'limini butunlay o'chirib tashlash mantiqan to'g'ri — aks holda foydalanuvchi pul to'lab, hech narsa olmaydi.

### d) Fayl-asosidagi saqlash (`user/*.step`, `set/channel`, `msgs.json`)
Railway konteynerlari **vaqtinchalik disk** bilan ishlaydi — har safar qayta deploy qilinganda yoki konteyner qayta ishga tushganda, shu fayllar **butunlay o'chib ketadi** (foydalanuvchi bosqichlari, referal statistikasi va h.k. yo'qoladi).
**Ikki yechim bor:**
1. **Tezkor:** Railway'da "Volume" (doimiy disk) ulash — kod o'zgarmaydi, lekin faqat 1 ta instance uchun ishlaydi.
2. **To'g'ri yechim:** bu fayllarni MySQL bazasidagi jadvalga ko'chirish (masalan `user_steps` jadvali). Buni ham xohlasangiz qilib beraman.

## 3. Railway'ga joylash — qadamlar

1. **GitHub repo yarating** va shu papkadagi barcha fayllarni (`public/bot.php`, `Procfile`, `nixpacks.toml`, `user/`, `set/`, `step/` papkalari) shu repoga yuklang.
2. Railway'da **New Project → Deploy from GitHub repo**.
3. **MySQL qo'shing:** Railway loyihasida "New → Database → MySQL" bosing. U sizga `MYSQL_URL`, host, user, parol beradi.
4. `app/controller/sql_connect.php` faylini (bu fayl sizda yuklanmagan, uni ham menga bersangiz tekshirib beraman) Railway'ning MySQL ma'lumotlari bilan yangilang — afzali, u yerda ham `getenv('MYSQLHOST')`, `getenv('MYSQLUSER')` va h.k. ishlatish.
5. **Environment Variables** qo'shing (Railway → Variables):
   - `BOT_TOKEN` — @BotFather'dan olingan token
   - `ADMIN_ID` — sizning Telegram ID'ingiz
   - `SIMKEY` — sms-activate.org kaliti
   - `CHANNEL_ID` — majburiy obuna kanali ID'si
6. **Volume qo'shing** (agar fayl-asosidagi saqlashni hozircha shu ko'yi qoldirsangiz): Settings → Volumes → mount path: `/app/user`, `/app/set`, `/app/step`.
7. Deploy tugagach, Railway sizga domen beradi (masalan `https://sizning-loyiha.up.railway.app`). Webhook o'rnating:
   ```
   https://api.telegram.org/bot<BOT_TOKEN>/setWebhook?url=https://sizning-loyiha.up.railway.app/bot.php
   ```

## 4. Keyingi qadam

Yuqoridagi (b), (c), (d) bandlari bo'yicha qanday yo'l tutishimni ayting — shunga qarab faylni yanada to'liqroq tuzatib beraman. Shuningdek, `app/controller/sql_connect.php` va `msgs.json` fayllaringiz bo'lsa, ularni ham yuklang — to'liq tekshirib chiqaman.

## 5. Yangi qo'shilgan: Nomer (SMS) API'ni bot ichidan sozlash

Avval faqat **Nakrutka (SMM) API** admin panel orqali ("🔑 API Sozlamalari" -> "➕ API qo'shish") qo'shilar edi; **Nomer (SMS) API** esa faqat Railway Environment Variables (`SIMKEY`) orqali qattiq kiritilgan edi.

Endi "🗄️ Boshqaruv" panelida yangi tugma bor: **"🔢 Nomer API sozlash"**. U orqali admin:
1. Nomer sotib olinadigan saytning API manzilini yuboradi (masalan `https://api.sms-activate.org/stubs/handler_api.php`, yoki shu protokolga mos boshqa sayt).
2. So'ng o'sha saytdan olingan API kalitni (SIMKEY) yuboradi.
3. Bot darhol saytga ulanib balansni tekshiradi va natijani ko'rsatadi.

Bu qiymatlar `set/sms_api_url.txt` va `set/simkey.txt` fayllarida saqlanadi va **Environment Variables'dagi `SIMKEY`/`SMS_API_URL`dan ustun turadi** — ya'ni Railway'da o'zgartirmasdan, to'g'ridan-to'g'ri botning o'zidan almashtirish mumkin. Fayllar topilmasa, avvalgidek `.env`dagi standart qiymatlarga qaytadi (orqaga moslik saqlangan). Kod ichidagi barcha ~20 ta joyda hardcode qilingan `sms-activate.org` manzili endi shu dinamik o'zgaruvchidan o'qiladi.

**Muhim (Railway uchun):** `set/` papkasi vaqtinchalik diskda saqlanadi (yuqoridagi (d) bandga qarang) — konteyner qayta ishga tushsa, bu yerga yozilgan API manzil/kalit ham o'chib ketishi mumkin, agar Volume ulanmagan bo'lsa. Doimiy saqlash uchun Volume ulashni yoki bu sozlamalarni MySQL'ga ko'chirishni tavsiya qilaman.

## 6. Funksiyalarni tekshirish bo'yicha eslatma

Men faylni statik (kodni o'qish, qavslar balansi, o'zgaruvchilar qamrovi) tarzda diqqat bilan tekshirdim va quyidagilarni tasdiqladim:
- **Nakrutka (SMM) API qo'shish/o'chirish/tahrirlash** — to'liq ishlaydigan holatda, `providers` jadvaliga yozadi ("🔑 API Sozlamalari" bo'limi).
- **Referal tizimi** — kanalga a'zo bo'lgach balansga referal puli qo'shiladi, referal soni oshiriladi.
- **Hisobni to'ldirish (pul kiritish)** va **buyurtma/balansdan yechish** oqimlari — bir xil naqshda (`UPDATE users SET balance=...`) ishlangan.
- Yangi qo'shilgan Nomer API bloki qavslar/qo'shtirnoqlar balansini buzmagani tekshirildi (butun fayl bo'yicha `{}`, `()` balansi 0, o'zgarmagan).

**Lekin:** menda bu muhitda ishlaydigan PHP interpretatori, jonli MySQL baza va tarmoq (internet) ulanishi yo'q — shuning uchun kodni haqiqatan **ishga tushirib**, botni Telegram orqali bosib-sinab ko'ra olmadim. Railway'ga joylagandan so'ng albatta quyidagilarni qo'lda tekshiring: `/start`, referal havolasi, hisob to'ldirish, "🔢 Nomer API sozlash" orqali API kiritish, nomer sotib olish, va nakrutka buyurtma berish. Xatolik chiqsa — Railway loglariga qarang (`log_errors` yoqilgan) va menga xato matnini yuboring, tezda tuzataman.

Shuningdek, 2-band (a)dagi **SQL Injection** muammosi hali ham tuzatilmagan (bu alohida katta ish) — xohlasangiz shu bilan ham yordam beraman.

## 7. "Umuman xato chiqmasin" so'rovi bo'yicha qilingan tuzatishlar

Siz yuborgan skrinshotda `composer install: command not found` xatosi bor edi — sababi va yechimi:

- **Composer xatosi:** paketda ishlatilmayotgan `composer.json` bor edi, Nixpacks uni ko'rib avtomatik `composer install` ishga tushirmoqchi bo'lgan, lekin composer o'rnatilmagan edi. **`composer.json` butunlay olib tashlandi** — endi bu bosqich umuman ishga tushmaydi.

Shundan keyin fayllarni yana chuqurroq tekshirganimda yana uchta jiddiy, "vaqt o'tgan sari xato ko'payishi"ga sabab bo'lishi mumkin bo'lgan muammoni topdim va tuzatdim:

- **`app/controller/sql_connect.php` fayli umuman yo'q edi** — bot.php uni `require` qilgani uchun, agar sizning haqiqiy repo'ingizda ham bu fayl bo'lmasa, bot HAR BIR so'rovda darhol yiqilishi kerak edi. Endi bu faylni yaratib qo'ydim (Railway MySQL Environment Variables orqali avtomatik ulanadi). **Agar sizda bu faylning ASL versiyasi allaqachon mavjud bo'lsa (masalan eski ma'lumotlaringiz bilan) — meniki bilan ALMASHTIRMANG**, faqat o'zingiznikini qoldiring.
- **`app/controller/schema.sql`** — agar hali hech qanday jadvalingiz bo'lmasa (yangi/bo'sh baza), shu faylni bir marta ishga tushirsangiz barcha kerakli jadvallar (`users`, `settings`, `providers`, `services` va h.k.) yaratiladi. Bu ham bot.php ichidagi so'rovlardan "qayta tiklangan" taxminiy struktura — agar eski bazangiz bor bo'lsa, bu faylni ishlatmang.
- **mysqli xatolar sababli butun bot yiqilishi (eng katta ehtimoliy sabab!):** PHP 8.1+ da `mysqli` standart holatda har qanday SQL xatosida (masalan takroriy yozuv, mavjud bo'lmagan qator) `Exception` "otadi". Bu kod esa eski uslubda (`if($natija){...}` deb tekshiradigan) yozilgan — Exception kutilmagan joyda chiqsa, butun so'rov 500-xato bilan yiqiladi. Baza kattalashib, turli-tuman holatlar (edge case'lar) ko'proq uchray boshlagani sari bunday yiqilishlar ham ko'payadi — siz ko'rgan **"xato borgan sari ko'payib boryapti"** degani aynan shu bo'lishi ehtimoli katta. Endi `mysqli_report(MYSQLI_REPORT_OFF)` qo'shildi — bu xatti-harakatni eski (xavfsiz) rejimga qaytaradi.
- **Har qanday kutilmagan xato endi botni yiqitmaydi:** global xato/Exception ushlagichlar qo'shildi — har qanday muammo Railway loglariga yoziladi, lekin foydalanuvchiga yoki Telegram'ga "500 xato" ko'rinishida chiqmaydi (Telegram webhook doim 200 javob oladi, aks holda Telegram webhookni vaqtincha o'chirib qo'yishi mumkin edi).
- **Nisbiy fayl yo'llari (`user/...`, `set/...`, `step/...`) barqarorlashtirildi:** botning ishga tushirilgan joyidan (CWD) qat'i nazar, bu 190+ joydagi yo'llar doim to'g'ri papkaga ishora qilishi uchun skript boshida `chdir()` qo'shildi.

**Halollik uchun aytib qo'yay:** bu — 5000+ qatorli, ko'p yillik "meros" (legacy) kod, va menda uni jonli internet, MySQL va Telegram bilan haqiqatan ishga tushirib sinash imkoniyati yo'q. Yuqoridagilar — men aniqlab, ishonchli tarzda tuzata olgan **haqiqiy, tekshirilgan muammolar**. Shu bilan birga, 2-banddagi SQL Injection kabi tuzatilmagan joylar yoki men bilmagan sizning maxsus sozlamalaringizga bog'liq boshqa nozik holatlar hali ham xato berishi mumkin — "100% hech qachon hech qanday xato chiqmaydi" deb va'da bera olmayman, lekin endi xato chiqsa ham, u botni yiqitmaydi va Railway loglarida aniq ko'rinadi, shu orqali tezda tuzatish mumkin bo'ladi.
