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
python3 -m myenv venv
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
**http://<EC2\_PUBLIC\_IP>:8000**

---

## 9. Create `index.html`

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>Music Albums</title>
  <style>
    body {
      font-family: Arial, sans-serif;
      padding: 20px;
      background-color: #f4f4f4;
    }
    table {
      width: 100%;
      border-collapse: collapse;
      margin-top: 20px;
      background: #fff;
    }
    th, td {
      border: 1px solid #ccc;
      padding: 8px 12px;
      text-align: center;
    }
    th {
      background-color: #333;
      color: white;
    }
    input, button {
      margin: 5px;
      padding: 8px;
    }
    .form-section {
      margin-top: 30px;
      background: #fff;
      padding: 15px;
      border-radius: 5px;
    }
  </style>
</head>
<body>
  <h1>🎵 Musical Albums</h1>

  <table id="albumTable">
    <thead>
      <tr>
        <th>Position</th>
        <th>Artist</th>
        <th>Album Name</th>
        <th>Label</th>
        <th>Year</th>
        <th>Critic</th>
      </tr>
    </thead>
    <tbody></tbody>
  </table>

  <div class="form-section">
    <h3>Add Album</h3>
    <input type="number" id="position" placeholder="Position" />
    <input type="text" id="artist" placeholder="Artist" />
    <input type="text" id="album_name" placeholder="Album Name" />
    <input type="text" id="label" placeholder="Label" />
    <input type="number" id="year" placeholder="Year" />
    <input type="text" id="critic" placeholder="Critic" />
    <button onclick="addAlbum()">Add</button>
  </div>

  <div class="form-section">
    <h3>Delete Album</h3>
    <input type="number" id="delete_position" placeholder="Position to delete" />
    <button onclick="deleteAlbum()">Delete</button>
  </div>

  <script>
    async function loadAlbums() {
      const res = await fetch("http://54.169.220.20:8000/albums");
      const data = await res.json();
      const tbody = document.querySelector("#albumTable tbody");
      tbody.innerHTML = "";
      data.forEach(album => {
        const row = document.createElement("tr");
        album.forEach(item => {
          const cell = document.createElement("td");
          cell.textContent = item;
          row.appendChild(cell);
        });
        tbody.appendChild(row);
      });
    }

    async function addAlbum() {
      const payload = {
        position: Number(document.getElementById("position").value),
        artist: document.getElementById("artist").value,
        album_name: document.getElementById("album_name").value,
        label: document.getElementById("label").value,
        year: Number(document.getElementById("year").value),
        critic: document.getElementById("critic").value
      };

      const res = await fetch("http://54.169.220.20:8000/add", {
        method: "POST",
        headers: {
          "Content-Type": "application/json"
        },
        body: JSON.stringify(payload)
      });

      const msg = await res.json();
      alert(msg.message);
      loadAlbums();
    }

    async function deleteAlbum() {
      const position = Number(document.getElementById("delete_position").value);

      const res = await fetch("http://54.169.220.20:8000/delete", {
        method: "POST",
        headers: {
          "Content-Type": "application/json"
        },
        body: JSON.stringify({ position })
      });

      const msg = await res.json();
      alert(msg.message);
      loadAlbums();
    }

    // Initial load
    loadAlbums();
  </script>
</body>
</html>
```

---

## 10. Host HTML on S3

1. Go to AWS Console → S3 → **Create Bucket**

   * Name: `laziza3tbc`
   * Uncheck: “Block all public access”

2. Enable **Static Website Hosting**

   * Index document: `index_laziza.html`

3. Upload your `index_laziza.html`

4. Go to **Permissions** → **Bucket Policy** and add:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:::laziza3tbc"
        },
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:*",
            "Resource": "arn:aws:s3:::laziza3tbc/*"
        }
    ]
}
```

---

## 11. Final Test

Visit:
**http://laziza3tbc.s3-website-ap-southeast-1.amazonaws.com**

Try all buttons:

* **Add Album**
* **Delete Album**

---

