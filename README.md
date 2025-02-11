# demty2025-aws-networking

The following document describes the process to setup a public webpage that connects to a private backend.

*The proof of AWS courses can be found in the directory `aws_courses`*

The resulting network will look something like this:

![Diagrama VPC](https://github.com/user-attachments/assets/fe2739c6-6e62-4a0a-81a6-65e8dff99ca3)


**1. VPC, Subnet, Routing table and Internet Gateway**

The first step is to create a new VPC. This VPC will serve as the main container of our components, allowing to interconnect between them.

When creating the VPC, it's necessary to select the option `VPC and more`, to also create the routing tables, the public and private subnets and the Internet Gateway.

**VPC Creation**
![Imagen creacion vpc](https://github.com/user-attachments/assets/9c2131e0-f9b5-4cd7-bbed-4c3a85dae0fe)

**Resulting VPC Preview**
![Preview VPC](https://github.com/user-attachments/assets/c8dc9770-a8b3-49e9-b3f4-1fa5eab43f83)

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

**Network configuration in EC2**

![Imagen networking ec2](https://github.com/user-attachments/assets/48962eb7-b726-4f14-bd6c-a452a9ae7561)

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
    fetch("http://52.53.126.198/api/hello")  // Ahora usa la IP pública de la instancia pública
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
pip install mysql-connector-python
```

**Create API**
```
sudo vi /home/ubuntu/main.py
```

The content should be the following:

*Keep in mind that this API is a simple demo and that any sensible info should be stored safely as secrets or env variables.*

```python
from flask import Flask, jsonify
from flask_cors import CORS
import mysql.connector
import os

app = Flask(_name_)
CORS(app)

def get_db_connection():
    return mysql.connector.connect(
        host=os.env["DB_HOST"]",
        user=os.env["DB_USER"],
        password=os.env["DB_PASS"],
        database=os.env["DB_NAME"]
    )

@app.route('/api/usuarios', methods=['GET'])
def obtener_usuarios():
    conn = get_db_connection()
    cursor = conn.cursor(dictionary=True)
    cursor.execute("SELECT * FROM empleados  LIMIT 5;")
    usuarios = cursor.fetchall()
    conn.close()

if _name_ == '_main_':
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
        proxy_pass http://127.0.0.1:8080;
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

** Working API **
![API](https://github.com/user-attachments/assets/25bf633f-c68e-4e33-92fe-235e627759b7)

** Public EC2 consuming API **
![Consuming API from public EC2](https://github.com/user-attachments/assets/c26b8db9-874b-437c-854b-ba5c5ec4ff27)


