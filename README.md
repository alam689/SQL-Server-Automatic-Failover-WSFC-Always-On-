# SQL-Server-Automatic-Failover-WSFC-Always-On-
Step-by-Step: SQL Server Automatic Failover (WSFC + Always On)

🧱 Assumptions
*  2 Windows Servers
    *  SQL01 (Primary)
    *  SQL02 (Secondary)
*  Same Windows Domain
*  Same SQL Server version
*  Static IPs
*  One production database (FULL recovery)

##  🔹 STEP 1: Prepare Both Servers
* ✔ Install Windows Updates
    * Patch both servers fully
    * Restart
* ✔ Set Static IP
  * Each server must have a static IP
 
##  🔹 STEP 2: Install SQL Server (Both Servers)
* Same SQL version & edition
* Same collation
  * Enable:
  * Database Engine
* SQL Server Agent
* ✅ Confirm SQL service is running on both

## * ১. SQL Version এবং Edition চেক করা
  * উভয় সার্ভারে একই ভার্সন (যেমন: SQL Server 2022) এবং এডিশন (যেমন: Enterprise বা Standard) আছে কি না তা জানতে নিচের কুয়েরিটি চালান:
    * SELECT @@VERSION AS 'SQL_Server_Details';
      * or
    * SELECT SERVERPROPERTY('ProductVersion') AS Product_Version,
       SERVERPROPERTY('ProductLevel') AS Patch_Level,
       SERVERPROPERTY('Edition') AS SQL_Edition;
## * ২. Collation চেক করা
* Collation এক না হলে ডেটা ট্রান্সফার বা জয়েন করার সময় এরর আসবে। এটি চেক করতে এই কুয়েরিটি লিখুন:
  * SELECT SERVERPROPERTY('Collation') AS Server_Collation;
## * ৩. Database Engine এবং Agent সক্রিয় আছে কি না দেখা
এটি চেক করার সবচেয়ে সহজ উপায় হলো SQL Server Configuration Manager অথবা SSMS ব্যবহার করা। পদ্ধতি খ: SQL Query ব্যবহার কর
ইঞ্জিন এবং এজেন্ট সার্ভিস চলছে কি না তা জানতে এই কোডটি রান করুন:
  * SELECT servicename, status_desc FROM sys.dm_server_services;

##  🔹 STEP 3: Enable Failover Clustering Feature
### On both servers:
  * Open Server Manager
  * Click Add Roles and Features
  * Select Failover Clustering
  * Install and reboot
    
##  🔹 STEP 4: Create Windows Server Failover Cluster (WSFC)
On SQL01:
  * Open Failover Cluster Manager
  * Click Validate Configuration
  * Add both servers (SQL01, SQL02)
  * Run all tests
    * ⚠ Ignore storage warnings (normal for AG)
* Create Cluster
  * Cluster Name: SQLCLUSTER
  * Cluster IP: Static IP
* ✅ WSFC is now ready

##  🔹 STEP 5: Configure Quorum (IMPORTANT)
 Recommended: File Share Witness
* Create shared folder (e.g. \\FileServer\Witness)
* Grant Full Control to cluster computer account
* In Failover Cluster Manager:
  * More Actions → Configure Cluster Quorum
  * Select File Share Witness
 
##  🔹 STEP 6: Enable Always On Availability Groups
On both SQL servers:
* Open SQL Server Configuration Manager
* SQL Server Services
* Right-click SQL Server → Properties
* Enable Always On Availability Groups
* Restart SQL Service

##  🔹 STEP 7: Prepare Database
On Primary (SQL01):
<pre>  ALTER DATABASE YourDB SET RECOVERY FULL;</pre>
* Take full backup:
<pre> BACKUP DATABASE YourDB TO DISK = 'C:\Backup\YourDB.bak';</pre>
* Restore on Secondary (SQL02):
<pre> 
     RESTORE DATABASE YourDB
     FROM DISK = 'C:\Backup\YourDB.bak'
     WITH NORECOVERY;
</pre>

##  🔹 STEP 8: Create Availability Group

On SQL01:
  * Open SSMS
  * Always On High Availability
  * Right-click Availability Groups
  * New Availability Group Wizard
* Select:
  * Database: YourDB
  * Replicas:
    * SQL01 → Primary
    * SQL02 → Secondary
* Commit mode: Synchronous
* Failover: Automatic
  
##  🔹 STEP 9: Create Listener (Very Important)
* During wizard or later:
* Listener Name: SQLAGLISTENER
* Port: 1433
* Static IP
* 👉 Applications MUST connect using this listener

 ##  🔹 STEP 10: Test Failover
* Stop SQL Service on SQL01
* Watch:
  * SQL02 becomes PRIMARY automatically
* Start SQL01 again → becomes SECONDARY
* ✅ Automatic failover confirmed

##  🔹 STEP 11: Update Application Connection String
* ❌ Old:
  * Server=SQL01;Database=YourDB;
* ✅ New:
  * Server=SQLAGLISTENER;Database=YourDB;

##  🔐 Final Architecture
<pre> Application
    |
    v
SQL Listener (DNS)
    |
---------------------
|                   |
SQL01 (Primary)  SQL02 (Secondary)
</pre> 

##  ⚠️ Common Mistakes to Avoid
* ❌ Different SQL versions
* ❌ No listener used
* ❌ Database not FULL recovery
* ❌ No quorum witness
* ❌ Async mode for HA
* 🧠 One-Line Summary
    * WSFC decides failover, SQL Always On syncs data, Listener keeps app connected.
