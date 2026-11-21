AYO CHORDS V7 — FIREBASE AUTH + FIRESTORE CLOUD SYNC

ไฟล์นี้เป็นเว็บแอป (PWA) ไม่ใช่ไฟล์ติดตั้ง .ipa หรือ APK
ให้นำไฟล์ทั้งหมดไปวางใน GitHub Pages แทนไฟล์เดิมใน repository เดิม

สิ่งที่เพิ่มในรุ่นนี้:
- Firebase Authentication: Google, Facebook และ Email/Password (คงระบบเดิม)
- เชื่อม Cloud Firestore กับคลังเพลงสาธารณะ public_songs
- เพลงที่เพิ่มใหม่จะบันทึก title, artist, owner_uid และ created_at อัตโนมัติด้วย serverTimestamp()
- รูปคอร์ดจะถูกบีบอัดและเก็บเป็นข้อมูลภาพใน Firestore เพื่อให้ทุกเครื่องที่ล็อกอินเห็นรูปเดียวกัน (ไม่ใช้ Firebase Storage/ไม่ต้องเปิด Blaze)
- ทุกบัญชีที่เข้าสู่ระบบอ่านคลังเพลงสาธารณะร่วมกันได้
- เจ้าของเพลงเท่านั้นที่แก้ไขหรือลบเพลงต้นฉบับได้
- Setlist และประวัติการเปิดเพลงซิงค์แยกตามบัญชี และเก็บเป็นข้อมูลส่วนตัว
- เก็บข้อมูลในเครื่องไว้ด้วย เพื่อให้ใช้งานต่อได้เมื่อออฟไลน์
- เปลี่ยน Service Worker cache เป็น v8 เพื่อบังคับโหลดไฟล์ชุดใหม่

ข้อกำหนด Firestore Security Rules:
ต้อง Publish Rules ที่อนุญาต public_songs และข้อมูลส่วนตัวของ users ตามโครงสร้างนี้ก่อนใช้ Cloud Sync:

rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    function signedIn() { return request.auth != null; }

    match /users/{userId} {
      allow read, write: if signedIn() && request.auth.uid == userId;
      match /{collection}/{docId} {
        allow read, write: if signedIn() && request.auth.uid == userId;
      }
    }

    match /public_songs/{songId} {
      allow read: if true;
      allow create: if signedIn() && request.resource.data.owner_uid == request.auth.uid;
      allow update: if signedIn()
        && resource.data.owner_uid == request.auth.uid
        && request.resource.data.owner_uid == resource.data.owner_uid;
      allow delete: if signedIn() && resource.data.owner_uid == request.auth.uid;
    }
  }
}

ข้อควรทราบ:
- Firestore ใช้เก็บข้อมูลเพลงรวมถึงรูปคอร์ดที่บีบอัดแล้ว และข้อมูล Setlist/ประวัติ
- รูปจะถูกจำกัดขนาดเพื่อให้อยู่ภายในข้อจำกัด 1 MiB ต่อเอกสาร Firestore; การใช้งานรูปจำนวนมากอาจใช้โควต้า Firestore เพิ่มขึ้น
- หมายเหตุ: รูปคอร์ดถูกเก็บในเอกสาร Firestore แทน Firebase Storage เพื่อหลีกเลี่ยงการเปิดแพ็กเกจ Blaze
- ข้อมูลเพลงเดิมใน localStorage จะถูกนำขึ้น public_songs เมื่อผู้ใช้เข้าสู่ระบบครั้งแรก โดยจะกำหนดบัญชีที่กำลังเข้าสู่ระบบเป็นเจ้าของเพลงที่ยังไม่มีเอกสารบนคลาวด์
- อย่าลบข้อมูลเว็บไซต์/Safari หรือข้อมูล localStorage ก่อนสำรองข้อมูล เพราะรูปคอร์ดเดิมยังอยู่ในเครื่อง

วิธีติดตั้ง:
1. แตก ZIP
2. อัปโหลดไฟล์ index.html, service-worker.js, README.txt, manifest.webmanifest และโฟลเดอร์ icons ไปทับไฟล์เดิมใน repository ayochords/ayo-chords
3. Commit changes
4. เปิด https://ayochords.github.io/ayo-chords/ ใน Safari แล้วรีเฟรช หรือเปิด Private Browsing เพื่อตรวจสอบ
5. เข้าสู่ระบบด้วยบัญชีที่ใช้ทดสอบ
6. ทดสอบเพิ่มเพลงใหม่ แล้วตรวจ Firestore > Data > public_songs ว่ามีเอกสารพร้อม created_at แบบ Timestamp หรือไม่

หมายเหตุ: หากหน้าแอปแจ้ง permission-denied ให้ตรวจว่า Firestore Rules ชุดข้างบนถูก Publish แล้วจริงหรือไม่
