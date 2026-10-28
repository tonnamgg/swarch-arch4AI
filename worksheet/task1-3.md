# Tutorial 6 worksheet: serving patterns

**Name / Jirapat Dangkowkiew**

**Student ID / 6632025821**

This is **individual work**: each student completes and submits their own copy, in their own fork.

*งานนี้เป็น**งานเดี่ยว** นิสิตแต่ละคนทำและส่งงานของตัวเอง โดยใช้ forked repository ของตัวเอง*

Copy result rows from `results/runs.csv` (or the load test's output) into the tables. Keep answers short: one to three sentences each. Answer in Thai or English.

ตอนที่ตอบ ให้นิสิตคัดลอกผลจาก `results/runs.csv` (หรือจาก output ของ load test) ลงในตาราง ตอบสั้นๆ ข้อละ 1–3 ประโยค จะตอบเป็นภาษาไทยหรือภาษาอังกฤษก็ได้

## Task 0: Baseline (copy the baseline results from `results/runs.csv` here / copy ผลลัพธ์ของ baseline จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|
| t0-large |516|626|93.5%|100%|
| t0-small |84|175|72.5%|0%|

**Q0.** What does the large model buy, and what does it cost?

*large model ให้อะไรเพิ่ม และต้องแลกด้วยอะไร*

> Large model ให้ความแม่นยำเพิ่มขึ้นจาก 72.5% เป็น 93.5%  แต่ต้องแลกด้วย latency ที่สูงกว่ามาก: p50 ช้าลงประมาณ 6 เท่า (84 → 516 ms) และ p95 ช้าลงประมาณ 3.6 เท่า (175 → 626 ms) นอกจากนี้ทุก request ยังต้องเรียก large model (100% large calls) ซึ่งทำให้ค่า compute ต่อ request สูงขึ้นด้วย



## Task 1: Fallback prediction (copy the Task 1 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 1 จาก `results/runs.csv` มาวางที่นี่)

| Run | run time (s) | p50 ms | p95 ms | errors | pending | accuracy |
|---|---|---|---|---|---|---|
| t1-before |28.4 s|358|526|17|0|93.4%|
| t1-after |10.1 s|428|1039|0|19|93.4%|

**Q1.1** In `t1-before`, p50 can look *better* than the baseline. Why is p50 misleading here? Which column tells the real story?

*ใน `t1-before` ค่า p50 อาจดู "ดีกว่า" baseline ทำไม p50 จึงทำให้เข้าใจผิดในกรณีนี้ และ column ไหนบอกความจริง*

> p50 ดูดีกว่า baseline 358 vs 516 ms เพราะคำนวณจาก request ที่สำเร็จเท่านั้น ส่วน request ที่ค้างจนกลายเป็น error 17 ตัวไม่ถูกนับรวม และ median ก็มองไม่เห็นหางของการกระจายอยู่แล้ว column ที่บอกความจริงคือ errors และ run time 28.4 s เทียบกับ 10.1 s ซึ่งแสดงว่ามี request ล้มเหลวและ throughput ของระบบทั้งระบบตกลงมาก

**Q1.2** The hang affects only 10% of large-model calls. Why is it worse for the system than 10% of reviews being slow? Think about what a waiting request holds on to in the review service and the gateway.

*การค้างเกิดกับ calls ไปยัง large model แค่ 10% ทำไมจึงส่งผลต่อระบบแย่กว่าการที่รีวิว 10% ช้าลงเฉยๆ ลองคิดว่า request ที่รออยู่ถือทรัพยากรอะไรไว้ใน review service และ gateway*

> request ที่ค้างอยู่ยังถือ worker/thread หรือ concurrency slot ใน review service ไว้ และถือ connection ที่เปิดค้างไว้ใน gateway ด้วย เมื่อ slot ถูก call ที่ค้างจับไว้จนเต็ม request ปกติที่เหลืออีก 90% ก็ต้องเข้าคิวรอตาม ผลคือระบบช้าหรือ error ไปทั้งหมด ไม่ได้ช้าแค่ 10% ที่ค้าง


**Q1.3** Pending reviews have no label. Suggest what should happen to them, and name the topology from the lecture that does it.

*รีวิวที่ pending ยังไม่มี label ควรจัดการกับรีวิวเหล่านี้อย่างไร และ topology ใดจาก lecture ที่ทำแบบนั้น*

> รีวิวที่ pending ไม่ควรถูกทิ้งหรือปล่อยไว้โดยไม่มี label ควรเก็บไว้แล้วส่งให้ large model ทำนายซ้ำทีหลังแบบเป็นรอบ เช่น ทุกไม่กี่นาทีหรือตอนกลางคืน จากนั้นจึงเติม label กลับเข้าไปในฐานข้อมูล topology ที่ทำแบบนี้คือ Batch processing ซึ่ง เป็นการประมวลผลข้อมูลจำนวนมากเป็นรอบ ไม่ต้องตอบแบบ real-time จึงไม่มี request ใดต้องรอและถือทรัพยากรค้างไว้ใน review service หรือ gateway


**Q1.4** We chose a 1-second timeout. What goes wrong if it is 0.1 s? And if it is 10 s?

*เราเลือก timeout 1 วินาที จะเกิดปัญหาอะไรถ้าตั้งเป็น 0.1 วินาที และถ้าตั้งเป็น 10 วินาที*

> 0.1 s: สั้นกว่าเวลาปกติของ large model (p50 516 ms, p95 626 ms) ทำให้ call ที่ทำงานปกติก็ถูกตัดเกือบทั้งหมด รีวิวเกือบทุกอันจึงกลายเป็น pending ต้องรอ batch ทีหลัง ทำให้ไม่ได้ label ทันทีและเปลือง compute ที่เรียกไปแล้วทิ้ง

> 10 s: call ที่ค้างจะถือ worker/connection ไว้นานถึง 10 วินาที slot จึงเต็มและ request ปกติต้องเข้าคิวรอเหมือนกรณี t1-before ทำให้ timeout แทบไม่ช่วยอะไร


## Task 2: Cascade (copy the Task 2 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 2 จาก `results/runs.csv` มาวางที่นี่)

| Run | threshold | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|---|
| t2-0.6 | 0.6 |40|536|91.5%|38%|
| t2-0.7 | 0.7 |351|583|93.0%|53%|
| t2-0.8 | 0.8 |417|564|93.5%|64%|
| t2-0.9 | 0.9 |450|590|93.5%|77%|

**Q2.1** Which threshold would you deploy for CampusEats, and why? Name the quality you give up.

*จะเลือก threshold ใดสำหรับ deploy ใน CampusEats เพราะอะไร และต้องยอมเสีย quality ใดไป*

> เลือก **0.7** เพราะได้ accuracy 93.0% ซึ่งต่างจากใช้ large model ทุกครั้ง (93.5%) แค่ 0.5% แต่ p50 เร็วขึ้นจาก 516 เป็น 351 ms และเรียก large model แค่ 53% ส่วน 0.8 และ 0.9 ได้ accuracy เพิ่มอีกนิดเดียวแต่ต้องเรียก large model มากขึ้นเยอะ (64% และ 77%) สิ่งที่ยอมเสียคือ **accuracy เล็กน้อย (0.5%)** เพื่อแลกกับ latency และ cost ที่ลดลง


**Q2.2** Look at p50 and p95 for your threshold. Why does the cascade help p50 much more than p95?

*ดูค่า p50 และ p95 ของ threshold ที่เลือก ทำไม cascade จึงช่วย p50 ได้มากกว่า p95 มาก*

> ที่ 0.7 ค่า p50 ลดลง 165 ms (516 → 351) แต่ p95 ลดลงแค่ 43 ms (626 → 583) เพราะรีวิวที่ small model มั่นใจจะจบเร็วมาก จึงดึงค่ากลางลงมาได้ แต่รีวิวที่ไม่มั่นใจต้องรอ small แล้วต่อด้วย large ซึ่งช้าที่สุด และมีจำนวนเกิน 5% แน่นอน ค่า p95 จึงยังเป็นรีวิวกลุ่มนี้ และไม่มีทางเร็วกว่า large model ได้

**Q2.3** The cascade trusts the small model's confidence. Give one situation where that trust is misplaced, and its effect on users.

*cascade เชื่อ confidence ของ small model ยกตัวอย่างหนึ่งสถานการณ์ที่ไม่ควรเชื่อ และผลที่เกิดกับผู้ใช้*

> รีวิวประชด เช่น "ดีมาก รอแค่ชั่วโมงเดียว อาหารเย็นชืดพอดี" small model อาจเห็นคำว่า "ดีมาก" แล้วตอบว่าเป็นรีวิวบวกด้วยความมั่นใจสูง รีวิวนี้จึงไม่ถูกส่งต่อให้ large model ตรวจ ผลคือ label ผิดโดยไม่มีใครรู้ ผู้ใช้เห็นคะแนนร้านดีเกินจริงและอาจสั่งอาหารจากร้านที่มีปัญหา

## Task 3: Cached predictions (copy the Task 3 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 3 จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls | cache hits |
|---|---|---|---|---|---|
| t3-cache |43|519|93.0%|38%|35%|

**Q3.1** Which reviews benefit from the cache, and which never will?

*รีวิวแบบไหนได้ประโยชน์จาก cache และแบบไหนไม่มีวันได้ประโยชน์*

> รีวิวสั้นๆ ที่ซ้ำกันบ่อย เช่น "อร่อย", "ดีมาก", "good" จะได้ประโยชน์ เพราะข้อความเดียวกันเคยถูกทำนายไปแล้ว จึงตอบจาก cache ได้ทันทีโดยไม่ต้องเรียกโมเดลอีก ส่วนรีวิวยาวที่เขียนรายละเอียดเฉพาะตัวจะไม่มีวันได้ประโยชน์ เพราะข้อความไม่ซ้ำกับรีวิวอื่นเลย ต่างกันแค่ตัวอักษรเดียวหรือ emoji เดียวก็ถือว่าเป็น key ใหม่แล้ว

**Q3.2** Next week you deploy a better large model. What must happen to the cache, and how would you design for it?

*สัปดาห์หน้าจะ deploy large model ตัวใหม่ที่ดีกว่าเดิม ต้องทำอะไรกับ cache และจะออกแบบรองรับเรื่องนี้อย่างไร*

> cache เก็บคำตอบของโมเดลตัวเก่าไว้ ถ้าไม่ล้าง ระบบจะยังตอบด้วยผลเก่าที่ด้อยกว่าต่อไป จึงต้องล้าง cache เดิม วิธีออกแบบคือ **ใส่ model version ไว้ใน cache key** เช่น `hash(ข้อความ) + model_v2` เมื่อ deploy โมเดลใหม่ key ก็เปลี่ยนเองโดยอัตโนมัติและไม่หยิบผลเก่ามาใช้ อาจตั้ง TTL ให้ข้อมูลเก่าหมดอายุไปเอง และ pre-warm cache ด้วยการรันรีวิวที่พบบ่อยผ่านโมเดลใหม่แบบ batch ล่วงหน้า เพื่อไม่ให้ช่วงแรกหลัง deploy ช้าลงเพราะ cache ว่าง

## Debrief

**Q4.** Name one thing you learned from these measurements that the lecture did not cover.

*ยกตัวอย่างหนึ่งสิ่งที่ได้เรียนรู้จากการวัดผลครั้งนี้ ที่ไม่มีพูดถึงใน lecture*

> lecture บอกว่า cascade ช่วยลดต้นทุน inference แต่จากการวัดพบว่า cascade ช่วยแค่ **p50** แทบไม่ช่วย **p95** เลย เพราะรีวิวที่ถูกส่งต่อต้องรอทั้ง small และ large ดังนั้นถ้าระบบมี SLA ที่กำหนดด้วย p95 การใช้ cascade อย่างเดียวจะไม่พอ
