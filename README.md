# ChatBot-LineOA
Chatbot LineOA + RAG + Pinecone + Gemini Embedding (PaLM) (embedding-001) + Hugging Face Inference API

## ภาพรวม

ระบบนี้คือ Chatbot บน LineOA ที่ตอบคำถาม HR, วันลา, เอกสาร และข้อมูลความรู้ขององค์กรโดยอัตโนมัติ  
ใช้เทคนิค Retrieval-Augmented Generation (RAG) ร่วมกับ Pinecone (Vector Database)  
และใช้ Google Gemini (PaLM) Models/embedding-001 ใน n8n สำหรับสร้างเวกเตอร์ embedding  
**โดยใช้ Hugging Face Inference API เป็น LLM หลัก เนื่องจากต้องการใช้งานฟรี และความเร็วไม่ใช่ปัจจัยสำคัญ**  
เพื่อให้ค้นหาข้อมูลได้แม่นยำและตอบกลับผู้ใช้ได้อย่างชาญฉลาด

---

## สถาปัตยกรรมระบบ

- **LineOA**: ช่องทางหลักสำหรับผู้ใช้
- **LINE LIFF**: สำหรับ Login/Register และดึง line_user_id
- **n8n**: Orchestration/Logic (Workflow Automation)
    - ใช้ Node: Embeddings Google Gemini (PaLM) Models/embedding-001
    - ใช้ Node: Pinecone
    - ใช้ Node: Hugging Face Inference API
- **Pinecone**: Vector Database สำหรับเก็บเวกเตอร์ embedding
- **Google Drive**: เก็บไฟล์เอกสารต้นฉบับ (PDF, Word, Text)
- **Google Apps Script/Web Admin**: สำหรับหน้า Admin จัดการเอกสาร/สิทธิ์
- **Google Sheets**: เก็บข้อมูลผู้ใช้ วันลา เอกสาร (สำหรับข้อมูลเชิงโครงสร้าง)

---

## โครงสร้างข้อมูล

### 1. Pinecone (Vector Database)

แต่ละ record จะมี:
- `id`: รหัสเฉพาะของ chunk
- `vector`: เวกเตอร์ embedding (ขนาด 768 จาก Gemini embedding-001)
- `metadata`:
    - `content`: ข้อความต้นฉบับ (chunk)
    - `source_pdf`: ชื่อไฟล์ต้นทาง
    - `page_number`: เลขหน้า (ถ้ามี)
    - `category`: ประเภทเอกสาร
    - `document_id`: รหัสเอกสาร (ถ้ามี)
    - `permission`: สิทธิ์การเข้าถึง (เช่น employee_id, group)

### 2. Google Sheets

- **Customers Sheet**: line_user_id, employee_id, customer_name, email, group, role
- **Leave Sheet**: employee_id, line_user_id, ข้อมูลวันลาแต่ละประเภท
- **Documents Sheet**: document_id, employee_id, line_user_id, document_type, document_name, document_url

---

## Flow การทำงาน (อธิบายทีละขั้นตอน)

### A. เตรียมข้อมูล (Ingestion Flow)

1. **Admin อัปโหลดไฟล์เอกสาร**  
   - ผ่านหน้า Admin (Google Apps Script หรือ Web App)
   - ไฟล์จะถูกเก็บใน Google Drive

2. **แปลงไฟล์เป็นข้อความ**  
   - ใช้ Library หรือ API (เช่น pdfplumber, PyPDF2, Google Drive OCR)  
   - ได้ข้อความดิบจากไฟล์ PDF/Word/Text

3. **แบ่งข้อความเป็น Chunk**  
   - แบ่งข้อความเป็นย่อหน้า/หัวข้อ/ขนาดที่เหมาะสม (เช่น 300-500 ตัวอักษรต่อ chunk)

4. **สร้างเวกเตอร์ Embedding**  
   - ส่งแต่ละ chunk ไปที่ n8n
   - ใช้ Node: Embeddings Google Gemini (PaLM) Models/embedding-001  
     - ได้เวกเตอร์ขนาด 768 มิติ

5. **บันทึกลง Pinecone**  
   - สร้าง index ใน Pinecone (dimension=768)
   - ส่งเวกเตอร์ + metadata (content, source_pdf, page_number, category, permission ฯลฯ) ไปเก็บใน Pinecone

---

### B. การตอบคำถาม (Query Flow)

1. **ผู้ใช้ส่งคำถามผ่าน LineOA**  
   - ข้อความถูกส่งเข้า n8n ผ่าน Webhook

2. **สร้างเวกเตอร์ Embedding ของคำถาม**  
   - ใช้ Node: Embeddings Google Gemini (PaLM) Models/embedding-001  
   - ได้เวกเตอร์ขนาด 768 มิติ

3. **ค้นหา Context ที่เกี่ยวข้องใน Pinecone**  
   - ใช้ Node: Pinecone  
   - ส่งเวกเตอร์ของคำถามไปค้นหาใน Pinecone (topK=3-5)
   - ได้ chunk ข้อความที่ใกล้เคียงที่สุด พร้อม metadata

4. **สร้าง Prompt สำหรับ LLM**  
   - รวม context (chunk ที่ได้จาก Pinecone) + คำถามของผู้ใช้  
   - ตัวอย่าง prompt:  
     ```
     ผู้ใช้ถาม: {question}
     ข้อมูลที่เกี่ยวข้อง: {context1} {context2} {context3}
     กรุณาสรุปคำตอบที่เหมาะสมและกระชับ โดยเน้นข้อมูลที่เกี่ยวข้องกับคำถามเท่านั้น
     ```

5. **สรุปคำตอบด้วย Hugging Face Inference API**  
   - ใช้ Node: Hugging Face Inference API  
   - เลือก Model ที่เหมาะสม (เช่น `google/flan-t5-xl`, `facebook/bart-large-cnn`)
   - ส่ง prompt ที่สร้างไปให้ Hugging Face Inference API สรุปและเรียบเรียงคำตอบ

6. **ส่งคำตอบกลับ LineOA**  
   - ใช้ Node: Line Notify หรือ HTTP Request  
   - ส่งข้อความคำตอบกลับไปยังผู้ใช้

---

### C. การดึงข้อมูลส่วนตัว (Personal Data Flow)

1. **ผู้ใช้ขอข้อมูลส่วนตัว (วันลา, เอกสาร)**  
   - n8n รับ line_user_id จาก LineOA
   - ค้นหา employee_id จาก Customers Sheet
   - Query Leave Sheet หรือ Documents Sheet ด้วย employee_id
   - สร้างข้อความตอบกลับและส่งกลับ LineOA

---

### D. การจัดการเอกสาร/สิทธิ์ (Admin Flow)

1. **Admin Login**  
   - เข้าสู่หน้า Admin (Google Apps Script/Web App)

2. **อัปโหลด/แก้ไข/ลบเอกสาร**  
   - อัปโหลดไฟล์ใหม่, แก้ไข metadata, ลบไฟล์ (ลบทั้งใน Google Drive และ Pinecone)

3. **เพิ่ม/แก้ไข/ลบ ประเภทเอกสาร**  
   - จัดการประเภทเอกสารใน Google Sheets หรือ Database

4. **กำหนดสิทธิ์ผู้ใช้**  
   - กำหนด role, group, หรือ permission ใน Customers Sheet

---

## ตัวอย่างการใช้งาน n8n (Pseudo Workflow)

- **Node 1:** Webhook (รับคำถามจาก LineOA)
- **Node 2:** Embeddings Google Gemini (PaLM) Models/embedding-001 (แปลงคำถามเป็นเวกเตอร์)
- **Node 3:** Pinecone (ค้นหา context ที่เกี่ยวข้อง)
- **Node 4:** Function (รวม context + คำถาม สร้าง prompt)
- **Node 5:** Hugging Face Inference API (สรุปคำตอบ)
- **Node 6:** HTTP Request/Line Notify (ส่งคำตอบกลับ LineOA)

---

## หมายเหตุสำคัญ

- Pinecone Free Tier มีข้อจำกัดด้านจำนวน index, ขนาดข้อมูล, และ request rate
- Google Gemini (PaLM) embedding-001 อาจมีข้อจำกัดด้าน API usage และ rate limit
- Hugging Face Inference API มีข้อจำกัดด้าน Rate Limit และ Performance
- ต้องออกแบบ permission และ security ให้รัดกุม โดยเฉพาะข้อมูลส่วนบุคคล
- การแปลงไฟล์ PDF/Word เป็นข้อความควรตรวจสอบความถูกต้องและความสมบูรณ์ของข้อมูล

---

## แหล่งข้อมูลเพิ่มเติม

- [Pinecone Documentation](https://www.pinecone.io/docs/)
- [n8n Documentation](https://docs.n8n.io/)
- [Google Cloud AI Platform](https://cloud.google.com/vertex-ai/docs/generative-ai/embeddings/get-text-embeddings)
- [Hugging Face Inference API](https://huggingface.co/docs/api-inference/quicktour)
- [LineOA Messaging API](https://developers.line.biz/en/docs/messaging-api/overview/)

---

## ติดต่อ

หากมีข้อสงสัยหรืออยากได้ตัวอย่าง workflow เพิ่มเติม ติดต่อได้ที่ [siraphob.an@gmail.com](mailto:siraphob.an@gmail.com)
