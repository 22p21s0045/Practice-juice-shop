## 1. Overview

โปรเจกต์นี้เป็นการพัฒนา Automated Security Scanning Pipeline โดยใช้ **GitHub Actions** ร่วมกับ **Snyk CLI** เพื่อใช้ตรวจสอบช่องโหว่ด้านความปลอดภัยของ Source Code และ Third-party Dependencies ภายในโปรเจกต์

Pipeline นี้ถูกออกแบบให้ทำหน้าที่เป็น **Security Gate** ในกระบวนการ CI/CD โดยจะทำการตรวจสอบความปลอดภัยโดยอัตโนมัติเมื่อมีการ Push Code ไปยัง Branch ที่กำหนด หรือเมื่อมีการสร้าง Pull Request

การตรวจสอบแบ่งออกเป็น 2 ประเภทหลัก ได้แก่

- **Snyk Code (SAST):** วิเคราะห์ Source Code เพื่อค้นหาช่องโหว่ด้านความปลอดภัยและรูปแบบการเขียนโปรแกรมที่ไม่ปลอดภัย โดยไม่จำเป็นต้อง Execute Application
- **Snyk Open Source (SCA):** วิเคราะห์ Third-party Dependencies รวมถึง Transitive Dependencies เพื่อค้นหาช่องโหว่ที่มีการเปิดเผยและบันทึกไว้ในฐานข้อมูลช่องโหว่ของ Snyk

Pipeline กำหนด Severity Threshold ไว้ที่ระดับ **High** ซึ่งหมายความว่าหากตรวจพบช่องโหว่ระดับ High หรือ Critical จะทำให้ Quality Gate ไม่ผ่าน และส่งผลให้ Pipeline มีสถานะ Failed

## 2. Pipeline Design

Pipeline ถูกแบ่งออกเป็น 6 ขั้นตอนหลัก ดังนี้

1. **SCM Trigger** – กำหนดเงื่อนไขในการเริ่มต้น Pipeline
2. **Environment Setup** – เตรียม Environment สำหรับการ Scan
3. **Security Scanning** – ตรวจสอบช่องโหว่ด้วย SAST และ SCA
4. **Result Collection** – รวบรวมผลการตรวจสอบ
5. **Quality Gate** – ประเมินผลการตรวจสอบตาม Security Policy
6. **Pipeline Status** – สรุปผลการทำงานของ Pipeline

### 2.1 SCM Trigger

กำหนดให้ Pipeline ทำงานเมื่อเกิด GitHub Events ตามเงื่อนไขดังนี้

| Event | Target Branch |
|---|---|
| Push | `master` |
| Push | `feature/**` |
| Pull Request | `master` |
| Pull Request | `feature/**` |




### 2.2 Environment Setup

เป็นขั้นตอนการเตรียม Environment บน GitHub Actions Runner เพื่อให้พร้อมสำหรับการตรวจสอบช่องโหว่

ขั้นตอนประกอบด้วย

- Checkout Source Code จาก Repository
- ติดตั้ง Node.js เวอร์ชัน 22
- ติดตั้ง Snyk CLI โดยกำหนดเวอร์ชันที่แน่นอนเป็น `1.1308.4`
- ติดตั้ง Dependencies ของโปรเจกต์ด้วย `npm install --ignore-scripts`
- Authentication กับ Snyk ผ่าน `SNYK_TOKEN` ที่จัดเก็บไว้ใน GitHub Actions Secrets

**Security Considerations**

- จัดเก็บ `SNYK_TOKEN` ไว้ใน GitHub Actions Secrets เพื่อป้องกันการเปิดเผย Credential ภายใน Repository
- Pin เวอร์ชันของ Snyk CLI เพื่อให้การทำงานของ Pipeline มีความสม่ำเสมอ
- อ้างอิง GitHub Actions ด้วย Commit SHA เพื่อลดความเสี่ยงจากการเปลี่ยนแปลงของ Action ที่อ้างอิงด้วย Tag
- ใช้ `--ignore-scripts` เพื่อป้องกันการ Execute Lifecycle Scripts ของ Dependencies ในระหว่างการติดตั้ง

### 2.3 Security Scanning

ขั้นตอนนี้เป็นการตรวจสอบช่องโหว่โดยใช้ Snyk แบ่งออกเป็น 2 ส่วน ซึ่งทำงานแยกจากกัน

**A. Snyk Code (SAST)**

ใช้สำหรับวิเคราะห์ Source Code เพื่อค้นหาช่องโหว่ด้านความปลอดภัย เช่น การเขียน Code ที่อาจทำให้เกิด Security Vulnerabilities โดยไม่จำเป็นต้อง Execute Application

คำสั่งที่ใช้:

```bash
snyk code test \
  --severity-threshold=high \
  --sarif-file-output=results/snyk-code.sarif
```

ผลลัพธ์:
- `results/snyk-code.sarif`

**B. Snyk Open Source (SCA)**

ใช้สำหรับตรวจสอบ Third-party Dependencies และ Transitive Dependencies เพื่อค้นหาช่องโหว่ที่มีการเปิดเผยใน Open Source Packages

คำสั่งที่ใช้:

```bash
snyk test \
  --severity-threshold=high \
  --json-file-output=results/snyk-sca.json
```

ผลลัพธ์:
- `results/snyk-sca.json`

ทั้งสองขั้นตอนกำหนด `continue-on-error: true` เพื่อให้ Pipeline สามารถดำเนินการในขั้นตอนถัดไปได้ แม้ว่าจะตรวจพบช่องโหว่หรือเกิดข้อผิดพลาดระหว่างการ Scan

แนวทางนี้ช่วยให้ Pipeline สามารถดำเนินการ Scan ทั้งสองประเภทและรวบรวมผลลัพธ์ได้ก่อนที่จะนำผลทั้งหมดไปประเมินในขั้นตอน Quality Gate

### 2.4 Result Collection

เป็นขั้นตอนการรวบรวมผลการตรวจสอบจาก Snyk และจัดเก็บเป็น GitHub Actions Artifacts เพื่อให้สามารถดาวน์โหลดและตรวจสอบรายละเอียดภายหลังได้

ชื่อ Artifact:

```text
snyk-results
```

ไฟล์ผลลัพธ์ประกอบด้วย

| File | Description |
|---|---|
| `snyk-code.sarif` | ผลการตรวจสอบช่องโหว่จาก SAST |
| `snyk-sca.json` | ผลการตรวจสอบช่องโหว่จาก SCA |

กำหนด `if: always()` ให้กับขั้นตอน Upload Results เพื่อให้ระบบพยายามรวบรวมผลลัพธ์ แม้ว่าขั้นตอนก่อนหน้าจะเกิดข้อผิดพลาดก็ตาม

ช่วยให้ Developer และ Security Engineer สามารถตรวจสอบรายละเอียดของช่องโหว่ที่ตรวจพบได้ แม้ว่า Pipeline จะมีสถานะ Failed

### 2.5 Quality Gate

เป็นขั้นตอนการประเมินผลการตรวจสอบจากทั้ง SAST และ SCA โดยพิจารณาจาก Exit Code ที่ส่งกลับมาจาก Snyk CLI

| Exit Code | ความหมาย | ผลการประเมิน |
|---|---|---|
| `0` | ไม่พบช่องโหว่ตั้งแต่ระดับ Severity Threshold ที่กำหนด | Pass |
| `1` | ตรวจพบช่องโหว่ตั้งแต่ระดับ Severity Threshold ที่กำหนด | Fail |
| `2` | เกิดข้อผิดพลาดในการทำงานของ Snyk CLI หรือการ Execute | Fail |
| `3` | ไม่พบ Project หรือไฟล์ที่รองรับสำหรับการ Scan | Fail |

**เงื่อนไขการทำงานของ Quality Gate**

1. หาก SAST หรือ SCA ส่ง Exit Code `2` หรือ `3` จะถือว่าเกิดข้อผิดพลาดในการตรวจสอบ และ Pipeline จะไม่ผ่าน
2. หาก SAST หรือ SCA ส่ง Exit Code `1` จะถือว่าตรวจพบช่องโหว่ระดับ High หรือ Critical และ Pipeline จะไม่ผ่าน
3. หากทั้ง SAST และ SCA ส่ง Exit Code `0` จะถือว่าผ่าน Security Gate
4. หากไม่สามารถรับ Exit Code หรือได้รับค่าที่ไม่ตรงกับเงื่อนไขที่กำหนด จะถือว่าไม่ผ่าน Security Gate

การออกแบบนี้ช่วยป้องกันไม่ให้ Pipeline ผ่านโดยไม่ตั้งใจ ในกรณีที่ Scanner ไม่สามารถทำงานได้อย่างถูกต้อง

### 2.6 Pipeline Status

สถานะสุดท้ายของ Pipeline จะถูกกำหนดโดยผลลัพธ์จาก Quality Gate

| Condition | Result |
|---|---|
| SAST และ SCA ผ่านทั้งคู่ | Pipeline PASSED |
| ตรวจพบช่องโหว่ระดับ High / Critical | Pipeline FAILED |
| เกิดข้อผิดพลาดระหว่างการ Scan | Pipeline FAILED |
| ไม่พบ Project หรือไฟล์ที่รองรับ | Pipeline FAILED |

Pipeline นี้ทำหน้าที่เป็น Security Enforcement Mechanism ภายใน CI/CD Workflow เพื่อช่วยลดความเสี่ยงจากการนำ Source Code หรือ Dependencies ที่มีช่องโหว่เข้าสู่กระบวนการพัฒนาในขั้นตอนถัดไป

## 3. Pipeline Flow

```text
          Developer Push / Pull Request
                       |
                       v
                  SCM Trigger
                       |
                       v
             GitHub Actions Runner
                       |
                       v
               Environment Setup
             +---------------------+
             | Checkout Source     |
             | Install Node.js 22  |
             | Install Snyk CLI    |
             | Install Dependencies|
             | Authenticate Snyk   |
             +---------------------+
                       |
                       v
                Security Scanning
                 /             \
                v               v
           Snyk Code       Snyk Open Source
             (SAST)              (SCA)
                |                 |
                v                 v
           SARIF Report       JSON Report
                |                 |
                +--------+--------+
                         |
                         v
                 Result Collection
                         |
                         v
                   Quality Gate
                         |
               +---------+---------+
               |                   |
               v                   v
          All Scans Pass     Vulnerability /
               |             Scanner Error
               v                   |
          Pipeline PASS            v
                              Pipeline FAIL
```

ข้อจำกัดด้าน Container และ Infrastructure Security
Pipeline ปัจจุบันตรวจสอบเพียง Source Code และ Dependencies เท่านั้น

## AI Usage
มีการใช้ AI เป็นเครื่องมือช่วยสนับสนุนการดำเนินโปรเจกต์ โดยใช้ในการค้นคว้าและรวบรวมข้อมูลที่เกี่ยวข้องกับ Security Scanning, Snyk CLI และ GitHub Actions รวมถึงช่วยจัดเรียงเนื้อหาและร่างเอกสารรายงาน (Report) เพื่อให้เนื้อหามีความชัดเจนและเป็นระบบมากยิ่งขึ้น