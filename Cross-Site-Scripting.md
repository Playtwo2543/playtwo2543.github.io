[<<back to Main](README.md)

# รายงานการแก้ไขช่องโหว่ Cross-Site Scripting (XSS)

ไฟล์ที่ใช้ในการทดสอบ  
> xss_json.php  
> Path: /var/www/html/bWAPP/xss_json.php

---

## 1. ตัวอย่างการทำงานปกติ

ไฟล์ `xss_json.php` เป็นส่วนหนึ่งของระบบ bWAPP ซึ่งทำงานอยู่เบื้องหลังหน้าเว็บ (Backend Endpoint)

ผู้ใช้งาน **ไม่ได้เรียกใช้งานผ่าน URL โดยตรง**  
แต่จะใช้งานผ่านหน้าเว็บ (User Interface) โดยระบบจะส่งค่าจากฟอร์มมายังไฟล์นี้โดยอัตโนมัติ

### ขั้นตอนการทำงานปกติ (Business Flow)

1. ผู้ใช้เปิดหน้าเว็บของระบบ bWAPP
2. ผู้ใช้กรอกชื่อภาพยนตร์ลงในช่องค้นหาบนหน้าเว็บ
3. กดปุ่ม Submit เพื่อส่งข้อมูล
4. ระบบ Frontend ส่งค่าที่กรอกไปยัง Backend (`xss_json.php`)
5. Backend ประมวลผลและส่งผลลัพธ์กลับมาในรูปแบบ JSON
6. หน้าเว็บนำผลลัพธ์ไปแสดงผลให้ผู้ใช้

### ตัวอย่างการทำงานของระบบ (อธิบายเชิงเทคนิค)

เมื่อผู้ใช้กรอกคำว่า **IRON MAN** ผ่าน UI  
ระบบจะส่งค่าไปยัง backend ในรูปแบบ request เช่น /xss_json.php?title=IRON MAN


> หมายเหตุ: URL ข้างต้นเป็นเพียงตัวอย่างเพื่ออธิบายการทำงานภายในระบบ  
> ผู้ใช้งานจริงไม่ได้พิมพ์ URL นี้เอง

ผลลัพธ์ที่แสดงบนหน้าเว็บ
        "Yes! We have that movie..."

ดังนั้น Business Function ของหน้านี้คือ

> “รับค่าชื่อภาพยนตร์จากผู้ใช้ผ่าน UI ของเว็บ  
> แล้วตอบกลับข้อมูลในรูปแบบ JSON เพื่อแสดงผลบนหน้าเว็บ”

---

## 2. ช่องโหว่ที่พบ และการทดสอบก่อนแก้ไข

จากการตรวจสอบ พบว่าหน้านี้มีช่องโหว่ประเภท

> Cross-Site Scripting (Reflected XSS via JSON)

![PIC1](im/IM1.png)

เนื่องจากมีการนำข้อมูลจากผู้ใช้ไปแสดงผลในหน้าเว็บ  
โดยไม่ผ่านการ Encode อย่างเหมาะสม

### โค้ดที่มีช่องโหว่


.```php
$title = $_GET["title"];
$response = $title . "??? Sorry, we don't have that movie :(";

## ข้อมูลจาก $title ถูกนำไปแสดงผลใน JavaScript โดยตรง

document.getElementById("result").innerHTML =
JSONResponse.movies[0].response;

ทำให้สามารถแทรก JavaScript ได้

![PIC](im/M0.png)

การทดสอบก่อนแก้ไข

ใช้ Payload ดังนี้

./xss_json.php?title=<script>alert(1)</script>


### ผลลัพธ์ที่เกิดขึ้น
        Script ถูก execute จริงใน Browser
        มีหน้าต่าง alert แสดงขึ้นมา

แสดงว่า
        
        ❌ ระบบมีช่องโหว่ XSS จริง

---

### 3. ตรวจสอบช่องโหว่ด้วย RIPS
        ทำการสแกนโค้ดด้วยเครื่องมือ RIPS Code Analysis Tool
        พบการแจ้งเตือนว่าไฟล์ xss_json.php มีช่องโหว่ XSS
### แสดงให้เห็นว่า        
        RIPS ตรวจพบช่องโหว่จากการใช้ $_GET
        มีการแสดงผลโดยไม่ผ่านการ Encode
        เป็นช่องโหว่ประเภท Reflected XSS
---

### 4. จุดที่เป็นต้นเหตุของปัญหา
  #### โครงสร้างของช่องโหว่
           Source : $_GET["title"]
           Sink   : innerHTML ใน JavaScript
           Result : Cross-Site Scripting
           
[IMG](im/IM2.png)

#### สาเหตุหลักคือ
        มีการนำข้อมูลจากผู้ใช้ไปแสดงผลโดยไม่ผ่านการ Encode

---

### 5. แนวทางการแก้ไข
        หลักการสำคัญของการป้องกัน XSS คือ
        “ห้ามนำข้อมูลจากผู้ใช้ไปแสดงผลโดยตรง ต้อง Encode ก่อนเสมอ”

#### แนวทางที่ใช้แก้ไข
        กรองข้อมูล input Encode ข้อมูลก่อนแสดงผล
        ใช้ json_encode() อย่างปลอดภัย

---

### 6. โค้ดที่แก้ไขเพื่อปิดช่องโหว่
        ก่อนแก้ไข (มีช่องโหว่)
                $title = $_GET["title"];
                $response = $title . "??? Sorry, we don't have that movie :(";

[IMG](im/IM6.png)

#### โค้ดด้านบนสามารถถูกโจมตี XSS ได้
        
        หลังแก้ไข (ปลอดภัย)
                $title = htmlentities($_GET["title"], ENT_QUOTES, "UTF-8");
                $response = $title . "??? Sorry, we don't have that movie :(";
                        $data = array(
                            "movies" => array(
                                array("response" => $response)
                    )
                );
                $string = json_encode(
                    $data,
                JSON_HEX_TAG | JSON_HEX_APOS | JSON_HEX_QUOT | JSON_HEX_AMP
                );

#### การแก้ไขสำคัญคือ

        ใช้ htmlentities() เพื่อ Encode ข้อมูล
        ใช้ json_encode() แบบปลอดภัย        
        ไม่แสดง input ของผู้ใช้โดยตรง

---

### 7. ทดสอบหลังการแก้ไข
        ใช้วิธีการทดสอบแบบเดียวกับก่อนแก้
                /xss_json.php?title=<script>alert(1)</script>
        ผลลัพธ์หลังแก้ไข
                หน้าเว็บแสดงผลเป็นข้อความธรรมดา
                <script>alert(1)</script>??? Sorry, we don't have that movie :(

โดย ไม่มี alert เด้งขึ้นมา
        Script ไม่ถูก execute
แสดงว่า ✅ ช่องโหว่ XSS ถูกปิดเรียบร้อยแล้ว

---

### 8. ทดสอบ Business Function หลังแก้ไข
        ทดสอบการทำงานผ่าน UI ของระบบอีกครั้ง โดยผู้ใช้กรอกชื่อหนังผ่านหน้าเว็บ
  ตัวอย่างค่าที่ระบบส่งไปยัง backend
          
          /xss_json.php?title=THOR
  
  ผลลัพธ์ที่หน้าเว็บแสดง
        Yes! We have that movie...

แสดงว่า

        ระบบยังทำงานตาม Business Function เดิมได้ครบถ้วน
        การแก้ไขไม่กระทบการทำงานของระบบ

---

### บทสรุป

        ปัญหาที่พบ:  มีช่องโหว่ Cross-Site Scripting ในไฟล์ xss_json.php
            เกิดจากการแสดงผล input ของผู้ใช้โดยไม่ Encode

        แนวทางแก้ไข:  ใช้ htmlentities() กับข้อมูลจากผู้ใช้
             ใช้ json_encode() แบบปลอดภัย
             ไม่แสดงข้อมูล input โดยตรง
        
        ผลลัพธ์หลังแก้ไข:  ปิดช่องโหว่ XSS สำเร็จ
                ระบบยังทำงานได้ตาม Business Function เดิม
                ผ่านการตรวจสอบจาก RIPS
                สูตรป้องกัน XSS โดยสรุป
                
                   User Input + Direct Output  = XSS ❌
                   User Input + Encode ก่อน Output = Safe ✅

