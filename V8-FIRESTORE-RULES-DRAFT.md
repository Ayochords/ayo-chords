# V8 Firestore Security Rules — Merged Review Draft

สถานะ: ร่างสำหรับตรวจสอบเท่านั้น — ยังไม่ได้ Deploy ไป Firebase

ร่างนี้รวมกฎปัจจุบันที่ผู้ใช้ส่งมา และเพิ่มการตรวจสอบความสัมพันธ์แบบ atomic ระหว่าง `public_songs` กับ `song_uniqueness` ตามที่โค้ด V8 ใช้งาน

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    function signedIn() {
      return request.auth != null;
    }

    // โปรไฟล์ส่วนตัว, Setlist และประวัติการค้นหา
    match /users/{userId} {
      allow read, write: if signedIn()
        && request.auth.uid == userId;

      match /{collection}/{docId} {
        allow read, write: if signedIn()
          && request.auth.uid == userId;
      }
    }

    // คลังเพลงสาธารณะ
    match /public_songs/{songId} {
      allow read: if true;

      allow create: if signedIn()
        && request.resource.data.owner_uid == request.auth.uid
        && request.resource.data.dedupe_lock_id is string
        && getAfter(/databases/$(database)/documents/song_uniqueness/$(request.resource.data.dedupe_lock_id)).data.song_id == songId
        && getAfter(/databases/$(database)/documents/song_uniqueness/$(request.resource.data.dedupe_lock_id)).data.owner_uid == request.auth.uid
        && getAfter(/databases/$(database)/documents/song_uniqueness/$(request.resource.data.dedupe_lock_id)).data.dedupe_key == request.resource.data.dedupe_key;

      allow update: if signedIn()
        && resource.data.owner_uid == request.auth.uid
        && request.resource.data.owner_uid == resource.data.owner_uid
        && request.resource.data.dedupe_lock_id is string
        && getAfter(/databases/$(database)/documents/song_uniqueness/$(request.resource.data.dedupe_lock_id)).data.song_id == songId
        && getAfter(/databases/$(database)/documents/song_uniqueness/$(request.resource.data.dedupe_lock_id)).data.owner_uid == request.auth.uid
        && getAfter(/databases/$(database)/documents/song_uniqueness/$(request.resource.data.dedupe_lock_id)).data.dedupe_key == request.resource.data.dedupe_key;

      allow delete: if signedIn()
        && resource.data.owner_uid == request.auth.uid
        && (!(resource.data.dedupe_lock_id is string)
          || !existsAfter(/databases/$(database)/documents/song_uniqueness/$(resource.data.dedupe_lock_id))
          || getAfter(/databases/$(database)/documents/song_uniqueness/$(resource.data.dedupe_lock_id)).data.song_id != songId);
    }

    // Reservation ป้องกันเพลงซ้ำ
    match /song_uniqueness/{lockId} {
      // Client ต้องอ่าน lock ของผู้อื่นได้เพื่อปฏิเสธเพลงซ้ำ
      // ข้อมูลใน collection นี้จึงต้องไม่มีข้อมูลส่วนตัวหรือข้อมูลลับ
      allow read: if signedIn();

      allow create: if signedIn()
        && request.resource.data.owner_uid == request.auth.uid
        && getAfter(/databases/$(database)/documents/public_songs/$(request.resource.data.song_id)).data.owner_uid == request.auth.uid
        && getAfter(/databases/$(database)/documents/public_songs/$(request.resource.data.song_id)).data.dedupe_lock_id == lockId
        && getAfter(/databases/$(database)/documents/public_songs/$(request.resource.data.song_id)).data.dedupe_key == request.resource.data.dedupe_key;

      // สำคัญ: ห้ามผู้ใช้แก้ reservation ที่เดิมเป็นของสมาชิกคนอื่น
      allow update: if signedIn()
        && resource.data.owner_uid == request.auth.uid
        && request.resource.data.owner_uid == request.auth.uid
        && getAfter(/databases/$(database)/documents/public_songs/$(request.resource.data.song_id)).data.owner_uid == request.auth.uid
        && getAfter(/databases/$(database)/documents/public_songs/$(request.resource.data.song_id)).data.dedupe_lock_id == lockId
        && getAfter(/databases/$(database)/documents/public_songs/$(request.resource.data.song_id)).data.dedupe_key == request.resource.data.dedupe_key;

      allow delete: if signedIn()
        && resource.data.owner_uid == request.auth.uid
        && (!existsAfter(/databases/$(database)/documents/public_songs/$(resource.data.song_id))
          || getAfter(/databases/$(database)/documents/public_songs/$(resource.data.song_id)).data.dedupe_lock_id != lockId);
    }

    // ข้อเสนอแก้ไขเพลง
    match /song_contributions/{contributionId} {
      allow create: if signedIn()
        && request.resource.data.user_uid == request.auth.uid;
      allow read: if signedIn();
      allow update, delete: if false;
    }
  }
}
```

## จุดที่แก้เพิ่มในการตรวจสอบรอบนี้

- แยก `create` และ `update` ของ `song_uniqueness`
- การ `update` ต้องตรวจว่า reservation เดิมเป็นของผู้ใช้ที่ล็อกอินอยู่ด้วย
- การอ่าน reservation ยังอนุญาตเฉพาะผู้ใช้ที่ล็อกอิน เพราะแอปต้องตรวจพบ lock ที่มีเจ้าของคนอื่นก่อนปฏิเสธเพลงซ้ำ
- ห้ามเก็บข้อมูลส่วนตัวหรือข้อมูลลับไว้ใน `song_uniqueness` เพราะผู้ใช้ที่ล็อกอินทุกคนอ่าน collection นี้ได้

## ข้อควรตรวจสอบก่อน Deploy

1. ทดสอบ Rules ใน Firebase Emulator หรือ Firebase test project ก่อน
2. ทดสอบเพิ่มเพลงใหม่และแก้ชื่อเพลง (ต้องสร้าง reservation ใหม่และปลดอันเก่าได้)
3. ทดสอบลบเพลงใหม่และเพลงเก่าที่ไม่มี `dedupe_lock_id`
4. ทดสอบสมาชิกสองบัญชีเพิ่มเพลงชื่อ/ศิลปินเดียวกันพร้อมกัน — ต้องมีเพียงหนึ่งบัญชีสำเร็จ
5. ทดสอบว่าบัญชี A ไม่สามารถ update/delete reservation ของบัญชี B ได้
6. ทดสอบ Setlist, ประวัติการค้นหา, Contributions และการอ่านคลังเพลง
7. ตรวจ schema ของเอกสารเก่าและข้อมูลจริงก่อน Deploy

ข้อจำกัด: ร่างนี้ยังไม่ได้ผ่าน Emulator/Test Project และยังไม่ได้ Publish ไปยัง Firebase จริง ห้ามนำไปแทนกฎ Production โดยตรงก่อนทดสอบ regression ครบทุกกรณี
