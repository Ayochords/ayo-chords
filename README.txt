AYO CHORDS V7 — Firebase Authentication

AYO CHORDS V6 UPDATE — Setlist & Random Drawer

ไฟล์นี้เป็นเว็บแอป (PWA) ไม่ใช่ไฟล์ติดตั้ง .ipa หรือ APK
ห้ามแตะเปิดไฟล์ ZIP เพื่อใช้งานโดยตรง

วิธีทดลองเปิด:
1. ดาวน์โหลด ZIP แล้วแตะไฟล์ในแอป Files เพื่อแตกไฟล์
2. เข้าโฟลเดอร์ที่แตกไฟล์ แล้วแตะ index.html
3. หาก iPhone แสดงเป็นตัวอย่างไฟล์หรือเปิดไม่สมบูรณ์ นั่นเป็นข้อจำกัดของการเปิด HTML จาก Files

วิธีติดตั้งเป็นไอคอนแอปบน iPhone:
1. ต้องนำไฟล์ทั้งหมดใน ZIP (index.html, manifest.webmanifest, service-worker.js และโฟลเดอร์ icons) ไปวางบนเว็บไซต์ HTTPS
2. เปิด URL เว็บไซต์นั้นด้วย Safari
3. แตะปุ่ม Share > Add to Home Screen > Add
4. เปิดจากไอคอน AYO CHORDS ที่เพิ่มบนหน้าจอ

ข้อควรระวัง:
- อย่าลบข้อมูลเว็บไซต์/เบราว์เซอร์เดิม และอย่าลบแอปเวอร์ชันเดิม เพราะเพลงและรูปคอร์ดเก็บในพื้นที่ของเบราว์เซอร์
- เวอร์ชันนี้เพิ่มตัวค้นหาเพลงใน Setlist, ปุ่มเลื่อนลำดับเพลงขึ้น/ลง, ปุ่มบันทึกที่ซ่อนรายชื่อเพลงหลังบันทึก, ช่องค้นหา Setlist แยกจากส่วนสร้าง, ปัดลงเพื่อปิดหน้ารูปคอร์ด และปัดขึ้นเพื่อเปิดแถบสุ่มเพลง 10 เพลงแบบ 2 แถว
- เปลี่ยนชื่อแคช Service Worker เป็น v2 เพื่อให้ Safari ดึงไฟล์ใหม่
- เวอร์ชันนี้ยังไม่ได้ทดสอบบน iPhone จริง และไม่รับประกันว่าจะนำข้อมูล V5 มาให้เอง
- ก่อนเปลี่ยนเวอร์ชัน ควรสำรองเพลงและรูปคอร์ดก่อน

V7 AUTHENTICATION:
- Login screen appears before the song library.
- Google and Facebook sign-in buttons are implemented.
- Email/password registration, login, and password reset are implemented.
- Firebase Auth local persistence is enabled.
- Facebook requires enabling the provider in Firebase Authentication and configuring Meta App ID/Secret plus OAuth redirect URL.
- Google requires the provider enabled and the deployed domain authorized in Firebase Authentication.
- Existing ayo_songs and ayo_sets localStorage keys are preserved; this build does not migrate data to Firestore or sync across devices.
