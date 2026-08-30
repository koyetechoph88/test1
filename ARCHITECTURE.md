13. โครงสร้างฐานข้อมูล
ทุกตารางอยู่ใน schema public ของ Supabase และเปิด Row Level Security ทั้งหมด · ผ่านการทดสอบรันจริงบน PostgreSQL 16 แล้ว
13.1 ตารางหลัก
profiles — ผู้ใช้ทุกบทบาท ผูก 1:1 กับ auth.users ของ Supabase Auth
คอลัมน์	ชนิด	หมายเหตุ
id	uuid	PK, FK → auth.users.id
full_name	text	
role	app_role	doctor, opd_nurse, or_nurse, company_rep, admin
company_id	uuid	FK → companies เฉพาะ company_rep เท่านั้นที่มีค่า
department	text	
is_active	boolean	ปิดการใช้งานได้โดยไม่ต้องลบ
created_at	timestamptz	
companies — บริษัทเลนส์
คอลัมน์	ชนิด	หมายเหตุ
id	uuid	PK
name / code	text	code unique
contact_email / contact_phone	citext / text	
return_window_days	int	เงื่อนไขคืนของ ใช้เตือนตอนเลื่อนนัด
is_active	boolean	
lens_catalog — รุ่นเลนส์และช่วงค่ากำลังที่มีจริง
คอลัมน์	ชนิด	หมายเหตุ
id	uuid	PK
company_id	uuid	FK → companies
model	text	unique ร่วมกับ company_id
power_min / power_max / power_step	numeric	ใช้ตรวจว่าค่าเลนส์ที่สั่งอยู่ในช่วงที่มี
price	numeric	
lead_time_days	int	ใช้พยากรณ์ว่าของจะทันวันผ่าตัดไหม
patients — ผู้ป่วย (ช่วงพัฒนาเป็นข้อมูลสังเคราะห์ทั้งหมด)
คอลัมน์	ชนิด	หมายเหตุ
id	uuid	PK
hn	citext	unique — ตัวยืนยันหลัก มาจากการสแกนบาร์โค้ด
full_name	text	มี GIN trigram index สำหรับ fuzzy matching
birth_date	date	ใช้ช่วยยืนยันตัวตนกรณีชื่อซ้ำ
is_synthetic	boolean	ธงกันข้อมูลจริงหลุดเข้าสภาพแวดล้อมพัฒนา
biometry — ค่าวัดตาที่ใช้กำหนดค่าเลนส์
คอลัมน์	ชนิด	หมายเหตุ
id	uuid	PK
patient_id / eye	uuid / eye_side	FK → patients, OD หรือ OS
axial_length, k1, k2, acd	numeric	ค่าจากเครื่องวัด
formula / target_power	text / numeric	สูตรและค่ากำลังที่คำนวณได้
source_file	text	path ไฟล์ต้นฉบับใน Supabase Storage
extracted_by_ai / ai_confidence	boolean / numeric	ค่านี้ AI อ่านมาหรือคนคีย์
verified_by / verified_at	uuid / timestamptz	ใครตรวจทานและเมื่อไร
orders — ใบสั่งเลนส์ (เวอร์ชันปัจจุบัน) ตารางกลางของทั้งระบบ
คอลัมน์	ชนิด	หมายเหตุ
id / order_no	uuid / text	order_no สร้างอัตโนมัติ รูปแบบ IOL-YYYYMMDD-XXXXXX
patient_id / eye / surgery_date / surgeon_id		ข้อมูลเคส
company_id / lens_id / lens_power / quantity		ข้อมูลเลนส์ที่สั่ง
biometry_id	uuid	ค่าที่ใช้อ้างอิงตอนกำหนดค่าเลนส์
status	order_status	10 สถานะ ตั้งแต่ draft ถึง used / on_hold / cancelled
version	int	เพิ่มเองอัตโนมัติเมื่อแก้เนื้อหาใบสั่ง
created_by / confirmed_by / confirmed_at		ผู้สร้างและผู้ยืนยัน (ต้องเป็นแพทย์)
duplicate_ack	boolean	ติ๊กเมื่อยืนยันว่าตั้งใจสั่งซ้ำ
cancel_reason / cancel_note / cancelled_at / hold_since		ข้อมูลการยกเลิกและการพักใบสั่ง
lot_no / received_by / received_at		ข้อมูลตอนตรวจรับ
used_lens_power / used_at	numeric / timestamptz	ค่าที่ใช้จริง อาจต่างจากที่สั่ง
order_versions — snapshot ทุกเวอร์ชันของใบสั่ง (order_id, version, snapshot jsonb, changed_by, change_reason, changed_at) ใช้ย้อนดูว่าเคยแก้อะไรไปบ้าง
order_events — ไทม์ไลน์ทุกเหตุการณ์ (order_id, event_type, from_status, to_status, note, actor_id, created_at) เขียนอัตโนมัติด้วย trigger
duplicate_overrides — บันทึกการกดข้ามคำเตือนสั่งซ้ำซ้อน (order_id, conflicting_order_id, reason, overridden_by, created_at)
returns — การคืนของจากเคสที่เลื่อนหรือยกเลิก (order_id, status, reason, requested_by, picked_up_at, completed_at, handled_by)
ai_runs — บันทึกทุกครั้งที่เรียก AI (task, order_id, patient_id, input_summary, output, confidence, model, latency_ms, accepted, corrected_value, reviewed_by) เพื่อให้ผลลัพธ์ AI ตรวจทานย้อนหลังได้ตามกติกาข้อ 7 โดยไม่เก็บภาพต้นฉบับ
app_settings — ค่าตั้งค่าของระบบและของผู้ใช้แต่ละคน
คอลัมน์	ชนิด	หมายเหตุ
id	uuid	PK
user_id	uuid	FK → profiles มีค่า = ค่าส่วนตัว, เป็น null = ค่าระดับระบบ
key	text	unique ต่อผู้ใช้หนึ่งคน และ unique ต่อระดับระบบ
value	jsonb	รูปแบบยืดหยุ่น ไม่ต้องแก้ schema เมื่อเพิ่มค่าตั้งค่าใหม่
created_at / updated_at	timestamptz	updated_at อัปเดตอัตโนมัติด้วย trigger
ตัวอย่างการใช้งาน: ค่าส่วนตัวเช่น notify_channels (แจ้งเตือนทางอีเมลหรือ LINE) และ default_company ส่วนค่าระดับระบบเช่น alert_days_before_surgery (เตือนก่อนวันผ่าตัดกี่วัน) duplicate_check_mode และ ai_confidence_threshold (ค่าความมั่นใจขั้นต่ำที่ยอมให้ AI เติมค่าอัตโนมัติ)
13.2 ความสัมพันธ์
companies      1:N  lens_catalog
companies      1:N  profiles          (เฉพาะ company_rep)
companies      1:N  orders
profiles       1:N  orders            (created_by / confirmed_by / received_by)
profiles       1:N  app_settings      (ค่าส่วนตัว — user_id null คือค่าระดับระบบ)
patients       1:N  biometry
patients       1:N  orders
lens_catalog   1:N  orders
biometry       1:1  orders            (อ้างอิงค่าที่ใช้กำหนดค่าเลนส์)
orders         1:N  order_versions
orders         1:N  order_events
orders         1:N  duplicate_overrides
orders         1:N  returns
orders         1:N  ai_runs
ข้อบังคับระดับฐานข้อมูลที่สำคัญที่สุด คือ partial unique index บน orders (patient_id, eye) เฉพาะใบสั่งที่ยัง active และยังไม่ติ๊ก duplicate_ack ทำให้ผู้ป่วยรายเดียวกันข้างเดียวกันมีใบสั่งที่เดินอยู่ได้เพียงใบเดียว การกันสั่งซ้ำจึงเกิดที่ฐานข้อมูล ไม่ใช่แค่ที่หน้าจอ
13.3 ความปลอดภัยระดับแถว
ทุกตารางที่มีข้อมูลผู้ใช้เปิด RLS และไม่มีเส้นทางใดที่เข้าถึงข้อมูลได้โดยไม่ผ่าน policy สิทธิ์อ่านสรุปได้ดังนี้
ตาราง	บุคลากรโรงพยาบาล	ผู้แทนบริษัท
patients	เข้าถึงได้	ไม่เห็นเลย
biometry	เข้าถึงได้	ไม่เห็นเลย
orders	เห็นทั้งหมด	เฉพาะบริษัทตนเอง และไม่เห็นสถานะ draft
lens_catalog	อ่านได้ทั้งหมด	แก้ได้เฉพาะรุ่นของตนเอง
returns	สร้างและติดตามได้	เห็นและปิดงานของตนเองได้
ai_runs	อ่านและตรวจทานได้	ไม่เห็น
app_settings	เฉพาะของตนเอง และค่าระดับระบบ	เฉพาะของตนเอง และค่าระดับระบบ
order_events / order_versions	ตามสิทธิ์ของใบสั่งแม่	ตามสิทธิ์ของใบสั่งแม่
หลักการที่ใช้ร่วมกันทั้งระบบ
1.	สิทธิ์ทั้งหมดตัดสินจาก auth.uid() ผ่านฟังก์ชัน app_current_role() และ current_company_id() ซึ่งประกาศเป็น security definer เพื่อไม่ให้ policy วนซ้ำกับ RLS ของ profiles เอง
2.	ผู้ใช้แก้โปรไฟล์ตัวเองได้ แต่แก้ role ของตัวเองไม่ได้ จึงเลื่อนสิทธิ์ตัวเองไม่ได้
3.	เฉพาะแพทย์เท่านั้นที่ยืนยันใบสั่งได้ และค่ากำลังเลนส์ต้องอยู่ในช่วงที่รุ่นนั้นมีจริง บังคับด้วย trigger ไม่ใช่แค่ตรวจที่หน้าจอ
4.	ไม่มี policy สำหรับ DELETE บน orders โดยตั้งใจ ใบสั่งยกเลิกได้แต่ลบไม่ได้ เพื่อรักษาร่องรอยการตรวจสอบ
5.	การเขียน ai_runs ทำผ่าน service role ในฝั่งเซิร์ฟเวอร์เท่านั้น ผู้ใช้ทั่วไปอ่านและตรวจทานได้แต่สร้างเองไม่ได้
6.	ค่าตั้งค่าใน app_settings แถวที่มี user_id แก้ได้เฉพาะเจ้าของ ส่วนแถวระดับระบบ (user_id เป็น null) ทุกคนอ่านได้แต่แก้ได้เฉพาะ admin
7.	ไฟล์ใน Storage แยกสิทธิ์ตาม bucket — ใบ biometry เข้าถึงได้เฉพาะบุคลากรโรงพยาบาล ส่วนสำเนาใบสั่ง PDF เก็บในโฟลเดอร์ตาม company_id เพื่อให้บริษัทอ่านได้เฉพาะของตนเอง
