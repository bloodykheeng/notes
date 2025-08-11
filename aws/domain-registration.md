Alright — I’ll merge everything you gave me into **clean, organized AWS deployment notes** and also include your YouTube links at the right places so you can reference them later.

---

# **Deploying on Amazon Web Services (AWS) – Step-by-Step Notes**

---

## **1. Registering a Domain with AWS Route 53**

📺 **Video:** [AWS Route 53 Domain Registration](https://youtu.be/W2fQFbkEQo0?si=CiCgGtKVe42tjLo1)

1. **Login** to your AWS Management Console.
2. **Search for** `Route 53`.

   * Route 53 is AWS’s service for **domain registration** and **traffic routing**.
3. In the left panel, click **Registered Domains** → **Register Domain**.
4. Search for the domain you want → **Add to Cart** → **Purchase**.
5. Once the domain is registered, go to **Hosted Zones**.

   * A **Hosted Zone** is created automatically.
   * **Public Hosted Zone** → Handles traffic from the internet.
   * **Private Hosted Zone** → Routes traffic only within a **VPC**.

---

## **2. Understanding DNS Records in AWS**

📺 **Video:** [AWS Route 53 Records & DNS Basics](https://youtu.be/JRZiQFVWpi8?si=kuYk4q-oVXUJulEY)

| Record Type     | Purpose                                             | Example                                                              |
| --------------- | --------------------------------------------------- | -------------------------------------------------------------------- |
| **A Record**    | Points hostname → IPv4                              | `aws.amazon.com → 205.251.242.103`                                   |
| **AAAA Record** | Points hostname → IPv6                              | `example.com → 2001:db8::ff00:42:8329`                               |
| **CNAME**       | Points hostname → another hostname                  | `amznwebsvcs.mywebsite.com → aws.amazon.com` (non-root domains only) |
| **ALIAS**       | AWS-specific record type, works for root & non-root | `mywebsite.com → website.s3.amazonaws.com`                           |

---

## **3. Elastic IP Setup**

Purpose: Assign a **persistent** IP to your EC2 instance.

1. Search for `EC2` → **Elastic IPs** under **Network & Security**.
2. **Allocate a new Elastic IP** → From Amazon’s IP pool → Name it.
3. Copy the IP → Go to Route 53 → Hosted Zone → Create A records:

   * **Root domain:** Name: (leave empty), Type: A, Value: Elastic IP
   * **www subdomain:** Name: `www`, Type: A, Value: Elastic IP

---

## **4. Security Groups** (AWS’s Firewall)

**a) Public Website Access Security Group**

* Name: `public-website-access`
* Opens ports **80 (HTTP)** and **443 (HTTPS)** from **Anywhere**.

**b) Dev SSH Access Security Group**

* Name: `dev-ssh-access`
* Opens port **22 (SSH)** from **My IP** only.

---

## **5. Creating an RDS MySQL Database**

1. Search for `RDS` → **Create Database**.
2. Settings:

   * Engine: MySQL 8
   * Template: Production
   * Instance: `db.t2.micro` (1GB)
   * Storage: 20GB SSD (auto-scaling optional)
   * **Multi-AZ Deployment:** Off (to save cost)
   * Public Access: No
3. After creation, find:

   * **Endpoint:** `database-1.ctmo6yc2y0jo.us-east-2.rds.amazonaws.com`
   * **Master Username:** `admin`
   * **Password:** Auto-generated (store securely)

---

## **6. Creating an EC2 Instance (Ubuntu Server)**

1. Search for `EC2` → **Launch Instance**.
2. AMI: **Ubuntu Server**
3. Instance Type: `t2.micro` (Free Tier)
4. Disable Auto-Assign Public IP (if using Elastic IP).
5. Attach Security Groups:

   * `public-website-access`
   * `dev-ssh-access`

---

✅ **End Result:**

* **Domain**: Registered in Route 53
* **Elastic IP**: Mapped to EC2 instance
* **Security Groups**: Configured for web and SSH access
* **RDS**: Separate MySQL database server
* **EC2**: Ubuntu server ready for deployment

---

Do you want me to also **extend these notes** so they include **the actual Linux commands** for connecting the EC2 instance to the RDS database and deploying a website? That would make it a full deployment guide.
