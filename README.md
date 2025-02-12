# VPC_exercise


# demty2025-aws-networking
 
The following document describes the process to setup a public webpage that connects to a private backend.
 
The resulting network will look something like this:
 

![WhatsApp Image 2025-02-11 at 5 36 38 PM](https://github.com/user-attachments/assets/7b07b659-73c3-4217-bcf3-1a4a61b11108)

 
**1. VPC, Subnet, Routing table and Internet Gateway**
 
The first step is to create a new VPC. This VPC will serve as the main container of our components, allowing to interconnect between them.
 
When creating the VPC, it's necessary to select the option `VPC and more`, to also create the routing tables, the public and private subnets and the Internet Gateway.
 

 ![WhatsApp Image 2025-02-11 at 5 33 50 PM (3)](https://github.com/user-attachments/assets/b240a24b-8817-4bd4-b196-a15eaef486ab)

![WhatsApp Image 2025-02-11 at 5 33 50 PM](https://github.com/user-attachments/assets/2584d34a-228f-49c5-b5fe-a35dd15dd9b9)

 
**2. Security Groups**
 
In order to restrict certain access to our EC2 instances, we need to create 2 new security groups, one for public access and a second one for private access.
 
- **Private SG:**
    This SG should have the following inbound and outbund rules:
    - **Inbound rules:**
 
        - HTTPS on port 443
        - TCP on port 8080 (for the API)
        - HTTP on port 80
        - SSH on port 22
   
    - **Outbund rules:**
 
        - All traffic is allowed
 
- **Public SG:**
    This SG should have the following inbound and outbund rules:
    - **Inbound rules:**
 
        - HTTP on port 80
        - SSH on port 22
   
    - **Outbund rules:**
 
        - All traffic is allowed
 
**3. EC2 instances**
 
**Network settings**
Once the security groups are ready, we can now create 2 new EC2 instances, a public and a private one.
 
When creating the instances, its necessary to select the newly created VPC and the existing security groups, just make sure one instance has the private SG and the other one has the public SG.
 

 ![WhatsApp Image 2025-02-11 at 5 33 50 PM (2)](https://github.com/user-attachments/assets/908ad6c7-bad2-4a58-9079-bf0d2f06f536)

 
**Key pairs**
 
Each EC2 should generate a new key pair in `.pem` format to be able to connect via SSH to both of them.
 
 
**IP Address**
 
After creating the public EC2 instance, we need to add an IP to be able to connect to it from the internet. To do so, go to `Elastic IPs` and and `Allocate Elastic IP address`.
 
When allocating the IP, you will be able to associate it with your public EC2 instance.
 
 
**4. Connecting to the public EC2 instance and configure NGIX**
 
Directly from the EC2 instances page, connect to the public instance, and once inside, execute the following commands that will allow us to install NGINX and setup a basic web page:
 
**Update apt-get**
```
sudo apt update
```
 
**Install NGINX**
```
sudo apt install nginx
```
 
**Configue NGINX to point to the private EC2**
```
sudo vi /etc/nginx/sites-available/default
```
 
This should be the content of the file
```bash
server {
    listen 80;
    server_name _;
 
    location / {
        root /var/www/html;
        index index.html;
    }
 
    location /api/ {
        proxy_pass http://{EC2 PRIVATE IP}:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```
 
**Create the webpage**
```
sudo vi /var/www/html/index.html
```
 
This should be the content of the file
 
```html
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>API desde Nginx</title>
</head>
<body>
    <h1>Hola desde Nginx</h1>
    <p id="mensaje">Cargando mensaje desde la API...</p>
<script>
    fetch("http://[IP PUBLICA_PUBLICA]/api/hello")  // Ahora usa la IP pública de la instancia pública
    .then(response => response.json())
    .then(data => {
        document.getElementById("mensaje").innerText = data.message;
    })
    .catch(error => console.error("Error:", error));
</script>
</body>
</html>
```
 
**Restart NGINX to reflect changes**
```
sudo systemctl restart nginx
```
 
**5. Connecting to the private EC2 and configuring the API**
 
Unlike the public EC2 instance, the private instance does not have a public IP, which makes it unnaccesible from the internet. So, in order to connect to it, we must use our public instance as a "bridge".
 
To do so, we need to copy the key `.pem` file of the private EC2 instance into the public one:
 
**Copy the private instance's key pair into the public instance**
 
```
scp -i public-instance-key.pem private-instance.pem public-instance-user@public-instance-ip4-dns:/home/public-instance-user
```
 
**Connect to the private instance**
 
Change the permissions for the copied key file
```
chmod 400 "private-instance-key.pem"
```
 
**Connect via SHH**
```
ssh -i private-instance-key.pem user@instance-endpoint
```
 
After connecting via SSH, we are effectively using the public instance as "bridge" to the private instance.
 
Now, what's left to do is configure NGINX and the API to serve our data.
 
**Creating the API**
 
Create a Python virtual environment and install the following:
```
pip install Flask
pip install flask-cors
```
 
**Create API**
```
sudo vi /home/ubuntu/main.py
```
 
The content should be the following:
 
```python
from flask import Flask, jsonify
from flask_cors import CORS
import mysql.connector

app = Flask(name)
CORS(app)

def get_db_connection():
    return mysql.connector.connect(
        host=["database-host"],
        user=["usuario"],
        password="*****",
        database=["db_name]"
    )

@app.route('/api/usuarios', methods=['GET'])
def obtener_usuarios():
    conn = get_db_connection()
    cursor = conn.cursor(dictionary=True)
    cursor.execute("SELECT * FROM ['TABLA']  LIMIT 5;")
    usuarios = cursor.fetchall()
    conn.close()
  

if name == 'main':
    app.run(host='0.0.0.0', port=8080)
```
 
**Update NGINX to serve our API**
```
sudo vi /etc/nginx/sites-available/default
```
 
The content should be:
 
```
server {
    listen 80;
    server_name _;
 
    location /api {
        proxy_pass http://IP:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```
 
**Run the API**
```
nohup python3 main.py > flask.log 2>&1 &
ps aux | grep python
```
 
**Restar NGINX to reflect the changes**
```
sudo systemctl restart nginx
```

**Received message to ensure everything is working well**
![WhatsApp Image 2025-02-11 at 5 33 50 PM (1)](https://github.com/user-attachments/assets/59c67627-baff-4a65-8638-91f1402fef48)


**CONCLUSION**

The final result shows us how networking is one of the most important topic that we should know and learn because it is involve in every task or tool into AWS, and provide us security in our projects and data.
