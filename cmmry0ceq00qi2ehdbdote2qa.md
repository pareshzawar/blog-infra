---
title: "Oracle Database@AWS: What It Is and Why It Changes Everything"
seoTitle: "Oracle Database@AWS: Complete Setup & Architecture Guide"
seoDescription: "Set up Oracle Autonomous Database on AWS: networking, wallet config, JDBC connection, pricing breakdown, and migration options — with real CLI command"
datePublished: 2026-03-15T16:02:11.291Z
cuid: cmmry0ceq00qi2ehdbdote2qa
slug: oracle-database-aws-guide
cover: https://cdn.hashnode.com/uploads/covers/66070b822018a4a9c83bbb3a/7d5e1069-3fee-4d12-9caf-bb3542adcf75.svg
ogImage: https://cdn.hashnode.com/uploads/og-images/66070b822018a4a9c83bbb3a/9090363e-3228-4846-977c-5257337b8930.svg
tags: oracle, aws, databases, oracle-database, oracle-cloud

---

## **Why Oracle Finally Partnered With AWS (And Why It Matters)**

For decades, Oracle and AWS operated like rival kingdoms — you could run Oracle Database on an EC2 instance, but you were doing it yourself, unsupported, with all the infrastructure complexity that entails. In June 2023, that changed dramatically.

Oracle and AWS announced Oracle Database@AWS: a fully managed Oracle Database service running on dedicated Oracle Exadata infrastructure inside AWS data centers — but owned, operated, and supported entirely by Oracle. You manage it through the OCI console. You pay Oracle for the database. AWS never touches it.

This is not a marketing rebrand of RDS for Oracle. This is fundamentally different infrastructure with a fundamentally different support model. This guide explains exactly what it is, how the networking works, how to set it up step by step, and — critically — when you should and should not use it.

> **🎯 Who This Post Is For**
> 
> DBAs, cloud architects, and engineering leads evaluating whether to run Oracle Autonomous Database or Oracle Exadata in AWS environments without re-platforming to PostgreSQL or Aurora.

## **What Is Oracle Database@AWS, Exactly?**

Oracle Database@AWS is a managed service that provisions Oracle Database infrastructure — specifically Oracle Exadata Database Service and Oracle Autonomous Database — inside AWS Availability Zones. The key technical details:

**Infrastructure**: Oracle Exadata X9M hardware racks physically located inside AWS data centers

**Network connectivity:** Connected to your AWS VPC via a high-speed, low-latency private network link (sub-millisecond latency to EC2)

**Management plane:** OCI Console, OCI APIs, and OCI CLI — not the AWS Console

**Support:** Oracle Cloud Support — not AWS Support

**Billing:** Oracle Cloud billing — appears on your OCI invoice, not your AWS bill

**Identity:** OCI IAM for database users; AWS IAM for your applications via cross-service roles

**Regions:** Initially US East (N. Virginia), expanding to additional regions

### **What Databases Are Available?**

<table style="min-width: 100px;"><colgroup><col style="min-width: 25px;"><col style="min-width: 25px;"><col style="min-width: 25px;"><col style="min-width: 25px;"></colgroup><tbody><tr><td colspan="1" rowspan="1"><p><strong>Database Service</strong></p></td><td colspan="1" rowspan="1"><p><strong>Shape</strong></p></td><td colspan="1" rowspan="1"><p><strong>Use Case</strong></p></td><td colspan="1" rowspan="1"><p><strong>Min Config</strong></p></td></tr><tr><td colspan="1" rowspan="1"><p>Oracle Autonomous Database Serverless</p></td><td colspan="1" rowspan="1"><p>ECPU-based</p></td><td colspan="1" rowspan="1"><p>OLTP, Analytics, Mixed</p></td><td colspan="1" rowspan="1"><p>2 ECPUs, 1 TB Storage</p></td></tr><tr><td colspan="1" rowspan="1"><p>Oracle Autonomous Database on Dedicated Exadata</p></td><td colspan="1" rowspan="1"><p>Exadata X9M</p></td><td colspan="1" rowspan="1"><p>Enterprise OLTP</p></td><td colspan="1" rowspan="1"><p>Quarter Rack</p></td></tr><tr><td colspan="1" rowspan="1"><p>Oracle Exadata Database Service</p></td><td colspan="1" rowspan="1"><p>Exadata X9M</p></td><td colspan="1" rowspan="1"><p>Custom DB workloads</p></td><td colspan="1" rowspan="1"><p>Quarter Rack</p></td></tr><tr><td colspan="1" rowspan="1"><p>Oracle Base Database Service</p></td><td colspan="1" rowspan="1"><p>VM.Standard shapes</p></td><td colspan="1" rowspan="1"><p>Dev/Test, SMB</p></td><td colspan="1" rowspan="1"><p>1 OCPU, 256 GB</p></td></tr></tbody></table>

> **💡 Key Insight**
> 
> For most teams evaluating Oracle Database@AWS, the answer is Oracle Autonomous Database Serverless — it requires zero Exadata rack commitment, scales by the ECPU, and is the fastest to provision (under 5 minutes).

## **Network Architecture: How It Actually Connects**

Understanding the network topology is critical before you provision anything. Oracle Database@AWS does not create an endpoint inside your VPC in the traditional sense. Instead, it creates an OCI VCN (Virtual Cloud Network) that is peered into your AWS VPC through a dedicated interconnect.

### **The Three-Layer Network Model**

**Layer 1** — **Physical interconnect:**

Oracle and AWS maintain a dedicated high-speed network link between Oracle Exadata racks and AWS network infrastructure within the same data center campus. This is NOT traversing the public internet.

**Layer 2** — **OCI VCN:**

Your Oracle Database instances live inside an OCI VCN subnet. Oracle manages this entirely. You see it in the OCI Console.

**Layer 3** — **AWS VPC endpoint:**

Oracle provisions a VPC endpoint in your AWS account. Your EC2 instances, Lambda functions, ECS containers, and EKS pods connect to Oracle DB through this endpoint using standard Oracle Net (TCP/1521 or TCP/2484 for TLS).

### **Latency Characteristics**

<table style="min-width: 75px;"><colgroup><col style="min-width: 25px;"><col style="min-width: 25px;"><col style="min-width: 25px;"></colgroup><tbody><tr><td colspan="1" rowspan="1"><p><strong>Connection Type</strong></p></td><td colspan="1" rowspan="1"><p><strong>Typical Latency</strong></p></td><td colspan="1" rowspan="1"><p><strong>Notes</strong></p></td></tr><tr><td colspan="1" rowspan="1"><p>EC2 → Oracle DB@AWS (same AZ)</p></td><td colspan="1" rowspan="1"><p>&lt; 1 ms</p></td><td colspan="1" rowspan="1"><p>Co-located rack, dedicated interconnect</p></td></tr><tr><td colspan="1" rowspan="1"><p>EC2 → Oracle DB@AWS (cross-AZ)</p></td><td colspan="1" rowspan="1"><p>1–3 ms</p></td><td colspan="1" rowspan="1"><p>Still within same AWS region physical campus</p></td></tr><tr><td colspan="1" rowspan="1"><p>EC2 → RDS Oracle</p></td><td colspan="1" rowspan="1"><p>&lt; 1 ms</p></td><td colspan="1" rowspan="1"><p>Fully within AWS network</p></td></tr><tr><td colspan="1" rowspan="1"><p>EC2 → OCI DB (cross-cloud)</p></td><td colspan="1" rowspan="1"><p>30–80 ms</p></td><td colspan="1" rowspan="1"><p>FastConnect or public internet</p></td></tr><tr><td colspan="1" rowspan="1"><p>On-premises → Oracle DB@AWS</p></td><td colspan="1" rowspan="1"><p>5–50 ms</p></td><td colspan="1" rowspan="1"><p>Via AWS Direct Connect + OCI interconnect</p></td></tr></tbody></table>

![](https://cdn.hashnode.com/uploads/covers/66070b822018a4a9c83bbb3a/ba3fc5b9-ead0-4b51-8a9e-d2fa4cc6e6a9.svg align="center")

## **Prerequisites Before You Start**

Before provisioning Oracle Database@AWS, you need accounts and configurations in both OCI and AWS. This is genuinely the most confusing part for teams new to the service — you're operating across two cloud consoles simultaneously.

### **OCI Requirements**

●      An active OCI tenancy (trial or paid)

●      OCI user with tenancy-level privileges (or appropriate IAM policies)

●      OCI CLI installed and configured: oci --version should return 3.x+

●      OCI tenancy subscribed to the US East (Ashburn) or relevant region

●      Credit card or Oracle Universal Credits on file

### **AWS Requirements**

●      AWS account with permissions to create VPCs, subnets, security groups

●      AWS CLI configured: aws --version should return 2.x+

●      A VPC in us-east-1 (N. Virginia) with at least one private subnet

●      AWS account linked to OCI via the Oracle Database@AWS integration

### **Linking Your AWS Account to OCI**

The linking process is a one-time setup done in the AWS Marketplace. Navigate to AWS Marketplace → find 'Oracle Database@AWS' → subscribe. This creates the cross-account trust relationship.

> `# Verify OCI CLI is configured`
> 
> `oci iam region list --output table`
> 
> `# Verify AWS CLI is configured`
> 
> `aws sts get-caller-identity`
> 
> `# Output should show your AWS account ID`
> 
> `# {`
> 
> `#   "UserId": "AIDAXXXXXXXXXXXXXXXXX",`
> 
> `#   "Account": "123456789012",`
> 
> `#   "Arn": "arn:aws:iam::123456789012:user/your-user"`
> 
> `# }`

## **Step-by-Step: Provisioning Oracle Autonomous Database on AWS**

We'll provision an Oracle Autonomous Database Serverless instance — the quickest path to a running Oracle DB in AWS. The full process takes approximately 10–15 minutes once prerequisites are met.

### **Step 1: Create the Oracle Database@AWS Resource in OCI Console**

Navigate to the OCI Console → Oracle Database@AWS → Autonomous Database. Click 'Create Autonomous Database'.

<table style="min-width: 75px;"><colgroup><col style="min-width: 25px;"><col style="min-width: 25px;"><col style="min-width: 25px;"></colgroup><tbody><tr><td colspan="1" rowspan="1"><p><strong>Field</strong></p></td><td colspan="1" rowspan="1"><p><strong>Value</strong></p></td><td colspan="1" rowspan="1"><p><strong>Notes</strong></p></td></tr><tr><td colspan="1" rowspan="1"><p>Display Name</p></td><td colspan="1" rowspan="1"><p>prod-adb-aws-01</p></td><td colspan="1" rowspan="1"><p>Your choice — use naming convention</p></td></tr><tr><td colspan="1" rowspan="1"><p>Database Name</p></td><td colspan="1" rowspan="1"><p>PRODADB01</p></td><td colspan="1" rowspan="1"><p>Uppercase, 1-14 chars, no spaces</p></td></tr><tr><td colspan="1" rowspan="1"><p>Workload Type</p></td><td colspan="1" rowspan="1"><p>Transaction Processing</p></td><td colspan="1" rowspan="1"><p>Or Data Warehouse / JSON / APEX</p></td></tr><tr><td colspan="1" rowspan="1"><p>Deployment Type</p></td><td colspan="1" rowspan="1"><p>Serverless</p></td><td colspan="1" rowspan="1"><p>Shared Exadata — fastest to start</p></td></tr><tr><td colspan="1" rowspan="1"><p>Database Version</p></td><td colspan="1" rowspan="1"><p>23ai</p></td><td colspan="1" rowspan="1"><p>Latest — use this unless legacy app requires 19c</p></td></tr><tr><td colspan="1" rowspan="1"><p>ECPU Count</p></td><td colspan="1" rowspan="1"><p>2</p></td><td colspan="1" rowspan="1"><p>Minimum; auto-scales if enabled</p></td></tr><tr><td colspan="1" rowspan="1"><p>Storage</p></td><td colspan="1" rowspan="1"><p>1 TB</p></td><td colspan="1" rowspan="1"><p>Minimum; auto-scales if enabled</p></td></tr><tr><td colspan="1" rowspan="1"><p>Password</p></td><td colspan="1" rowspan="1"><p>OCI#Secure2026!</p></td><td colspan="1" rowspan="1"><p>ADMIN user — store in Secrets Manager</p></td></tr><tr><td colspan="1" rowspan="1"><p>AWS VPC</p></td><td colspan="1" rowspan="1"><p>vpc-xxxxxxxxx</p></td><td colspan="1" rowspan="1"><p>Your target VPC in us-east-1</p></td></tr><tr><td colspan="1" rowspan="1"><p>AWS Subnet</p></td><td colspan="1" rowspan="1"><p>subnet-xxxxxxxxx</p></td><td colspan="1" rowspan="1"><p>Private subnet recommended</p></td></tr><tr><td colspan="1" rowspan="1"><p>License Type</p></td><td colspan="1" rowspan="1"><p>License Included</p></td><td colspan="1" rowspan="1"><p>Or BYOL if you have Oracle licenses</p></td></tr></tbody></table>

### **Step 2: Retrieve the Connection Endpoint**

After provisioning (3–8 minutes), navigate to your ADB instance → DB Connection → Download Wallet. The wallet contains:

●      tnsnames.ora — connection string definitions

●      sqlnet.ora — network configuration

●      cwallet.sso and ewallet.p12 — SSL certificates

●      [ojdbc.properties](http://ojdbc.properties) — JDBC properties file

> `# Unzip wallet to a secure directory`
> 
> `mkdir -p ~/oracle-wallet/prod-adb`
> 
> `unzip Wallet_PRODADB01.zip -d ~/oracle-wallet/prod-adb`
> 
> `# Check available connection strings`
> 
> `cat ~/oracle-wallet/prod-adb/tnsnames.ora`
> 
> `# Output will show connection strings like:`
> 
> `# prodadb01_high = (description=(retry_count=20)(retry_delay=3)`
> 
> `#   (address=(protocol=tcps)(port=1522)`
> 
> `#   (host=adb.us-east-1.oraclecloud.com))`
> 
> `#   (connect_data=(service_name=xxxxx_prodadb01_high.adb.oraclecloud.com))`
> 
> `#   (security=(ssl_server_dn_match=yes)))`

**Step 3: Configure Security Groups**

Your EC2 instances need outbound access to the Oracle DB endpoint on port 1522 (TLS). Create a security group rule:

> `# Get the VPC endpoint DNS name from OCI Console`
> 
> `# Then create outbound rule for your application security group`
> 
> `aws ec2 authorize-security-group-egress \`
> 
> `--group-id sg-xxxxxxxxxxxxxxxxx \`
> 
> `--ip-permissions '[`
> 
> `{`
> 
> `"IpProtocol": "tcp",`
> 
> `"FromPort": 1522,`
> 
> `"ToPort": 1522,`
> 
> `"IpRanges": [{"CidrIp": "10.0.0.0/8", "Description": "Oracle DB@AWS endpoint"}]`
> 
> `}`
> 
> `]'`
> 
> `# For mutual TLS (recommended), also allow port 1521`
> 
> `aws ec2 authorize-security-group-egress \`
> 
> `--group-id sg-xxxxxxxxxxxxxxxxx \`
> 
> `--ip-permissions '[{"IpProtocol": "tcp", "FromPort": 1521, "ToPort": 1521,`
> 
> `"IpRanges": [{"CidrIp": "10.0.0.0/8"}]}]'`

### **Step 4: Test Connectivity from EC2**

SSH into an EC2 instance in the same VPC and run a connection test using SQLcl or SQL\*Plus:

> \# Install SQLcl on Amazon Linux 2023
> 
> sudo dnf install java-21-amazon-corretto -y
> 
> wget [https://download.oracle.com/otn\_software/java/sqldeveloper/sqlcl-latest.zip](https://download.oracle.com/otn_software/java/sqldeveloper/sqlcl-latest.zip)
> 
> unzip [sqlcl-latest.zip](http://sqlcl-latest.zip)
> 
> \# Set wallet location in environment
> 
> export TNS\_ADMIN=~/oracle-wallet/prod-adb
> 
> \# Connect to Autonomous Database
> 
> sql ADMIN/'OCI#Secure2026!'@prodadb01\_high
> 
> \# Expected output:
> 
> \# SQLcl: Release 24.3 Production on ...
> 
> \# Connected to:
> 
> \# Oracle Database 23ai Enterprise Edition
> 
> \# SQL>
> 
> \# Verify you are on Exadata infrastructure
> 
> SELECT platform\_name, version\_full FROM v$database d, v$instance i;

### **Step 5: Connect a Java Application via JDBC**

For production applications, use the Oracle JDBC driver with the wallet. Add the dependency to your pom.xml:

> `com.oracle.database.jdbc`
> 
> `ojdbc11`
> 
> `23.5.0.24.07`
> 
> `com.oracle.database.security`
> 
> `oraclepki`
> 
> `23.5.0.24.07`
> 
> `// Java connection example with wallet`
> 
> `import oracle.jdbc.pool.OracleDataSource;`
> 
> `OracleDataSource ods = new OracleDataSource();`
> 
> `ods.setURL("jdbc:oracle:thin:@prodadb01_high?TNS_ADMIN=/path/to/wallet");`
> 
> `ods.setUser("ADMIN");`
> 
> `ods.setPassword(System.getenv("ORACLE_DB_PASSWORD")); // Never hardcode!`
> 
> `try (Connection conn = ods.getConnection()) {`
> 
> `Statement stmt = conn.createStatement();`
> 
> `ResultSet rs = stmt.executeQuery("SELECT * FROM v$version");`
> 
> `while (rs.next()) {`
> 
> `System.out.println(rs.getString(1));`
> 
> `}`
> 
> `}`

## **Pricing: What It Actually Costs**

Oracle Database@AWS pricing has two components: the Oracle database service (ECPU + Storage + Options) and any AWS data transfer charges. There are no Oracle fees for the network interconnect itself.

### **Autonomous Database Serverless Pricing (us-east-1, April 2026)**

<table style="min-width: 100px;"><colgroup><col style="min-width: 25px;"><col style="min-width: 25px;"><col style="min-width: 25px;"><col style="min-width: 25px;"></colgroup><tbody><tr><td colspan="1" rowspan="1"><p><strong>Component</strong></p></td><td colspan="1" rowspan="1"><p><strong>Rate</strong></p></td><td colspan="1" rowspan="1"><p><strong>Example (2 ECPU, 1 TB, 730 hrs)</strong></p></td><td colspan="1" rowspan="1"><p><strong>Notes</strong></p></td></tr><tr><td colspan="1" rowspan="1"><p>ECPU (compute)</p></td><td colspan="1" rowspan="1"><p>$0.2688/ECPU/hr</p></td><td colspan="1" rowspan="1"><p>2 × $0.2688 × 730 = ~$393/mo</p></td><td colspan="1" rowspan="1"><p>Auto-scaling adds ECPUs on demand</p></td></tr><tr><td colspan="1" rowspan="1"><p>Storage</p></td><td colspan="1" rowspan="1"><p>$0.0230/GB/mo</p></td><td colspan="1" rowspan="1"><p>1,000 × $0.0230 = $23/mo</p></td><td colspan="1" rowspan="1"><p>Includes backups in same region</p></td></tr><tr><td colspan="1" rowspan="1"><p>Exadata Storage</p></td><td colspan="1" rowspan="1"><p>Included</p></td><td colspan="1" rowspan="1"><p>$0</p></td><td colspan="1" rowspan="1"><p>Smart Scan, Smart Flash Cache included</p></td></tr><tr><td colspan="1" rowspan="1"><p>BYOL discount</p></td><td colspan="1" rowspan="1"><p>−50%</p></td><td colspan="1" rowspan="1"><p>~$196/mo compute with existing licenses</p></td><td colspan="1" rowspan="1"><p>Requires Oracle SE2/EE license</p></td></tr><tr><td colspan="1" rowspan="1"><p>AWS Data Transfer</p></td><td colspan="1" rowspan="1"><p>$0.09/GB out</p></td><td colspan="1" rowspan="1"><p>Varies</p></td><td colspan="1" rowspan="1"><p>EC2→DB in same AZ: $0 transfer cost</p></td></tr></tbody></table>

<table style="min-width: 50px;"><colgroup><col style="min-width: 25px;"><col style="min-width: 25px;"></colgroup><tbody><tr><td colspan="1" rowspan="1"><p><strong>⚠️ Important Pricing Note</strong></p></td><td colspan="1" rowspan="1"><p>ECPU auto-scaling can significantly increase costs under heavy load. Always set max_cpu_count to cap spending: ALTER DATABASE SET max_cpu_count = 8; Monitor usage in OCI Cost Analysis dashboard.</p></td></tr></tbody></table>

### **Oracle Database@AWS vs Alternatives: Cost Comparison**

<table style="min-width: 100px;"><colgroup><col style="min-width: 25px;"><col style="min-width: 25px;"><col style="min-width: 25px;"><col style="min-width: 25px;"></colgroup><tbody><tr><td colspan="1" rowspan="1"><p><strong>Service</strong></p></td><td colspan="1" rowspan="1"><p><strong>8 OCPU / 64GB / 2TB / Oracle DB</strong></p></td><td colspan="1" rowspan="1"><p><strong>Oracle Support</strong></p></td><td colspan="1" rowspan="1"><p><strong>Managed?</strong></p></td></tr><tr><td colspan="1" rowspan="1"><p>Oracle DB@AWS (ADB Serverless)</p></td><td colspan="1" rowspan="1"><p>~$650–900/mo</p></td><td colspan="1" rowspan="1"><p>Yes (Oracle)</p></td><td colspan="1" rowspan="1"><p>Yes (Oracle)</p></td></tr><tr><td colspan="1" rowspan="1"><p>Oracle DB@AWS (Exadata Qtr Rack)</p></td><td colspan="1" rowspan="1"><p>~$12,000+/mo</p></td><td colspan="1" rowspan="1"><p>Yes (Oracle)</p></td><td colspan="1" rowspan="1"><p>Yes (Oracle)</p></td></tr><tr><td colspan="1" rowspan="1"><p>RDS for Oracle (db.r6g.2xlarge)</p></td><td colspan="1" rowspan="1"><p>~$1,100/mo + license</p></td><td colspan="1" rowspan="1"><p>Yes (AWS)</p></td><td colspan="1" rowspan="1"><p>Yes (AWS)</p></td></tr><tr><td colspan="1" rowspan="1"><p>Oracle on EC2 (r6g.2xlarge)</p></td><td colspan="1" rowspan="1"><p>~$350/mo compute</p></td><td colspan="1" rowspan="1"><p>No (DIY)</p></td><td colspan="1" rowspan="1"><p>No (DIY)</p></td></tr><tr><td colspan="1" rowspan="1"><p>OCI Autonomous Database (native)</p></td><td colspan="1" rowspan="1"><p>~$580/mo</p></td><td colspan="1" rowspan="1"><p>Yes (Oracle)</p></td><td colspan="1" rowspan="1"><p>Yes (Oracle)</p></td></tr></tbody></table>

![](https://cdn.hashnode.com/uploads/covers/66070b822018a4a9c83bbb3a/153a9917-5d0b-495f-971f-87af0517d1f6.svg align="center")

## **When to Use Oracle Database@AWS (And When Not To)**

This is the section most comparison articles skip. Oracle Database@AWS is genuinely excellent for specific scenarios and genuinely wrong for others.

### **✅ Use Oracle Database@AWS When:**

●      Your application is already on AWS (ECS, EKS, Lambda, EC2) and you need Oracle DB — no migration, no re-architecture

●      You need Oracle-specific features: Partitioning, Advanced Compression, RAC, Data Guard — features RDS for Oracle supports poorly or not at all

●      Oracle licensing compliance is critical — Oracle Customer Success provides direct support, not AWS

●      You need Oracle 23ai features: JSON Relational Duality Views, True Cache, AI Vector Search

●      Your DBA team knows Oracle but not AWS-native databases — lowest re-training cost

●      Regulatory requirements mandate Oracle as the database (common in banking, healthcare, government)

### **❌ Do NOT Use Oracle Database@AWS When:**

●      You are a greenfield project — consider Aurora PostgreSQL or DynamoDB first

●      Cost is the primary constraint — Oracle licensing costs will dominate

●      Your workload is 100% cloud-native and Oracle-agnostic — you are paying for features you will not use

●      You need multi-region active-active — Oracle Data Guard is active-passive; global tables on DynamoDB or Aurora Global are better

●      Your team has no Oracle DBA expertise — managed does not mean zero-knowledge

●      You are evaluating this as a "migration path off Oracle" — this keeps you on Oracle; if you want off, use AWS DMS to PostgreSQL instead

<table style="min-width: 50px;"><colgroup><col style="min-width: 25px;"><col style="min-width: 25px;"></colgroup><tbody><tr><td colspan="1" rowspan="1"><p><strong>🔑 The Bottom Line</strong></p></td><td colspan="1" rowspan="1"><p>Oracle Database@AWS is the best answer for "I need Oracle on AWS" — it is not the answer for "I need a database on AWS." Those are very different questions.</p></td></tr></tbody></table>

![](https://cdn.hashnode.com/uploads/covers/66070b822018a4a9c83bbb3a/aa8e4aea-6908-49e7-ad6b-3e9da61e0c89.svg align="center")

## **Migrating an Existing Oracle Database to Oracle DB@AWS**

If you have Oracle running on-premises or on EC2, here are the three primary migration paths:

### **Option A: Oracle Data Pump (Export/Import) — Best for < 500 GB**

> `-- On source database: export schema`
> 
> `expdp SYSTEM/password@source_db \`
> 
> `SCHEMAS=APP_USER \`
> 
> `DIRECTORY=DATA_PUMP_DIR \`
> 
> `DUMPFILE=app_export_%U.dmp \`
> 
> `LOGFILE=app_export.log \`
> 
> `PARALLEL=4`
> 
> `-- Upload to OCI Object Storage`
> 
> `oci os object bulk-upload \`
> 
> `--bucket-name migration-bucket \`
> 
> `--src-dir /oracle/datapump/`
> 
> `-- On target ADB: import`
> 
> `impdp ADMIN/password@prodadb01_high \`
> 
> `SCHEMAS=APP_USER \`
> 
> `DIRECTORY=DATA_PUMP_DIR \`
> 
> `DUMPFILE=app_export_%U.dmp \`
> 
> `REMAP_SCHEMA=APP_USER:APP_USER \`
> 
> `PARALLEL=4`

### **Option B: Oracle GoldenGate — Best for Zero-Downtime Migration**

GoldenGate provides continuous replication from your source database to ADB, allowing you to cut over with < 1 minute downtime. This is the recommended path for production systems with SLA requirements.

The process: configure GoldenGate Extract on source → configure GoldenGate Replicat on ADB → initial load → catch up CDC → validate data → cutover.

### **Option C: AWS Database Migration Service (DMS) + Schema Conversion**

DMS supports Oracle as a source and Oracle as a target. Use this path if you are also evaluating conversion to PostgreSQL — SCT (Schema Conversion Tool) shows you the conversion complexity before you commit.

<table style="min-width: 125px;"><colgroup><col style="min-width: 25px;"><col style="min-width: 25px;"><col style="min-width: 25px;"><col style="min-width: 25px;"><col style="min-width: 25px;"></colgroup><tbody><tr><td colspan="1" rowspan="1"><p><strong>Migration Method</strong></p></td><td colspan="1" rowspan="1"><p><strong>Downtime</strong></p></td><td colspan="1" rowspan="1"><p><strong>Max Size</strong></p></td><td colspan="1" rowspan="1"><p><strong>Complexity</strong></p></td><td colspan="1" rowspan="1"><p><strong>Best For</strong></p></td></tr><tr><td colspan="1" rowspan="1"><p>Data Pump</p></td><td colspan="1" rowspan="1"><p>4–24 hours</p></td><td colspan="1" rowspan="1"><p>&lt; 500 GB</p></td><td colspan="1" rowspan="1"><p>Low</p></td><td colspan="1" rowspan="1"><p>Dev/test, small prod databases</p></td></tr><tr><td colspan="1" rowspan="1"><p>GoldenGate</p></td><td colspan="1" rowspan="1"><p>&lt; 5 min</p></td><td colspan="1" rowspan="1"><p>Any size</p></td><td colspan="1" rowspan="1"><p>High</p></td><td colspan="1" rowspan="1"><p>Large prod with SLA</p></td></tr><tr><td colspan="1" rowspan="1"><p>AWS DMS</p></td><td colspan="1" rowspan="1"><p>2–8 hours</p></td><td colspan="1" rowspan="1"><p>&lt; 2 TB</p></td><td colspan="1" rowspan="1"><p>Medium</p></td><td colspan="1" rowspan="1"><p>Oracle → Oracle or Oracle → PostgreSQL</p></td></tr><tr><td colspan="1" rowspan="1"><p>RMAN Backup/Restore</p></td><td colspan="1" rowspan="1"><p>8–48 hours</p></td><td colspan="1" rowspan="1"><p>&lt; 5 TB</p></td><td colspan="1" rowspan="1"><p>Medium</p></td><td colspan="1" rowspan="1"><p>Infrequent migrations, DR setup</p></td></tr></tbody></table>

## **Monitoring & Observability**

Oracle Database@AWS exposes metrics through both OCI Monitoring and — critically — AWS CloudWatch. This dual-plane observability is one of the service's underrated strengths.

### **OCI Monitoring Metrics**

> `# Query ADB CPU utilization via OCI CLI`
> 
> `oci monitoring metric-data summarize-metrics-data \`
> 
> `--compartment-id ocid1.compartment.oc1..xxxx \`
> 
> `--namespace oracle_autonomous_database \`
> 
> `--query-text 'CpuUtilization[1m]{resourceId = "ocid1.autonomousdatabase.oc1..xxxx"}.mean()' \`
> 
> `--start-time 2026-03-17T00:00:00Z \`
> 
> `--end-time 2026-03-17T06:00:00Z`

### **CloudWatch Integration**

Enable CloudWatch metrics publishing from the OCI Console → Autonomous Database → CloudWatch Integration. Metrics appear in CloudWatch under the OracleDatabase namespace within 5 minutes.

<table style="min-width: 75px;"><colgroup><col style="min-width: 25px;"><col style="min-width: 25px;"><col style="min-width: 25px;"></colgroup><tbody><tr><td colspan="1" rowspan="1"><p><strong>Metric</strong></p></td><td colspan="1" rowspan="1"><p><strong>CloudWatch Namespace</strong></p></td><td colspan="1" rowspan="1"><p><strong>Alert Threshold (Suggestion)</strong></p></td></tr><tr><td colspan="1" rowspan="1"><p>CpuUtilization</p></td><td colspan="1" rowspan="1"><p>OracleDatabase</p></td><td colspan="1" rowspan="1"><p>&gt; 85% for 5 mins → scale up</p></td></tr><tr><td colspan="1" rowspan="1"><p>StorageUtilization</p></td><td colspan="1" rowspan="1"><p>OracleDatabase</p></td><td colspan="1" rowspan="1"><p>&gt; 75% → expand storage</p></td></tr><tr><td colspan="1" rowspan="1"><p>SessionCount</p></td><td colspan="1" rowspan="1"><p>OracleDatabase</p></td><td colspan="1" rowspan="1"><p>&gt; 80% of max → connection pool alert</p></td></tr><tr><td colspan="1" rowspan="1"><p>SQLResponseTime</p></td><td colspan="1" rowspan="1"><p>OracleDatabase</p></td><td colspan="1" rowspan="1"><p>&gt; 2x baseline → investigate</p></td></tr><tr><td colspan="1" rowspan="1"><p>FailedConnections</p></td><td colspan="1" rowspan="1"><p>OracleDatabase</p></td><td colspan="1" rowspan="1"><p>&gt; 10/min → security alert</p></td></tr></tbody></table>

## **Best Practices for Production Deployments**

<table style="min-width: 75px;"><colgroup><col style="min-width: 25px;"><col style="min-width: 25px;"><col style="min-width: 25px;"></colgroup><tbody><tr><td colspan="1" rowspan="1"><p><strong>Category</strong></p></td><td colspan="1" rowspan="1"><p><strong>Best Practice</strong></p></td><td colspan="1" rowspan="1"><p><strong>Why</strong></p></td></tr><tr><td colspan="1" rowspan="1"><p>Security</p></td><td colspan="1" rowspan="1"><p>Use mutual TLS (mTLS) with wallet, not clear text</p></td><td colspan="1" rowspan="1"><p>Oracle enforces this for ADB; do not disable</p></td></tr><tr><td colspan="1" rowspan="1"><p>Security</p></td><td colspan="1" rowspan="1"><p>Store wallet and credentials in AWS Secrets Manager</p></td><td colspan="1" rowspan="1"><p>Never embed in code or environment variables</p></td></tr><tr><td colspan="1" rowspan="1"><p>Security</p></td><td colspan="1" rowspan="1"><p>Create application-specific DB users, never use ADMIN</p></td><td colspan="1" rowspan="1"><p>Principle of least privilege — ADMIN is your break-glass account</p></td></tr><tr><td colspan="1" rowspan="1"><p>Networking</p></td><td colspan="1" rowspan="1"><p>Deploy Oracle DB@AWS endpoint in private subnet only</p></td><td colspan="1" rowspan="1"><p>No public internet access to database ever</p></td></tr><tr><td colspan="1" rowspan="1"><p>Networking</p></td><td colspan="1" rowspan="1"><p>Use VPC endpoint policies to restrict access to specific IAM roles</p></td><td colspan="1" rowspan="1"><p>Defense in depth</p></td></tr><tr><td colspan="1" rowspan="1"><p>Performance</p></td><td colspan="1" rowspan="1"><p>Enable connection pooling (UCP or HikariCP)</p></td><td colspan="1" rowspan="1"><p>ADB Serverless has per-session overhead; pool amortizes this</p></td></tr><tr><td colspan="1" rowspan="1"><p>Performance</p></td><td colspan="1" rowspan="1"><p>Use the <em>high connection string for OLTP, </em>low for reporting</p></td><td colspan="1" rowspan="1"><p>Different parallelism and priority profiles</p></td></tr><tr><td colspan="1" rowspan="1"><p>Cost</p></td><td colspan="1" rowspan="1"><p>Set max_cpu_count to prevent runaway auto-scaling costs</p></td><td colspan="1" rowspan="1"><p>Without cap, a query storm can 10x your monthly bill</p></td></tr><tr><td colspan="1" rowspan="1"><p>Cost</p></td><td colspan="1" rowspan="1"><p>Enable auto-pause for dev/test databases (15-min idle threshold)</p></td><td colspan="1" rowspan="1"><p>ADB stops billing compute when paused</p></td></tr><tr><td colspan="1" rowspan="1"><p>Operations</p></td><td colspan="1" rowspan="1"><p>Enable Automatic Indexing — it is a 23ai superpower</p></td><td colspan="1" rowspan="1"><p>Oracle ML models identify and create optimal indexes automatically</p></td></tr><tr><td colspan="1" rowspan="1"><p>Operations</p></td><td colspan="1" rowspan="1"><p>Configure Data Guard (Autonomous Data Guard) for production</p></td><td colspan="1" rowspan="1"><p>Automatic failover, RPO &lt; 5 seconds, no extra config</p></td></tr><tr><td colspan="1" rowspan="1"><p>Backup</p></td><td colspan="1" rowspan="1"><p>Verify backup retention is set to 60 days minimum</p></td><td colspan="1" rowspan="1"><p>Default is 60 days; confirm in console</p></td></tr></tbody></table>

## **Common Issues and Fixes**

### **Issue: ORA-12541: TNS:no listener**

Most common cause: your EC2 security group outbound rules do not allow traffic to port 1522. Secondary cause: TNS\_ADMIN environment variable not pointing to the correct wallet directory.

> `# Verify TNS_ADMIN is set`
> 
> `echo $TNS_ADMIN`
> 
> `# Test port connectivity from EC2`
> 
> `nc -zv 1522`
> 
> `# If nc fails, check security group egress rules`
> 
> `aws ec2 describe-security-groups \`
> 
> `--group-ids sg-xxxxxxxxxxxxxxxxx \`
> 
> `--query "SecurityGroups[0].IpPermissionsEgress"`

### **Issue: ORA-28000: Account is locked**

The ADMIN account locks after 10 failed login attempts. Unlock via OCI Console → Autonomous Database → Database Users → ADMIN → Unlock. For application users, unlock via SQL:

> `-- Connect as ADMIN and unlock application user`
> 
> `ALTER USER app_user ACCOUNT UNLOCK;`
> 
> `ALTER USER app_user IDENTIFIED BY "NewSecurePassword2026!";`

### **Issue: Slow Query Performance vs. On-Premises Oracle**

If queries are slower than expected after migration, the most common causes are: (1) missing indexes that Automatic Indexing hasn't yet identified, (2) stale statistics, (3) using the wrong connection string (\_low instead of \_high for OLTP).

> `-- Check and gather fresh statistics`
> 
> `EXEC DBMS_STATS.GATHER_SCHEMA_STATS(ownname => 'APP_USER', cascade => TRUE);`

> `-- Check Automatic Indexing recommendations`
> 
> `SELECT * FROM DBA_AUTO_INDEX_IND_ACTIONS ORDER BY creation_time DESC;`

> `-- Check current connection service level`
> 
> `SELECT SYS_CONTEXT('USERENV', 'SERVICE_NAME') FROM DUAL;`

* * *

### ***Safe Harbour & Disclaimer***

*PRICING DISCLAIMER: All prices shown are estimates based on publicly available AWS and Oracle pricing pages as of March 2026 and may not reflect current rates, promotional pricing, committed use discounts, or enterprise agreements. Verify current pricing at* [*cloud.oracle.com*](http://cloud.oracle.com) *and aws.amazon.com/pricing before making purchasing decisions.*

*SAFE HARBOUR: This post contains forward-looking statements based on current service capabilities. Oracle and AWS may change, discontinue, or alter Oracle Database@AWS features, availability, and pricing at any time without notice.*

*NO AFFILIATION:* [*cloudexplorers.club*](http://cloudexplorers.club) *is an independent technical blog. This post is not sponsored by, endorsed by, or affiliated with Oracle Corporation or Amazon Web Services. All opinions are those of the author.*

*LICENSE: This post is published under Creative Commons Attribution 4.0 International (CC BY 4.0). You may share and adapt with attribution to* [*cloudexplorers.club*](http://cloudexplorers.club)*.*