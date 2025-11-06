# Reverse Proxy Sample (Nginx + Node.js Microservices)

ตัวอย่างนี้แสดงวิธีการตั้งค่า Reverse Proxy โดยใช้ Nginx เพื่อจัดการและส่งต่อคำขอไปยัง Node.js Microservice สองตัว ได้แก่ `deviceMs` และ `twinMs` ซึ่งรันอยู่ใน Docker Container ต่างหาก การใช้ Docker Compose จะช่วยให้การจัดการระบบที่มีหลายบริการทำได้ง่ายขึ้นมาก

## วัตถุประสงค์การเรียนรู้

*   ทำความเข้าใจบทบาทและการทำงานของ Nginx ในฐานะ Reverse Proxy
*   เรียนรู้วิธีการใช้ Docker Compose เพื่อจัดการแอปพลิเคชันที่มีหลาย Container
*   ตระหนักถึงความสำคัญของการทำความเข้าใจ "สภาพแวดล้อม" ในการพัฒนาและ Debugging
*   เข้าใจแนวคิดเบื้องต้นด้านความปลอดภัยจากการสร้างสภาพแวดล้อมที่ควบคุมได้

## สิ่งที่ต้องมี (Prerequisites)

*   **Docker:** ติดตั้ง Docker Desktop หรือ Docker Engine บนระบบปฏิบัติการของคุณ
*   **Docker Compose:** โดยปกติจะมาพร้อมกับ Docker Desktop หรือสามารถติดตั้งแยกได้ (`docker compose` เป็นคำสั่งใหม่ แทน `docker-compose`)

## โครงสร้างโปรเจกต์

```
reverse-proxy-sample/
├── deviceMs/             # Node.js Microservice สำหรับจัดการ Device
│   ├── Dockerfile
│   ├── index.js
│   ├── package.json
│   └── ...
├── twinMs/               # Node.js Microservice สำหรับจัดการ Twin
│   ├── Dockerfile
│   ├── index.js
│   ├── package.json
│   └── ...
├── ngnixReverseProxy/    # การตั้งค่า Nginx Reverse Proxy
│   └── default.conf
└── docker-compose.yml    # ไฟล์สำหรับจัดการและรันบริการทั้งหมดด้วย Docker Compose
```

## ขั้นตอนการตั้งค่าและรันระบบ

1.  **โคลน Repository (หรือ Fork ของคุณ):**
    หากคุณยังไม่ได้โคลน Repository นี้ หรือต้องการใช้ Fork ของคุณเอง:
    ```bash
    git clone <URL_ของ_Repository_นี้_หรือ_Fork_ของคุณ>
    cd MicroServices-Samples/reverse-proxy-sample
    ```
    (หากคุณใช้ Fork และมีไฟล์ที่แก้ไขอยู่แล้ว ให้คัดลอกไฟล์ที่แก้ไขเหล่านั้นมายังโฟลเดอร์นี้)

2.  **แก้ไขไฟล์ Microservices:**
    เนื่องจาก Microservice ทั้งสอง (`deviceMs` และ `twinMs`) มีการคาดหวังไฟล์ `token` ซึ่งอาจไม่มีอยู่จริง คุณจะต้องแก้ไขโค้ดเล็กน้อยเพื่อให้สามารถรันได้:

    *   **เปิดไฟล์ `deviceMs/index.js`** และค้นหาบรรทัด:
        ```javascript
        const token = fs.readFileSync('/etc/connecthing-api/token')
        ```
        เปลี่ยนเป็น:
        ```javascript
        const token = process.env.DAVRA_API_TOKEN || ''
        ```

    *   **เปิดไฟล์ `twinMs/index.js`** และค้นหาบรรทัด:
        ```javascript
        const token = fs.readFileSync('/etc/connecthing-api/token')
        ```
        เปลี่ยนเป็น:
        ```javascript
        const token = process.env.DAVRA_API_TOKEN || ''
        ```

    *เหตุผลของการแก้ไข:* เพื่อให้ Microservice สามารถเริ่มทำงานได้โดยไม่ต้องหาไฟล์ token ในช่วงเริ่มต้น หากต้องการใช้งานฟังก์ชันที่ต้องยืนยันตัวตน คุณจะต้องตั้งค่า `DAVRA_API_TOKEN` เป็น environment variable ใน `docker-compose.yml`

3.  **รันระบบทั้งหมดด้วย Docker Compose:**
    ไปที่โฟลเดอร์ `reverse-proxy-sample/` แล้วรันคำสั่งนี้:
    ```bash
    sudo docker compose up --build -d
    ```
    *   `sudo docker compose up`: อ่านไฟล์ `docker-compose.yml` เพื่อสร้างและรันบริการทั้งหมด
    *   `--build`: สั่งให้ Docker สร้าง Image ใหม่สำหรับ `deviceMs` และ `twinMs` (จาก `Dockerfile` ในโฟลเดอร์ย่อย) ทุกครั้งที่รัน เผื่อมีการเปลี่ยนแปลงโค้ด
    *   `-d`: รัน Container ทั้งหมดในโหมดเบื้องหลัง (detached mode)

## การตรวจสอบ (Verification)

1.  **ตรวจสอบสถานะของบริการ:**
    ใช้คำสั่งนี้เพื่อดูว่า Container ทั้งหมดกำลังทำงานอยู่หรือไม่:
    ```bash
    sudo docker compose ps
    ```
    คุณควรจะเห็น `nginx-proxy`, `devicemspoc`, และ `twinmspoc` มีสถานะเป็น `Up`

2.  **ทดสอบ Reverse Proxy:**
    ใช้ `curl` หรือเว็บเบราว์เซอร์เพื่อทดสอบการเข้าถึง Microservice ผ่าน Nginx Proxy:
    ```bash
    curl http://localhost:8080/device
    curl http://localhost:8080/twin
    ```
    คุณควรได้รับข้อความต้อนรับจากแต่ละ Microservice:
    *   `Hello World from Device MS`
    *   `Hello World from Twin microservice`

    หากได้รับผลลัพธ์เหล่านี้ แสดงว่าระบบทั้งหมดทำงานได้อย่างถูกต้องแล้ว

## การทำความสะอาด (Cleanup)

เมื่อคุณต้องการหยุดและลบ Container, Network และ Volume ทั้งหมดที่สร้างโดย Docker Compose:
```bash
sudo docker compose down
```

## แนวคิดสำคัญที่ได้เรียนรู้

*   **Nginx Reverse Proxy:** ทำหน้าที่เป็นด่านหน้าในการรับคำขอและส่งต่อ (proxy) ไปยังบริการภายใน ช่วยเพิ่มความปลอดภัย, การทำ Load Balancing และการจัดการ SSL/TLS
*   **Docker Compose:** ทำให้การจัดการแอปพลิเคชันที่มี Container หลายตัวทำได้ง่ายขึ้น โดยกำหนดการตั้งค่าทั้งหมดไว้ในไฟล์ `docker-compose.yml` เพียงไฟล์เดียว
*   **ความสำคัญของ "สภาพแวดล้อม":** การทำความเข้าใจสภาพแวดล้อมที่โค้ดทำงาน (OS, Runtime, Libraries, Configuration, Network) เป็นหัวใจสำคัญในการพัฒนาและ Debugging
*   **ความมั่นคงปลอดภัยใน Container:** การสร้าง Container ที่มีส่วนประกอบน้อยที่สุด (Minimal Image) ช่วยลด Attack Surface และทำให้ Hacker เจาะระบบได้ยากขึ้น เนื่องจากไม่มีเครื่องมือพื้นฐานให้ใช้เมื่อเจาะเข้ามาได้

---
หวังว่า README นี้จะเป็นประโยชน์ในการเรียนรู้และทำความเข้าใจตัวอย่างนี้และแนวคิดที่เกี่ยวข้องนะครับ!