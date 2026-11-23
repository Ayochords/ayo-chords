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

      // สร้าง/แก้ไขเพลงได้เฉพาะเจ้าของ และต้องมี reservation
      // ที่ชี้กลับมาหาเพลงนี้หลังจบ transaction
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

      // เพลงเก่าที่ไม่มี dedupe_lock_id ยังลบได้โดยเจ้าของ
      // ส่วนเพลงใหม่ต้องปลด reservation ใน transaction เดียวกัน
      allow delete: if signedIn()
        && resource.data.owner_uid == request.auth.uid
        && (!(resource.data.dedupe_lock_id is string)
          || !existsAfter(/databases/$(database)/documents/song_uniqueness/$(resource.data.dedupe_lock_id))
          || getAfter(/databases/$(database)/documents/song_uniqueness/$(resource.data.dedupe_lock_id)).data.song_id != songId);
    }

    // Reservation ป้องกันเพลงซ้ำ
    match /song_uniqueness/{lockId} {
      // ต้องอ่านได้เมื่อเข้าสู่ระบบ เพื่อให้ transaction ตรวจพบ
      // reservation ที่สมาชิกคนอื่นถืออยู่และปฏิเสธเพลงซ้ำ
      allow read: if signedIn();

      allow create, update: if signedIn()
        && request.resource.data.owner_uid == request.auth.uid
        && getAfter(/databases/$(database)/documents/public_songs/$(request.resource.data.song_id)).data.owner_uid == request.auth.uid
        && getAfter(/databases/$(database)/documents/public_songs/$(request.resource.data.song_id)).data.dedupe_lock_id == lockId
        && getAfter(/databases/$(database)/documents/public_songs/$(request.resource.data.song_id)).data.dedupe_key == request.resource.data.dedupe_key;

      // ปลด reservation ได้เฉพาะเจ้าของ และเมื่อเพลงหลัง transaction
      // ไม่มีการอ้างอิง lock นี้แล้ว
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

## เหตุผลที่เพิ่มกฎ

- V8 อ่านและเขียน `song_uniqueness` ภายใน Firestore Transaction แต่กฎเดิมไม่มี match สำหรับ collection นี้
- `getAfter()` ใช้ตรวจว่าการเขียนเพลงและ reservation เชื่อมโยงกันหลัง transaction ไม่ใช่เขียนแยกกัน
- จำกัดการแก้ไข/ลบ reservation ให้เฉพาะเจ้าของ
- รักษาสิทธิ์ users, public_songs และ song_contributions ตามกฎเดิมที่ได้รับ

## ข้อควรตรวจสอบก่อน Deploy

1. ทดสอบ rules ใน Firebase Emulator หรือ Firebase test project ก่อน
2. ทดสอบเพิ่มเพลงใหม่และแก้ชื่อเพลง (ต้องสร้าง reservation ใหม่และปลดอันเก่าได้)
3. ทดสอบลบเพลงใหม่และเพลงเก่าที่ไม่มี `dedupe_lock_id`
4. ทดสอบสมาชิกสองบัญชีเพิ่มเพลงชื่อ/ศิลปินเดียวกันพร้อมกัน — ต้องมีเพียงหนึ่งบัญชีสำเร็จ
5. ทดสอบ Setlist, ประวัติการค้นหา, Contributions และการอ่านคลังเพลง
6. ตรวจ schema ของเอกสารเก่าและข้อมูลจริงก่อน Deploy

ข้อจำกัด: ร่างนี้ยังไม่ได้ผ่าน Emulator/Test Project และยังไม่ได้ Publish ไปยัง Firebase จริง ห้ามนำไปแทนกฎ Production โดยตรงก่อนทดสอบ regression ครบทุกกรณี
