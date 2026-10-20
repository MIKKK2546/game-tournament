# การออกแบบฐานข้อมูล

เอกสารนี้อธิบายตารางและคำสั่ง SQL (รูปแบบ MySQL) ตามแผนภาพใน [er-diagram.md](er-diagram.md)

## ข้อมูลที่เก็บจริงในเวอร์ชันปัจจุบัน

เว็บเวอร์ชันปัจจุบันยังไม่ได้ต่อฐานข้อมูล แต่เก็บข้อมูลเป็น JSON ในเบราว์เซอร์ (localStorage คีย์ `tourney.v1`) โครงสร้างสอดคล้องกับตารางด้านล่าง

```json
{
  "games":   [ { "id": "", "name": "", "img": "" } ],
  "teams":   [ { "id": "", "name": "", "img": "", "gameId": "",
                 "members": [ { "id": "", "name": "", "role": "", "img": "" } ] } ],
  "wheel":   [ "ชื่อในวงล้อ" ],
  "bsel":    [ "รหัสทีมที่เลือกเข้าสาย" ],
  "bracket": { "rounds": [ [ { "a": "ทีม A", "b": "ทีม B", "w": "ผู้ชนะ" } ] ] }
}
```

รูปภาพเก็บเป็นข้อความ (data URL) ที่ถูกย่อขนาดแล้ว ส่วนในฐานข้อมูลจริงแนะนำให้เก็บไฟล์ไว้ในโฟลเดอร์ `uploads/` และเก็บเฉพาะที่อยู่ไฟล์ในคอลัมน์ `image`

## ตารางในฐานข้อมูล

### 1. games (เกม)

```sql
CREATE TABLE games (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(100) NOT NULL,
  image VARCHAR(255)
);
```
เก็บชื่อเกมและที่อยู่ไฟล์รูปเกม

### 2. teams (ทีม)

```sql
CREATE TABLE teams (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(100) NOT NULL,
  image VARCHAR(255),
  game_id INT NOT NULL,
  FOREIGN KEY (game_id) REFERENCES games(id) ON DELETE RESTRICT
);
```
แต่ละทีมผูกกับเกมที่เล่นผ่าน `game_id` และ `RESTRICT` ทำให้ลบเกมไม่ได้ถ้ายังมีทีมใช้อยู่ ตรงกับการทำงานของเว็บ

### 3. members (สมาชิก)

```sql
CREATE TABLE members (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(100) NOT NULL,
  role VARCHAR(50),
  image VARCHAR(255),
  team_id INT NOT NULL,
  FOREIGN KEY (team_id) REFERENCES teams(id) ON DELETE CASCADE
);
```
สมาชิกผูกกับทีมผ่าน `team_id` และ `CASCADE` ทำให้ลบทีมแล้วสมาชิกถูกลบตามไปด้วย

### 4. brackets (สายแข่ง)

```sql
CREATE TABLE brackets (
  id INT PRIMARY KEY AUTO_INCREMENT,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```
หนึ่งแถวคือหนึ่งสายแข่งที่ถูกสร้างขึ้น

### 5. matches (คู่แข่ง)

```sql
CREATE TABLE matches (
  id INT PRIMARY KEY AUTO_INCREMENT,
  bracket_id INT NOT NULL,
  round_no INT NOT NULL,
  match_no INT NOT NULL,
  team_a_id INT NULL,
  team_b_id INT NULL,
  winner_team_id INT NULL,
  UNIQUE (bracket_id, round_no, match_no),
  FOREIGN KEY (bracket_id) REFERENCES brackets(id) ON DELETE CASCADE,
  FOREIGN KEY (team_a_id) REFERENCES teams(id) ON DELETE SET NULL,
  FOREIGN KEY (team_b_id) REFERENCES teams(id) ON DELETE SET NULL,
  FOREIGN KEY (winner_team_id) REFERENCES teams(id) ON DELETE SET NULL
);
```
เก็บคู่แข่งแต่ละคู่ ค่า `team_b_id` เป็นว่างเมื่อได้บาย และ `winner_team_id` เป็นว่างจนกว่าจะมีผล ส่วน `UNIQUE` กันไม่ให้ตำแหน่งคู่ซ้ำกันในสายเดียว

### 6. wheel_entries (ชื่อในวงล้อ)

```sql
CREATE TABLE wheel_entries (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(100) NOT NULL
);
```
เก็บรายชื่อที่อยู่ในวงล้อสุ่มตอนนี้

## ตัวอย่างคำสั่งค้นหา

### ดูทีมพร้อมชื่อเกมและจำนวนสมาชิก

```sql
SELECT t.name AS team, g.name AS game, COUNT(m.id) AS members
FROM teams t
JOIN games g ON g.id = t.game_id
LEFT JOIN members m ON m.team_id = t.id
GROUP BY t.id, t.name, g.name;
```
รวมตาราง 3 ตารางเข้าด้วยกัน แล้วนับสมาชิกของแต่ละทีม

### ดูผลคู่แข่งของสายแข่งหนึ่งสาย

```sql
SELECT mt.round_no, mt.match_no,
       a.name AS team_a, b.name AS team_b, w.name AS winner
FROM matches mt
LEFT JOIN teams a ON a.id = mt.team_a_id
LEFT JOIN teams b ON b.id = mt.team_b_id
LEFT JOIN teams w ON w.id = mt.winner_team_id
WHERE mt.bracket_id = 1
ORDER BY mt.round_no, mt.match_no;
```
ใช้ `LEFT JOIN` เพราะบางคู่ยังไม่มีทีมหรือยังไม่มีผู้ชนะ

### ดูแชมป์ของสายแข่ง

```sql
SELECT w.name AS champion
FROM matches mt
JOIN teams w ON w.id = mt.winner_team_id
WHERE mt.bracket_id = 1
ORDER BY mt.round_no DESC
LIMIT 1;
```
ผู้ชนะของรอบสุดท้าย (รอบที่เลขมากที่สุด) คือแชมป์
