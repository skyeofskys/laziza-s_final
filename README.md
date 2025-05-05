# laziza-s_final

rds: https://ap-southeast-1.console.aws.amazon.com/rds/home?region=ap-southeast-1#database:id=dblaziza-3t;is-cluster=false 
ec2: https://ap-southeast-1.console.aws.amazon.com/ec2/home?region=ap-southeast-1#InstanceDetails:instanceId=i-0a4aa44eea5e913e2
s3: https://ap-southeast-1.console.aws.amazon.com/s3/buckets/laziza3tbc?region=ap-southeast-1&bucketType=general&tab=objects

**Laziza's Musical Albums Dashboard**

This project is a web application that displays musical album data from a PostgreSQL database hosted on AWS RDS. The backend is built using Flask and deployed on an EC2 instance. The frontend is a simple HTML file hosted on Amazon S3.

1. Set up EC2 Instance

Go to AWS Console → EC2 → Launch Instance
Name: webapp_laziza
AMI: Ubuntu
Instance type: t2.micro
Key pair: Create or use an existing .pem file (e.g., laziza_key.pem)

## 1. Set up EC2 Instance

1. Go to AWS Console → EC2 → **Launch Instance**

   * **Name**: `webapp_laziza`
   * **AMI**: Ubuntu
   * **Instance type**: t2.micro
   * **Key pair**: Create or use an existing `.pem` file (e.g., `lazizawho.pem`)

2. Configure **Security Group** (Inbound Rules):

| Type       | Protocol | Port Range | Source    | Purpose           |
| ---------- | -------- | ---------- | --------- | ----------------- |
| SSH        | TCP      | 22         | Your IP   | SSH access to EC2 |
| HTTP       | TCP      | 80         | 0.0.0.0/0 | Web server access |
| Custom TCP | TCP      | 8000       | 0.0.0.0/0 | Access Flask API  |
| PostgreSQL | TCP      | 5432       | EC2 SG    | Allow EC2 to RDS  |

Click **Launch Instance**.

---

## 2. Set up RDS PostgreSQL Database

1. Go to AWS Console → RDS → **Create database**

   * **Creation method**: Standard Create
   * **Engine**: PostgreSQL
   * **DB identifier**: `db_laziza`
   * **Master username**: `postgres`
   * **Password**: `postgres`
   * **Public access**: Yes

2. In **Connectivity**:

   * **VPC**: default
   * **Subnet group**: default
   * **Security group**: same as EC2 instance SG

Click **Create Database**

3. Go to RDS → Databases → `db_laziza` → **Connectivity & Security** → Edit **Inbound Rules** of VPC security group:

| Type       | Protocol | Port | Source (EC2 SG ID) | Purpose                 |
| ---------- | -------- | ---- | ------------------ | ----------------------- |
| PostgreSQL | TCP      | 5432 | EC2 Security Group | Allow EC2 to access RDS |

---

## 3. Import Data Using DBeaver

1. Connect to RDS:

   * Host: `dblaziza-3t.c32geugqgvkj.ap-southeast-1.rds.amazonaws.com`
   * Database: `postgres`
   * User: `postgres`
   * Password: `postgres`

2. Import your CSV:

   * Table name: `tbl_laziza_musicalbums_data`
   * Columns: `position`, `artist`, `album_name`, `label`, `year`, `critic`
   * Set appropriate data types.

---

## 4. SSH into EC2

```bash
ssh -i "/path/to/laziza_key.pem" ubuntu@<EC2_PUBLIC_IP>
```

---

## 5. Set Up Flask App

```bash
sudo apt update
sudo apt install python3-pip python3.10-venv -y
```

---

## 6. Project Setup

```bash
mkdir webapp_laziza
cd webapp_laziza
python3 -m venv venv
source myenv/bin/activate
pip install flask flask-cors psycopg2-binary
```

---

## 7. Create `app.py`

```python
from flask import Flask, jsonify, request
import psycopg2
from flask_cors import CORS

app = Flask(__name__)
CORS(app)

conn = psycopg2.connect(
    host='dblaziza-3t.c32geugqgvkj.ap-southeast-1.rds.amazonaws.com',
    user='postgres',
    password='postgres',
    dbname='postgres'
)

@app.route('/albums', methods=['GET'])
def get_videos():
    with conn.cursor() as cursor:
        cursor.execute("SELECT * FROM tbl_laziza_musicalbums_data;")
        data = cursor.fetchall()
    return jsonify(data)
    
@app.route('/add', methods=['POST'])
def add_video():
    data = request.json
    with conn.cursor() as cursor:
        cursor.execute("""
            INSERT INTO tbl_laziza_musicalbums_data (position, artist, album_name, label, year, critic)
            VALUES (%s, %s, %s, %s, %s, %s);
        """, (
            data.get('position'),
            data.get('artist'),
            data.get('album_name'),
            data.get('label'),
            data.get('year'),
            data.get('critic'),
        ))
        conn.commit()
    return jsonify({'message': 'Album added successfully'})

@app.route('/delete', methods=['POST'])
def delete_video():
    position = request.json.get('position')
    with conn.cursor() as cursor:
        cursor.execute("DELETE FROM tbl_laziza_musicalbums_data WHERE position = %s;", (position,))
        conn.commit()
    return jsonify({'message': f'Album with position {position} deleted'})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8000)
```

---

## 8. Run the Flask App

```bash
python app.py
```

Your API will be running on:
**http\://\<EC2\_PUBLIC\_IP>:8000**

---

## 9. Create `index.html`

```html
<!DOCTYPE html>
<html>
<head>
  <title>Musical Albums</title>
  <script>
    async function loadData() {
      const res = await fetch("http://<EC2_Public_IP>:8000/albums");
      const data = await res.json();
      let table = "<table border='1'><tr><th>Position</th><th>Artist</th><th>Album</th><th>Label</th><th>Year</th><th>Critic</th></tr>";
      for (let row of data) {
        table += `<tr><td>${row[0]}</td><td>${row[1]}</td><td>${row[2]}</td><td>${row[3]}</td><td>${row[4]}</td><td>${row[5]}</td></tr>`;
      }
      table += "</table>";
      document.getElementById("result").innerHTML = table;
    }

    async function addAlbum() {
      await fetch("http://<EC2_Public_IP>:8000/add", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          position: 999,
          artist: "Laziza Test",
          album_name: "Demo Album",
          label: "Demo Label",
          year: 2025,
          critic: "Demo Critic"
        })
      });
      loadData();
    }

    async function deleteAlbum() {
      await fetch("http://<EC2_Public_IP>:8000/delete", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ position: 999 })
      });
      loadData();
    }
  </script>
</head>
<body>
  <h1>Musical Albums Dashboard</h1>
  <button onclick="loadData()">Load Albums</button>
  <button onclick="addAlbum()">Add Demo Album</button>
  <button onclick="deleteAlbum()">Delete Demo Album</button>
  <div id="result"></div>
</body>
</html>
```

---

## 10. Host HTML on S3

1. Go to AWS Console → S3 → **Create Bucket**

   * Name: `musical-dashboard-laziza`
   * Uncheck: “Block all public access”

2. Enable **Static Website Hosting**

   * Index document: `index.html`

3. Upload your `index.html`

4. Go to **Permissions** → **Bucket Policy** and add:

```json
{
  "Version":"2012-10-17",
  "Statement":[{
    "Effect":"Allow",
    "Principal":"*",
    "Action":"s3:GetObject",
    "Resource":"arn:aws:s3:::musical-dashboard-laziza/*"
  }]
}
```

---

## 11. Final Test

Visit:
**[http://musical-dashboard-laziza.s3-website-](http://musical-dashboard-laziza.s3-website-)<region>.amazonaws.com**

Try all buttons:

* **Load Albums**
* **Add Demo Album**
* **Delete Demo Album**

---

