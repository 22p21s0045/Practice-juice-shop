# รายงานช่องโหว่ด้านความปลอดภัย
## OWASP Juice Shop

**โครงการ:** `juice-shop@20.4.0-SNAPSHOT`  
**วันที่ตรวจสอบ:** 20 พฤศจิกายน 2026  
**เครื่องมือที่ใช้:** Snyk Open Source (SCA) และ Snyk Code (SAST)  

---

# 1. สรุปผลการตรวจสอบ (Executive Summary)

จากการตรวจสอบพบปัญหาด้านความปลอดภัยทั้งหมด 56 รายการ แบ่งออกเป็นช่องโหว่จาก Dependencies และช่องโหว่จาก Source Code ดังนี้

| ประเภท | จำนวนปัญหา | Critical | High |
|---|---:|---:|---:|
| SCA (Dependencies) | 25 รายการ (41 เส้นทาง) | 1 | 24 |
| SAST (Source Code) | 31 รายการ | 0 | 31 |
| **รวม** | **56 รายการ** | **1** | **55** |

### ความหมายของผลการตรวจสอบ

- **SCA:** ตรวจสอบแพ็กเกจและไลบรารีที่โปรเจกต์นำมาใช้งานว่ามีช่องโหว่หรือไม่
- **SAST:** ตรวจสอบ Source Code เพื่อค้นหาจุดที่อาจทำให้เกิดช่องโหว่ เช่น SQL Injection และ Hardcoded Secrets

### ประเด็นที่ต้องให้ความสำคัญ

1. พบ `marsdb` ที่มีช่องโหว่ Arbitrary Code Injection และยังไม่มีเวอร์ชันแก้ไข
2. พบการกำหนด Secret Key ไว้ใน Source Code ซึ่งเกี่ยวข้องกับการตรวจสอบ JWT และ HMAC
3. พบช่องโหว่ SQL Injection และ NoSQL Injection ที่อาจทำให้ผู้โจมตีเข้าถึงหรือแก้ไขข้อมูลในฐานข้อมูล
4. พบช่องโหว่ Path Traversal ที่อาจทำให้เข้าถึงไฟล์นอกเหนือจาก Directory ที่อนุญาต

---

# 2. ผลการตรวจสอบ Dependencies (SCA)

พบช่องโหว่จาก Dependencies ทั้งหมด 25 รายการ โดยแบ่งเป็น Critical 1 รายการ และ High 24 รายการ

## 2.1 ช่องโหว่ระดับ Critical

### `marsdb` — Arbitrary Code Injection

| รายละเอียด | ข้อมูล |
|---|---|
| เวอร์ชันที่พบ | `0.6.11` |
| ความรุนแรง | Critical |
| CVSS | 9.8 |
| Vulnerability ID | `SNYK-JS-MARSDB-480405` |
| เวอร์ชันที่แก้ไข | ไม่มี |

**ปัญหาคืออะไร?**

แพ็กเกจ `marsdb` มีช่องโหว่ที่อาจทำให้ผู้โจมตีสามารถแทรก JavaScript Code ผ่าน `$where` Operator และนำไปสู่การรันคำสั่งโดยไม่ได้รับอนุญาต

**แนวทางแก้ไข**

- เปลี่ยนไปใช้แพ็กเกจอื่น เนื่องจากไม่มีเวอร์ชันแก้ไขช่องโหว่
- พิจารณาใช้ `lowdb`, `nedb-promises` หรือ `better-sqlite3`
- ตรวจสอบการใช้งาน `$where` และทดสอบระบบหลังเปลี่ยนแพ็กเกจ

---

## 2.2 ช่องโหว่ระดับ High

เพื่อให้แก้ไขได้ง่าย แบ่งตามแพ็กเกจต้นทางที่เป็นสาเหตุของปัญหา

### 1. ช่องโหว่ในการแตกไฟล์ Archive (`tar`)
**Dependency Path:**

`sqlite3 → node-gyp → tar`

**แนวทางแก้ไข**
พบช่องโหว่ในแพ็กเกจ tar ซึ่งเป็น Dependency ที่ถูกเรียกใช้งานผ่านลำดับ Dependency ดังนี้

| ID                   | รายละเอียดช่องโหว่                                | CVSS | CVE            | เวอร์ชัน tar ที่แก้ไขแล้ว |
| -------------------- | ------------------------------------------------- | ---- | -------------- | -------------------- |
| SNYK-JS-TAR-15307072 | Directory Traversal ผ่าน Symlink ไปยัง Hardlink   | 8.4  | CVE-2026-26960 | `≥ 7.5.8`            |
| SNYK-JS-TAR-15416075 | Symlink Attack ผ่าน Drive-relative Path (`C:../`) | 8.2  | CVE-2026-29786 | `≥ 7.5.10`           |
| SNYK-JS-TAR-15456201 | Symlink Attack ผ่านฟังก์ชัน `tar.x()`             | 8.2  | CVE-2026-31802 | `≥ 7.5.11`           |
| SNYK-JS-TAR-17909068 | Infinite Loop จากค่า Negative Base-256 Size       | 8.7  | CVE-2026-59874 | `≥ 7.5.18`           |
| SNYK-JS-TAR-17909152 | Resource Exhaustion จาก Gzip Bomb                 | 8.7  | CVE-2026-59873 | `≥ 7.5.19`           |

### **แนวทางการแก้ไข**

แนะนำให้อัปเดตแพ็กเกจ `sqlite3` เป็นเวอร์ชัน `6.0.1` เพื่อให้ Dependency ที่เกี่ยวข้องได้รับการอัปเดต โดยมีเป้าหมายให้ใช้ `tar` เวอร์ชัน `7.5.19` ขึ้นไป

### 2. ช่องโหว่ในระบบ JWT / Authentication

**Dependency Path:**

`express-jwt → jsonwebtoken → jws → base64url`
`express-jwt → jsonwebtoken → moment`

**แนวทางแก้ไข**
พบช่องโหว่ในแพ็กเกจกลุ่ม JWT authentication ซึ่งเป็น Dependency ที่ถูกเรียกใช้งานผ่านลำดับ Dependency ดังนี้

| ID | รายละเอียดช่องโหว่ | CVSS | CVE | เวอร์ชันที่แก้ไขแล้ว |
| --- | --- | --- | --- | --- |
| `npm:jsonwebtoken:20150331` | Authentication Bypass ผ่าน `alg:none` | 7.5 | CVE-2015-9235 | `≥ 4.2.2` |
| `npm:jws:20160726` | Forgeable Tokens จาก HMAC/RSA Confusion | 8.7 | CVE-2016-1000223 | `≥ 3.0.0` |
| `SNYK-JS-JWS-14188253` | Improper Verification of Cryptographic Signature | 8.2 | CVE-2025-65945 | `3.2.3` / `4.0.1` |
| `SNYK-JS-EXPRESSJWT-575022` | Authorization Bypass (ไม่บังคับ `algorithms`) | 7.4 | CVE-2020-15084 | `≥ 6.0.0` |
| `npm:base64url:20180511` | Uninitialized Memory Exposure | 7.1 | — | `≥ 3.0.0` |
| `SNYK-JS-MOMENT-2440688` | Directory Traversal ผ่าน `locale()` | 7.5 | CVE-2022-24785 | `≥ 2.29.2` |
| `SNYK-JS-MOMENT-19821607` | Directory Traversal (Bypass ของ CVE-2022-24785) | 8.7 | CVE-2026-17495 | `≥ 2.31.0` |

### **แนวทางการแก้ไข**

แนะนำให้อัปเดตแพ็กเกจหลักทั้ง 3 ตัว เพื่อให้ Dependency ที่เกี่ยวข้องทั้งหมดได้รับการแก้ไข โดยมีเป้าหมายดังนี้

| แพ็กเกจ | เวอร์ชันเป้าหมาย | หมายเหตุ |
| --- | --- | --- |
| `express-jwt` | `≥ 6.0.0` | แก้ Authorization Bypass + ดึง `jws`, `jsonwebtoken`, `moment` เวอร์ชันใหม่ |
| `jsonwebtoken` | `≥ 5.0.0` | แก้ Authentication Bypass (`alg:none`) |
| `moment` | `≥ 2.31.0` | แก้ Directory Traversal ทั้ง 2 CVE (รวม bypass) |

> ⚠️ **หมายเหตุเพิ่มเติม:** หลังอัปเดตแล้ว ควรตรวจสอบการเรียกใช้ `jsonwebtoken.verify()` และ `express-jwt` middleware ว่ามีการระบุ `algorithms: ['RS256']` (หรือ algorithm ที่ต้องการ) อย่างชัดเจน เพื่อป้องกัน Key Confusion Attack ในอนาคต

### 3. ช่องโหว่ Prototype Pollution / Code Injection (`lodash`)

**Dependency Path:**

`sanitize-html → lodash`

**แนวทางแก้ไข**
พบช่องโหว่ในแพ็กเกจ `lodash` ซึ่งเป็น Dependency ที่ถูกเรียกใช้งานผ่านลำดับ Dependency ดังนี้

| ID | รายละเอียดช่องโหว่ | CVSS | CVE | เวอร์ชัน lodash ที่แก้ไขแล้ว |
| --- | --- | --- | --- | --- |
| SNYK-JS-LODASH-73638 | Prototype Pollution ผ่าน `merge`, `mergeWith` | 7.3 | CVE-2018-16487 | `≥ 4.17.11` |
| SNYK-JS-LODASH-450202 | Prototype Pollution ผ่าน `defaultsDeep` | 7.3 | CVE-2019-10744 | `≥ 4.17.12` |
| SNYK-JS-LODASH-608086 | Prototype Pollution ผ่าน `set`, `setWith` | 7.3 | — | `≥ 4.17.17` |
| SNYK-JS-LODASH-6139239 | Prototype Pollution ผ่าน `zipObjectDeep` | 7.5 | — | `≥ 4.17.17` |
| SNYK-JS-LODASH-1040724 | Code Injection ผ่าน `_.template` (variable) | 7.2 | CVE-2021-23337 | `≥ 4.17.21` |
| SNYK-JS-LODASH-15869625 | Arbitrary Code Injection ผ่าน `_.template` (imports) | 8.6 | CVE-2026-4800 | `≥ 4.18.1` |

### **แนวทางการแก้ไข**

แนะนำให้อัปเดตแพ็กเกจ `sanitize-html` เป็นเวอร์ชัน `1.7.1` เพื่อให้ Dependency ที่เกี่ยวข้องได้รับการอัปเดต โดยมีเป้าหมายให้ใช้ `lodash` เวอร์ชัน `4.18.1` ขึ้นไป ซึ่งจะแก้ไขช่องโหว่ทั้ง 6 รายการข้างต้นได้ในคราวเดียว

> ⚠️ **หมายเหตุเพิ่มเติม:** ช่องโหว่ `SNYK-JS-LODASH-15869625` (CVE-2026-4800) มี CVSS สูงสุดที่ 8.6 และเป็นการ inject code ผ่าน `_.template` imports ซึ่งอาจนำไปสู่ Remote Code Execution ได้ ควรให้ความสำคัญในการอัปเดตเป็นลำดับแรก

### 4. ช่องโหว่ XML และเครื่องมือสร้าง SBOM

**Dependency Path:**

`@cyclonedx/cyclonedx-npm → libxmljs2`
`@cyclonedx/cyclonedx-npm → ajv-formats-draft2019 → uri-js`
`pdfkit → crypto-js`

**แนวทางแก้ไข**
พบช่องโหว่ในแพ็กเกจกลุ่ม XML parsing และ SBOM tooling ซึ่งเป็น Dependency ที่ถูกเรียกใช้งานผ่านลำดับ Dependency ดังนี้

| ID | รายละเอียดช่องโหว่ | CVSS | CVE | เวอร์ชันที่แก้ไขแล้ว |
| --- | --- | --- | --- | --- |
| SNYK-JS-LIBXMLJS2-6808810 | Type Confusion ผ่าน `attrs()` → RCE/DoS | 8.1 | CVE-2024-34393 | ❌ ไม่มีเวอร์ชันแก้ไข |
| SNYK-JS-LIBXMLJS2-6808816 | Type Confusion ผ่าน `namespaces()` → RCE/DoS | 8.1 | CVE-2024-34394 | ❌ ไม่มีเวอร์ชันแก้ไข |
| SNYK-JS-URIJS-19963963 | Infinite Loop ผ่าน `removeDotSegments` (U+2028/U+2029) | 8.7 | CVE-2026-93690 | ❌ ไม่มีเวอร์ชันแก้ไข |
| SNYK-JS-CYCLONEDXCYCLONEDXNPM-17817903 | Command Injection ผ่าน `--workspace` | 8.5 | CVE-2026-55849 | `≥ 5.0.0` |
| SNYK-JS-CYCLONEDXCYCLONEDXNPM-20158546 | Command Injection ผ่าน `--workspace` (Windows) | 8.5 | CVE-2026-71538 | `≥ 6.0.0` |
| SNYK-JS-CRYPTOJS-6028119 | Weak Hash (PBKDF2 ใช้ SHA-1 + 1 iteration) | 7.2 | CVE-2023-46233 | `≥ 4.2.0` |

### **แนวทางการแก้ไข**

| แพ็กเกจ | แนวทาง | หมายเหตุ |
| --- | --- | --- |
| `libxmljs2` | **เปลี่ยนเป็น** `@xmldom/xmldom` หรือ `fast-xml-parser` | แพ็กเกจไม่ maintained แล้ว ไม่มี patch แก้ไข |
| `uri-js` | **เปลี่ยนเป็น** `whatwg-url` หรือ validate input ก่อนเรียก `normalize()`/`resolve()` | แพ็กเกจไม่ maintained แล้ว ไม่มี patch แก้ไข |
| `@cyclonedx/cyclonedx-npm` | อัปเดตเป็น `6.0.0` | แก้ Command Injection ทั้ง 2 CVE |
| `pdfkit` | อัปเดตเป็น `0.12.2` | จะดึง `crypto-js@≥4.2.0` มาให้ |

> ⚠️ **หมายเหตุเพิ่มเติม:** 
> - `libxmljs2` และ `uri-js` **ไม่มีเวอร์ชันแก้ไข** — ต้องเปลี่ยนแพ็กเกจเท่านั้น ไม่สามารถอัปเดตเพื่อแก้ได้
> - `crypto-js` maintainer แจ้งว่า **หยุดพัฒนาแล้ว** — ควรพิจารณาเปลี่ยนไปใช้ Node.js native `crypto` module แทน

# 3. ผลการตรวจสอบ Source Code (SAST)

พบปัญหาจาก Source Code ทั้งหมด 31 รายการ โดยแบ่งตามประเภทดังนี้

## 3.1 Hardcoded Secrets — การกำหนด Secret ไว้ใน Source Code

**พบ 5 จุด — ไฟล์ `lib/insecurity.ts`**

| บรรทัด | ปัญหาที่พบ |
|---|---|
| 20 | กำหนด Secret สำหรับตรวจสอบ JWT ไว้ใน Source Code |
| 21 | กำหนด Secret สำหรับ HMAC ไว้ใน Source Code |
| 42 | พบ Secret ที่ใช้กับ HMAC |
| 52 | พบ Secret ที่ใช้กับ express-jwt |

**ความเสี่ยง**

หากผู้โจมตีเข้าถึง Source Code ได้ อาจนำ Secret เหล่านี้ไปใช้ปลอมแปลง Token หรือลายเซ็นที่ระบบใช้ตรวจสอบความถูกต้อง

**แนวทางแก้ไข**

- ย้าย Secret ออกจาก Source Code
- ใช้ Environment Variables หรือ Secret Manager เช่น HashiCorp Vault
- เปลี่ยน Secret ที่เคยถูกเปิดเผยแล้ว (Secret Rotation)

## 3.2 NoSQL Injection — การแทรกคำสั่งผ่านฐานข้อมูล NoSQL

**พบ 6 จุด**

ไฟล์ที่เกี่ยวข้อง:
- `routes/likeProductReviews.ts`
- `routes/updateProductReviews.ts`

**ความเสี่ยง**

ระบบนำข้อมูลจาก HTTP Request ไปใช้กับคำสั่งฐานข้อมูลโดยตรง ทำให้ผู้โจมตีอาจส่งข้อมูลที่ถูกดัดแปลงเพื่อเปลี่ยนเงื่อนไขการค้นหาหรือแก้ไขข้อมูลได้

**แนวทางแก้ไข**

- ตรวจสอบรูปแบบข้อมูลก่อนส่งไปยังฐานข้อมูล
- ไม่รับ Query Operator จากผู้ใช้โดยตรง
- กำหนด Schema สำหรับข้อมูลที่รับเข้ามา
- ใช้ฟังก์ชัน Query ที่ปลอดภัยและจำกัดรูปแบบ Input

## 3.3 Path Traversal — การเข้าถึงไฟล์นอกพื้นที่ที่กำหนด

**พบ 8 จุด**

ไฟล์ที่เกี่ยวข้อง:
- `routes/fileServer.ts`
- `routes/keyServer.ts`
- `routes/logfileServer.ts`
- `routes/quarantineServer.ts`
- `routes/fileUpload.ts`
- `routes/profileImageUrlUpload.ts`
- `routes/vulnCodeSnippet.ts`

**ความเสี่ยง**

ผู้โจมตีอาจแก้ไข Path ที่ส่งเข้ามาเพื่อเข้าถึงไฟล์นอก Directory ที่ระบบอนุญาต หรือเขียนไฟล์ไปยังตำแหน่งที่ไม่ควรเข้าถึง

**แนวทางแก้ไข**

- ใช้ `path.resolve()` เพื่อตรวจสอบตำแหน่งไฟล์
- ตรวจสอบว่า Path ที่ได้ยังอยู่ภายใน Directory ที่อนุญาต
- ใช้ชื่อไฟล์ที่ระบบสร้างขึ้นเองแทนชื่อจากผู้ใช้
- จำกัดสิทธิ์การอ่านและเขียนไฟล์ของ Application

## 3.4 SQL Injection — การแทรกคำสั่ง SQL

**พบ 6 จุด**

ไฟล์ที่เกี่ยวข้อง:
- `data/static/codefixes/dbSchemaChallenge_1.ts`
- `data/static/codefixes/unionSqlInjectionChallenge_1.ts`
- `data/static/codefixes/unionSqlInjectionChallenge_3.ts`
- `routes/search.ts`
- `routes/login.ts`

**ความเสี่ยง**

ผู้โจมตีอาจส่ง SQL Statement ผ่าน Input ของผู้ใช้ เพื่อเปลี่ยนแปลงเงื่อนไขการ Query หรือเข้าถึงข้อมูลที่ไม่ควรได้รับอนุญาต

**แนวทางแก้ไข**

- ใช้ Parameterized Queries หรือ Prepared Statements
- หลีกเลี่ยงการนำ Input มาต่อ String เพื่อสร้าง SQL Query
- ตรวจสอบ Input ก่อนนำไปใช้งาน

## 3.5 Server-Side Request Forgery (SSRF)

**พบ 1 จุด**

ไฟล์: `routes/profileImageUrlUpload.ts`  
บรรทัด: 24  
ฟังก์ชัน: `fetch()`

**ความเสี่ยง**

ระบบรับ URL จากผู้ใช้แล้วนำไปเรียกใช้งานจากฝั่ง Server โดยตรง ผู้โจมตีอาจใช้ช่องทางนี้ให้ Server ส่ง Request ไปยังระบบภายในที่ไม่ควรเข้าถึงได้

**แนวทางแก้ไข**

- อนุญาตเฉพาะ Domain ที่อยู่ใน Allowlist
- ป้องกันการเรียก IP ภายในและ Loopback Address
- ตรวจสอบ DNS Resolution และป้องกันการเปลี่ยนปลายทางระหว่าง Request
- จำกัด Protocol ให้ใช้เฉพาะที่จำเป็น เช่น HTTPS

## 3.6 Cross-Site Scripting (XSS)

**พบ 5 จุด**

ไฟล์ที่เกี่ยวข้อง:
- `frontend/.../search-result.component.ts`
- `frontend/.../track-result.component.ts`
- `routes/recycles.ts`
- `routes/userProfile.ts`

**ความเสี่ยง**

ผู้โจมตีอาจแทรก JavaScript ผ่านข้อมูลที่ระบบนำไปแสดงผล ทำให้ Browser ของผู้ใช้งานประมวลผล Script ที่ไม่พึงประสงค์

**แนวทางแก้ไข**

- หลีกเลี่ยงการใช้ `bypassSecurityTrustHtml()` กับข้อมูลที่ผู้ใช้ควบคุม
- ใช้ระบบ Sanitization ของ Angular
- Escape ข้อมูลก่อนนำไปแสดงผล
- เพิ่ม Content Security Policy (CSP)

---

# 4. แผนการแก้ไขตามลำดับความสำคัญ (Priority Action Plan)

แบ่งการแก้ไขออกเป็น 3 ระดับ เพื่อให้ทีม Developer สามารถวางแผนดำเนินงานได้ง่ายขึ้น

## 🔴 ต้องดำเนินการเร่งด่วน

| รายการ | แนวทางแก้ไข |
|---|---|
| `marsdb` | เปลี่ยนแพ็กเกจ เนื่องจากไม่มีเวอร์ชันแก้ไข |
| `libxmljs2` | เปลี่ยน XML Parser |
| Hardcoded Secrets | ย้าย Secret ออกจาก Source Code และเปลี่ยน Secret เดิม |

## 🟠 ควรแก้ไขในรอบพัฒนาปัจจุบัน

| รายการ | แนวทางแก้ไข |
|---|---|
| SQL Injection | เปลี่ยนเป็น Parameterized Queries |
| NoSQL Injection | ตรวจสอบและจำกัด Input ก่อน Query |
| Path Traversal | ตรวจสอบ Path และจำกัด Directory |
| XSS | ใช้ Sanitization และ Escape Output |
| SSRF | ใช้ Domain Allowlist และป้องกัน Internal Network Access |
| `sqlite3` / `tar` | อัปเดต Dependency |
| JWT Dependencies | อัปเดตและทดสอบ Authentication |
| `lodash` | อัปเดต Dependency ที่ได้รับผลกระทบ |
| `@cyclonedx/cyclonedx-npm` | อัปเดตเป็นเวอร์ชันที่แก้ไขแล้ว |
| `crypto-js` | อัปเดตเวอร์ชัน |

## 🟡 ปรับปรุงเพิ่มเติม

| รายการ | แนวทางแก้ไข |
|---|---|
| `moment` | อัปเดตหรือเปลี่ยนไปใช้ `dayjs` / `date-fns` |
| `uri-js` | เปลี่ยนแพ็กเกจหรือเพิ่ม Input Validation |
| Security Automation | เพิ่ม Snyk ใน CI/CD Pipeline |

---

**หมายเหตุ:** การอัปเดตแพ็กเกจควรตรวจสอบความเข้ากันได้กับระบบก่อนเสมอ ไม่ควรใช้ `npm audit fix` หรือบังคับเปลี่ยน Dependency โดยไม่ทดสอบ Regression Test

---

**แหล่งข้อมูล:** Snyk Open Source (SCA) และ Snyk Code (SAST)  
**วันที่จัดทำรายงาน:** 20 พฤศจิกายน 2026