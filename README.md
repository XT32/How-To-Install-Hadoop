# Hadoop Multi-Node Cluster Installation and Configuration
# Instalasi dan Konfigurasi Klaster Hadoop (Multi-Node)

Cara untuk menginstal dan mengonfigurasi klaster Hadoop versi 3.4.1 dengan satu Master Node dan dua Slave Node di Ubuntu Server 24.04 LTS.
How to install and configure a Hadoop version 3.4.1 cluster with one Master Node and two Slave Nodes on Ubuntu Server 24.04 LTS.

## Daftar Isi

* [English Version](#english-version)
    * [Main Prerequisites](#main-prerequisites-en)
    * [A. Tasks on MASTER NODE - Preparation and Core Configuration](#a-tasks-on-master-node---preparation-and-core-configuration-en)
    * [B. Tasks on SLAVE1 NODE - Initial Setup](#b-tasks-on-slave1-node---initial-setup-en)
    * [C. Tasks on SLAVE2 NODE](#c-tasks-on-slave2-node-en)
    * [D. Tasks on MASTER NODE - Configuration Distribution and Cluster Start-up](#d-tasks-on-master-node---configuration-distribution-and-cluster-start-up-en)
    * [E. Cluster Verification](#e-cluster-verification-en)
    * [Important Notes](#important-notes-en)
* [Versi Bahasa Indonesia](#versi-bahasa-indonesia)
    * [Prasyarat Utama](#prasyarat-utama-id)
    * [A. Tugas di MASTER NODE - Persiapan dan Konfigurasi Inti](#a-tugas-di-master-node---persiapan-dan-konfigurasi-inti-id)
    * [B. Tugas di SLAVE1 NODE - Persiapan Awal](#b-tugas-di-slave1-node---persiapan-awal-id)
    * [C. Tugas di SLAVE2 NODE](#c-tugas-di-slave2-node-id)
    * [D. Tugas di MASTER NODE - Distribusi Konfigurasi dan Menjalankan Klaster](#d-tugas-di-master-node--distribusi-konfigurasi-dan-menjalankan-klaster-id)
    * [E. Verifikasi Klaster](#e-verifikasi-klaster-id)
    * [Catatan Penting (Notes)](#catatan-penting-notes-id)

---

## English Version
<a name="english-version"></a>

<a name="main-prerequisites-en"></a>
### Main Prerequisites (Complete these before starting the installation)

Before starting the steps below, ensure the following conditions are met for a smooth installation:

1.  **Three Virtual Machines (VMs)**: Prepare three VMs running Ubuntu Server 24.04 LTS.
    * One will serve as the **Master Node** (Hadoop key node).
    * The other two as **Slave Nodes** (slave1 and slave2).
    * If using physical machines, proceed to step 2.
2.  **Static IP Addresses**: Each VM must be configured with a unique static IP address. All static IPs must be in the same network subnet to communicate with each other.
    * If using the same router, check the IP Address with the command:
        ```bash
        ip add
        ```
    * Example:
        * Master: `192.168.159.131` (adjust to your static IP)
        * Slave1: `192.168.159.129` (adjust to your static IP)
        * Slave2: `192.168.159.130` (adjust to your static IP)
    * For ease, try to sequence the IPs (master, slave1, slave2). If not, it's okay. Remember all three IP addresses.
    * **IMPORTANT**: Must be STATIC IP, DO NOT USE DHCP OR AUTOMATIC ASSIGNMENT FROM ROUTER (Configure carefully to avoid IP conflicts).
3.  **Configure `/etc/hosts` File**: The `/etc/hosts` file on each VM (Master, Slave1, and Slave2) must be edited.
    * Open with:
        ```bash
        sudo nano /etc/hosts
        ```
    * Example content of `/etc/hosts` (must be the same on all nodes):
        ```
        127.0.0.1       localhost
        # Replace/add the IPs you obtained and noted earlier into the file
        192.168.159.131 master
        192.168.159.129 slave1
        192.168.159.130 slave2
        ```
    * For `localhost`, you can add `#` at the beginning to prevent conflicts or issues where slaves cannot contact the master. Use the same hostnames as in the example for simplicity.

---

<a name="a-tasks-on-master-node---preparation-and-core-configuration-en"></a>
### A. Tasks on MASTER NODE - Preparation and Core Configuration

Perform all the following steps on the **Master Node**.

1.  **Update Operating System**:
    ```bash
    sudo apt update && sudo apt upgrade -y
    ```
    *(Optional, depending on the situation. To ensure all packages are up to date)*

2.  **Install Java Development Kit (OpenJDK 11)**:
    ```bash
    sudo apt install openjdk-11-jdk -y
    ```

3.  **Create a Dedicated User and Group for Hadoop**:
    ```bash
    sudo addgroup hadoop
    sudo adduser --ingroup hadoop hduser
    ```
    * Follow the prompts to create a password and fill in info (can be skipped).
    * Optional, to allow `hduser` to run `sudo` commands without a password:
        ```bash
        sudo usermod -aG sudo hduser
        ```
    * It's recommended to run Hadoop with a separate user (`hduser`).

4.  **Login as `hduser`**:
    ```bash
    su - hduser
    ```
    * All subsequent Hadoop commands will be run as this user. The prompt will change (e.g., `hduser@ubuntu $`).

5.  **Set Environment Variables for `hduser`**:
    Edit `~/.bashrc`:
    ```bash
    nano ~/.bashrc
    ```
    Add to the end of the file:
    ```bash
    export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
    export HADOOP_HOME=/usr/local/hadoop
    export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
    export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$JAVA_HOME/bin
    # Add other variables like HADOOP_MAPRED_HOME, HADOOP_COMMON_HOME, HADOOP_HDFS_HOME, YARN_HOME if needed
    export HADOOP_OPTS="-Djava.library.path=$HADOOP_HOME/lib/native"
    ```
    Apply the changes:
    ```bash
    source ~/.bashrc
    ```

6.  **Create Hadoop Data Storage Directories on Master**:
    ```bash
    mkdir -p /home/hduser/hadoop_data/tmp_dir
    mkdir -p /home/hduser/hadoop_data/namenode_dir
    ```
    * The directory names `namenode_dir` and `tmp_dir` can be changed, but must be remembered for consistency.

7.  **Create SSH Key Pair for `hduser` (for Passwordless SSH)**:
    ```bash
    ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
    ```
    * Press Enter for all prompts if asked for a passphrase (leave it empty).

8.  **Authorize SSH to Self (localhost/master)**:
    ```bash
    cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
    chmod 700 ~/.ssh
    chmod 600 ~/.ssh/authorized_keys
    ```

9.  **Download, Verify, and Extract Hadoop**:
    Change to home directory:
    ```bash
    cd ~
    ```
    Download Hadoop (example version 3.4.1, adjust if necessary):
    ```bash
    wget [https://dlcdn.apache.org/hadoop/common/hadoop-3.4.1/hadoop-3.4.1.tar.gz](https://dlcdn.apache.org/hadoop/common/hadoop-3.4.1/hadoop-3.4.1.tar.gz)
    wget [https://dlcdn.apache.org/hadoop/common/hadoop-3.4.1/hadoop-3.4.1.tar.gz.sha512](https://dlcdn.apache.org/hadoop/common/hadoop-3.4.1/hadoop-3.4.1.tar.gz.sha512)
    ```
    Verify checksum:
    ```bash
    sha512sum -c hadoop-3.4.1.tar.gz.sha512
    ```
    * Ensure the output is `OK`.
    Extract to `/usr/local/`:
    ```bash
    sudo tar -xzvf hadoop-3.4.1.tar.gz -C /usr/local/
    ```
    Rename directory:
    ```bash
    sudo mv /usr/local/hadoop-3.4.1 /usr/local/hadoop
    ```
    Change ownership:
    ```bash
    sudo chown -R hduser:hadoop /usr/local/hadoop
    ```

10. **Edit Core Hadoop Configuration Files**:
    Change to configuration directory:
    ```bash
    cd $HADOOP_HOME/etc/hadoop
    ```
    * **Edit `hadoop-env.sh`**:
        ```bash
        nano hadoop-env.sh
        ```
        *(Use `sudo nano` if the file is unwriteable)*
        Ensure the line `export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64` (or your Java path) is uncommented (no `#`) and correct.
        Add or ensure this line is present:
        ```bash
        export HADOOP_CONF_DIR="/usr/local/hadoop/etc/hadoop"
        ```
    * **Edit `core-site.xml`**:
        ```bash
        nano core-site.xml
        ```
        Content:
        ```xml
        <configuration>
            <property>
                <name>fs.defaultFS</name>
                <value>hdfs://master:9000</value>
            </property>
            <property>
                <name>hadoop.tmp.dir</name>
                <value>/home/hduser/hadoop_data/tmp_dir</value>
            </property>
        </configuration>
        ```
    * **Edit `hdfs-site.xml`**:
        ```bash
        nano hdfs-site.xml
        ```
        Content:
        ```xml
        <configuration>
            <property>
                <name>dfs.replication</name>
                <value>2</value>
            </property>
            <property>
                <name>dfs.namenode.name.dir</name>
                <value>file:///home/hduser/hadoop_data/namenode_dir</value>
            </property>
            <property>
                <name>dfs.datanode.data.dir</name>
                <value>file:///home/hduser/hadoop_data/datanode_dir</value>
            </property>
            <property>
                <name>dfs.namenode.secondary.http-address</name>
                <value>master:9868</value>
            </property>
        </configuration>
        ```
    * **Edit `mapred-site.xml`**:
        Copy from template if necessary:
        ```bash
        cp mapred-site.xml.template mapred-site.xml
        nano mapred-site.xml
        ```
        Content:
        ```xml
        <configuration>
            <property>
                <name>mapreduce.framework.name</name>
                <value>yarn</value>
            </property>
            <property>
                <name>mapreduce.application.classpath</name>
                <value>$HADOOP_HOME/share/hadoop/mapreduce/*:$HADOOP_HOME/share/hadoop/mapreduce/lib/*</value>
            </property>
        </configuration>
        ```
    * **Edit `yarn-site.xml`**:
        ```bash
        nano yarn-site.xml
        ```
        Content:
        ```xml
        <configuration>
            <property>
                <name>yarn.nodemanager.aux-services</name>
                <value>mapreduce_shuffle</value>
            </property>
            <property>
                <name>yarn.resourcemanager.hostname</name>
                <value>master</value>
            </property>
            <property>
                <name>yarn.nodemanager.resource.memory-mb</name>
                <value>3072</value>
            </property>
            <property>
                <name>yarn.nodemanager.resource.cpu-vcores</name>
                <value>2</value>
            </property>
            <property>
                <name>yarn.log-aggregation-enable</name>
                <value>true</value>
            </property>
            <property>
                <name>yarn.log.server.url</name>
                <value>http://master:19888/jobhistory/logs</value>
            </property>
            <property>
                <name>yarn.web-proxy.address</name>
                <value>master:8081</value>
            </property>
        </configuration>
        ```
    * **Edit `workers`**:
        ```bash
        nano workers
        ```
        Delete the default content (usually `localhost`), and add your slave hostnames, one per line:
        ```
        slave1
        slave2
        ```

---

<a name="b-tasks-on-slave1-node---initial-setup-en"></a>
### B. Tasks on SLAVE1 NODE - Initial Setup

Perform all the following steps on **Slave1 Node**. (Log in as your main user, not `hduser` unless specified).

1.  **Update Operating System** (Same as Master Node - Step A.1)
2.  **Install Java Development Kit (OpenJDK 11)** (Same as Master Node - Step A.2)
3.  **Create a Dedicated User and Group for Hadoop** (Same as Master Node - Step A.3)
4.  **Login as `hduser`** (Same as Master Node - Step A.4)
    ```bash
    su - hduser
    ```
5.  **Set Environment Variables for `hduser`**:
    Edit `~/.bashrc`:
    ```bash
    nano ~/.bashrc
    ```
    Add the exact same lines as on the Master Node for `JAVA_HOME`, `HADOOP_HOME`, `PATH`, `HADOOP_OPTS`, etc.
    Apply the changes:
    ```bash
    source ~/.bashrc
    ```
6.  **Create Hadoop Data Storage Directories on Slave1**:
    (As `hduser`)
    ```bash
    mkdir -p /home/hduser/hadoop_data/tmp_dir
    mkdir -p /home/hduser/hadoop_data/datanode_dir
    ```
7.  **Prepare `.ssh` Directory to Receive Key from Master**:
    (As `hduser`)
    ```bash
    rm -f ~/.ssh/authorized_keys
    mkdir -p ~/.ssh
    chmod 700 ~/.ssh
    ```
    *(The `rm` command is to remove old keys if any)*
8.  **Prepare Target Directory for Hadoop Installation**:
    (Switch back to your main user if needed, or use `sudo` as `hduser` if configured in `sudoers`)
    ```bash
    sudo mkdir -p /usr/local/hadoop
    sudo chown hduser:hadoop /usr/local/hadoop
    ```

---

<a name="c-tasks-on-slave2-node-en"></a>
### C. Tasks on SLAVE2 NODE

Perform the **EXACT SAME** steps as on **SLAVE1 NODE** (Section B).
**Caution**: Be meticulous and ensure no configuration mix-ups between Slave1 and Slave2.

---

<a name="d-tasks-on-master-node---configuration-distribution-and-cluster-start-up-en"></a>
### D. Tasks on MASTER NODE  - Configuration Distribution and Cluster Start-up

Return to the **Master Node** and ensure you are logged in as `hduser`.

1.  **Copy Master's SSH Public Key to All Slave Nodes**:
    ```bash
    ssh-copy-id hduser@slave1
    # Enter hduser's password for slave1 if prompted (password of hduser on slave1)
    ssh-copy-id hduser@slave2
    # Enter hduser's password for slave2 if prompted (password of hduser on slave2)
    ```
    * Answer `yes` if prompted about host authenticity.
    * Example output during `ssh-copy-id`:
        ```
        master@ubuntu $ ssh-copy-id hduser@slave1
        /usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/home/hduser/.ssh/id_rsa.pub"
        The authenticity of host 'slave1 (192.168.159.129)' can't be established.
        ED25519 key fingerprint is SHA256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx.
        This key is not known by any other names.
        Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
        /usr/bin/ssh-copy-id: INFO: 1 key(s)
        ```

2.  **Copy Entire Hadoop Installation Directory to Slave Nodes**:
    ```bash
    scp -rp /usr/local/hadoop hduser@slave1:/usr/local/
    scp -rp /usr/local/hadoop hduser@slave2:/usr/local/
    ```
    * The `-p` option preserves file permissions.

3.  **Format HDFS NameNode (One-Time Operation!)**:
    ```bash
    hdfs namenode -format
    ```
    * **IMPORTANT**: Check the output! Ensure there are no errors and the NameNode identifies itself with the correct Master LAN IP (not `127.0.0.1` or `127.0.1.1`). If incorrect, recheck IP and `/etc/hosts` configurations.

4.  **Start HDFS Services**:
    ```bash
    start-dfs.sh
    ```

5.  **Start YARN Services**:
    ```bash
    start-yarn.sh
    ```

6.  **Start MapReduce JobHistory Server (Optional)**:
    ```bash
    mr-jobhistory-daemon.sh start historyserver
    ```

---

<a name="e-cluster-verification-en"></a>
### E. Cluster Verification (Perform on the Respective Nodes)

1.  **Check Running Java Processes (`jps`)**:
    * **On Master Node**: Run `jps`. You should see: `NameNode`, `SecondaryNameNode`, `ResourceManager`, `JobHistoryServer` (if started), `Jps`.
    * **On Slave1 Node**: Login (`ssh hduser@slave1`) and run `jps`. You should see: `DataNode`, `NodeManager`, `Jps`.
    * **On Slave2 Node**: Login (`ssh hduser@slave2`) and run `jps`. You should see: `DataNode`, `NodeManager`, `Jps`.

2.  **Access Hadoop Web UIs**:
    Open a browser on your computer and access the following URLs (replace `<IP_MASTER>` with your Master Node's static IP):
    * **HDFS NameNode Overview**: `http://<IP_MASTER>:9870`
        * Check the "Datanode Information" -> "Live Nodes" section. The count should be `2`.
    * **YARN ResourceManager Overview**: `http://<IP_MASTER>:8088`
        * Click "Nodes" in the menu. There should be `2` active nodes (slave1 and slave2) with `RUNNING` status.
    * **MapReduce JobHistory Server**: `http://<IP_MASTER>:19888`
    * **Firewall**: Ensure the firewall on the Master Node allows these ports.
        Check firewall status:
        ```bash
        sudo ufw status verbose
        ```
        If inactive and you want to enable it:
        ```bash
        sudo ufw enable
        ```
        Allow necessary ports (if not already listed):
        ```bash
        sudo ufw allow 22/tcp   # SSH (IMPORTANT!)
        sudo ufw allow 9870/tcp # HDFS NameNode UI
        sudo ufw allow 8088/tcp # YARN ResourceManager UI
        sudo ufw allow 19888/tcp# MapReduce JobHistory Server UI
        sudo ufw allow 8030/tcp # YARN RPC
        sudo ufw allow 8031/tcp # YARN RPC
        sudo ufw allow 8032/tcp # YARN RPC
        sudo ufw reload
        ```

3.  **Test Basic HDFS Operations**:
    From the Master Node, as `hduser`:
    ```bash
    hdfs dfs -mkdir -p /user/hduser/test_rekap
    echo "Test rekap HDFS successful at $(date)" > ~/file_rekap.txt
    hdfs dfs -put ~/file_rekap.txt /user/hduser/test_rekap/
    hdfs dfs -ls /user/hduser/test_rekap/
    hdfs dfs -cat /user/hduser/test_rekap/file_rekap.txt
    ```

4.  **Run a Sample MapReduce Job (WordCount)**:
    Prepare an input file (e.g., `book.txt`) in HDFS. Example path: `/user/hduser/wordcount_rekap/input/book.txt`.
    Create the input directory in HDFS and upload the file:
    ```bash
    hdfs dfs -mkdir -p /user/hduser/wordcount_rekap/input
    # Assuming you have a book.txt file in hduser's home directory
    hdfs dfs -put ~/book.txt /user/hduser/wordcount_rekap/input/
    ```
    Remove old output directory if it exists:
    ```bash
    hdfs dfs -rm -r /user/hduser/wordcount_rekap/output
    ```
    Run the job:
    ```bash
    hadoop jar $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-examples-*.jar wordcount /user/hduser/wordcount_rekap/input /user/hduser/wordcount_rekap/output
    ```
    Monitor progress in the terminal and YARN UI. After completion (`SUCCEEDED`), check the results:
    ```bash
    hdfs dfs -cat /user/hduser/wordcount_rekap/output/part-r-00000
    ```

---

<a name="important-notes-en"></a>
### Important Notes

* `sudo` is used if environment files are not writable or `unwriteable`.
* Pay close attention to the **IP ADDRESS** of each computer or VM. IP conflicts will cause communication failure.
* Commands can be executed sequentially (separated by `&&`), but be careful.
* If you encounter the `rm -rf` command, carefully check which directory or file is being deleted. Mistakes can be fatal.
* When editing configuration files (`<configuration>...</configuration>`), it's advisable to delete the existing `<configuration>...</configuration>` block (if any) and then paste the entire new XML content.
* For copy-pasting in a Linux terminal, use `CTRL+SHIFT+C` (copy) and `CTRL+SHIFT+V` (paste). Do not use the regular `CTRL+C`/`CTRL+V` as `CTRL+C` is an interrupt signal.
* XML configuration files are very sensitive to syntax (dots, commas, spaces, opening/closing tags). Be precise!
* Using 3 VMs on a personal laptop is not recommended if its specifications are limited. Hadoop is quite resource-intensive. Basic networking knowledge (NAT, IPv4) is also important.
* Linux is an art; it requires meticulousness and extra attention. A slight lapse can cause significant damage.

---
---

## Versi Bahasa Indonesia
<a name="versi-bahasa-indonesia"></a>

<a name="prasyarat-utama-id"></a>
### PRASYARAT UTAMA (Selesaikan terlebih dahulu sebelum memulai instalasi)

Sebelum memulai langkah-langkah di bawah, pastikan kondisi berikut terpenuhi untuk kelancaran instalasi:

1.  **Tiga Virtual Machine (VM)**: Siapkan tiga VM yang menjalankan Ubuntu Server 24.04 LTS.
    * Satu akan berfungsi sebagai **Master Node** (kunci hadoop).
    * Dua lainnya sebagai **Slave Node** (slave1 dan slave2).
    * Jika penggunaan melalui komputer langsung, lanjut ke tahap ke-2.
2.  **Alamat IP Statis**: Setiap VM harus dikonfigurasi dengan alamat IP statis yang unik. Semua IP statis ini harus berada dalam subnet jaringan yang sama agar bisa saling berkomunikasi.
    * Jika menggunakan router yang sama, lakukan pengecekan IP Address dengan command:
        ```bash
        ip add
        ```
    * Contoh:
        * Master: `192.168.159.131` (sesuaikan dengan IP statisnya)
        * Slave1: `192.168.159.129` (sesuaikan dengan IP statisnya)
        * Slave2: `192.168.159.130` (sesuaikan dengan IP statisnya)
    * Agar lebih mudah, usahakan urutan IP nya dari master, slave1, slave2. Jika tidak masalah, bisa dibiarkan. Usahakan mengingat IP address ketiganya.
    * **PENTING**: Harus IP STATIS, JANGAN DHCP ATAU ASSIGN OTOMATIS DARI ROUTER (Konfigurasi harus hati-hati karena bisa menyebabkan 2 perangkat memiliki IP address yang sama).
3.  **Konfigurasi File `/etc/hosts`**: File `/etc/hosts` di setiap VM (Master, Slave1, dan Slave2) harus diedit.
    * Buka dengan:
        ```bash
        sudo nano /etc/hosts
        ```
    * Contoh isi file `/etc/hosts` (harus sama di semua node):
        ```
        127.0.0.1       localhost
        # Ganti/tambahkan IP yang sudah didapat dan diingat tadi ke dalam file
        192.168.159.131 master
        192.168.159.129 slave1
        192.168.159.130 slave2
        ```
    * Untuk localhost, bisa ditambahkan `#` di depannya agar tidak terjadi tabrakan atau slave tidak bisa menghubungi master. Berikan hostname sama seperti contoh di atas untuk kemudahan.

---

<a name="a-tugas-di-master-node---persiapan-dan-konfigurasi-inti-id"></a>
### A. Tugas di MASTER NODE - Persiapan dan Konfigurasi Inti

Lakukan semua langkah berikut di **Master Node**.

1.  **Update Sistem Operasi**:
    ```bash
    sudo apt update && sudo apt upgrade -y
    ```
    *(Opsional, tergantung kondisi. Untuk memastikan semua paket terbaru)*

2.  **Instal Java Development Kit (OpenJDK 11)**:
    ```bash
    sudo apt install openjdk-11-jdk -y
    ```

3.  **Buat User dan Grup Khusus untuk Hadoop**:
    ```bash
    sudo addgroup hadoop
    sudo adduser --ingroup hadoop hduser
    ```
    * Ikuti prompt untuk membuat password dan mengisi info (bisa dilewati).
    * Opsional, agar `hduser` bisa menjalankan perintah `sudo` tanpa password:
        ```bash
        sudo usermod -aG sudo hduser
        ```
    * Disarankan menjalankan Hadoop dengan user terpisah (`hduser`).

4.  **Login sebagai `hduser`**:
    ```bash
    su - hduser
    ```
    * Semua command Hadoop selanjutnya dijalankan sebagai user ini. Prompt akan berubah (misal `hduser@ubuntu $`).

5.  **Set Variabel Lingkungan untuk `hduser`**:
    Edit `~/.bashrc`:
    ```bash
    nano ~/.bashrc
    ```
    Tambahkan di akhir file:
    ```bash
    export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
    export HADOOP_HOME=/usr/local/hadoop
    export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
    export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$JAVA_HOME/bin
    # Tambahkan variabel lain seperti HADOOP_MAPRED_HOME, HADOOP_COMMON_HOME, HADOOP_HDFS_HOME, YARN_HOME jika diperlukan
    export HADOOP_OPTS="-Djava.library.path=$HADOOP_HOME/lib/native"
    ```
    Terapkan perubahan:
    ```bash
    source ~/.bashrc
    ```

6.  **Buat Direktori Penyimpanan Data Hadoop di Master**:
    ```bash
    mkdir -p /home/hduser/hadoop_data/tmp_dir
    mkdir -p /home/hduser/hadoop_data/namenode_dir
    ```
    * Nama direktori `namenode_dir` dan `tmp_dir` boleh diubah, tapi wajib diingat untuk konsistensi.

7.  **Buat Pasangan Kunci SSH untuk `hduser` (SSH Tanpa Password)**:
    ```bash
    ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
    ```
    * Tekan Enter untuk semua prompt jika ditanya passphrase (biarkan kosong).

8.  **Otorisasi SSH ke Diri Sendiri (localhost/master)**:
    ```bash
    cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
    chmod 700 ~/.ssh
    chmod 600 ~/.ssh/authorized_keys
    ```

9.  **Unduh, Verifikasi, dan Ekstrak Hadoop**:
    Pindah ke direktori home:
    ```bash
    cd ~
    ```
    Unduh Hadoop (contoh versi 3.4.1, sesuaikan jika perlu):
    ```bash
    wget [https://dlcdn.apache.org/hadoop/common/hadoop-3.4.1/hadoop-3.4.1.tar.gz](https://dlcdn.apache.org/hadoop/common/hadoop-3.4.1/hadoop-3.4.1.tar.gz)
    wget [https://dlcdn.apache.org/hadoop/common/hadoop-3.4.1/hadoop-3.4.1.tar.gz.sha512](https://dlcdn.apache.org/hadoop/common/hadoop-3.4.1/hadoop-3.4.1.tar.gz.sha512)
    ```
    Verifikasi checksum:
    ```bash
    sha512sum -c hadoop-3.4.1.tar.gz.sha512
    ```
    * Pastikan outputnya `OK`.
    Ekstrak ke `/usr/local/`:
    ```bash
    sudo tar -xzvf hadoop-3.4.1.tar.gz -C /usr/local/
    ```
    Rename direktori:
    ```bash
    sudo mv /usr/local/hadoop-3.4.1 /usr/local/hadoop
    ```
    Ubah kepemilikan:
    ```bash
    sudo chown -R hduser:hadoop /usr/local/hadoop
    ```

10. **Edit File Konfigurasi Inti Hadoop**:
    Pindah ke direktori konfigurasi:
    ```bash
    cd $HADOOP_HOME/etc/hadoop
    ```
    * **Edit `hadoop-env.sh`**:
        ```bash
        nano hadoop-env.sh 
        ```
        *(Gunakan `sudo nano` jika file unwriteable)*
        Pastikan baris `export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64` (atau path Java Anda) tidak dikomentari (`#`) dan sudah benar.
        Tambahkan atau pastikan ada:
        ```bash
        export HADOOP_CONF_DIR="/usr/local/hadoop/etc/hadoop"
        ```
    * **Edit `core-site.xml`**:
        ```bash
        nano core-site.xml
        ```
        Isi:
        ```xml
        <configuration>
            <property>
                <name>fs.defaultFS</name>
                <value>hdfs://master:9000</value>
            </property>
            <property>
                <name>hadoop.tmp.dir</name>
                <value>/home/hduser/hadoop_data/tmp_dir</value>
            </property>
        </configuration>
        ```
    * **Edit `hdfs-site.xml`**:
        ```bash
        nano hdfs-site.xml
        ```
        Isi:
        ```xml
        <configuration>
            <property>
                <name>dfs.replication</name>
                <value>2</value> 
            </property>
            <property>
                <name>dfs.namenode.name.dir</name>
                <value>file:///home/hduser/hadoop_data/namenode_dir</value>
            </property>
            <property>
                <name>dfs.datanode.data.dir</name>
                <value>file:///home/hduser/hadoop_data/datanode_dir</value>
            </property>
            <property>
                <name>dfs.namenode.secondary.http-address</name>
                <value>master:9868</value>
            </property>
        </configuration>
        ```
    * **Edit `mapred-site.xml`**:
        Salin dari template jika perlu:
        ```bash
        cp mapred-site.xml.template mapred-site.xml
        nano mapred-site.xml
        ```
        Isi:
        ```xml
        <configuration>
            <property>
                <name>mapreduce.framework.name</name>
                <value>yarn</value>
            </property>
            <property>
                <name>mapreduce.application.classpath</name>
                <value>$HADOOP_HOME/share/hadoop/mapreduce/*:$HADOOP_HOME/share/hadoop/mapreduce/lib/*</value>
            </property>
        </configuration>
        ```
    * **Edit `yarn-site.xml`**:
        ```bash
        nano yarn-site.xml
        ```
        Isi:
        ```xml
        <configuration>
            <property>
                <name>yarn.nodemanager.aux-services</name>
                <value>mapreduce_shuffle</value>
            </property>
            <property>
                <name>yarn.resourcemanager.hostname</name>
                <value>master</value>
            </property>
            <property>
                <name>yarn.nodemanager.resource.memory-mb</name>
                <value>3072</value> 
            </property>
            <property>
                <name>yarn.nodemanager.resource.cpu-vcores</name>
                <value>2</value> 
            </property>
            <property>
                <name>yarn.log-aggregation-enable</name>
                <value>true</value>
            </property>
            <property>
                <name>yarn.log.server.url</name>
                <value>http://master:19888/jobhistory/logs</value>
            </property>
            <property>
                <name>yarn.web-proxy.address</name>
                <value>master:8081</value>
            </property>
        </configuration>
        ```
    * **Edit `workers`**:
        ```bash
        nano workers
        ```
        Hapus isi default (biasanya `localhost`), dan tambahkan hostname slave Anda, satu per baris:
        ```
        slave1
        slave2
        ```

---

<a name="b-tugas-di-slave1-node---persiapan-awal-id"></a>
### B. Tugas di SLAVE1 NODE - Persiapan Awal

Lakukan semua langkah berikut di **Slave1 Node**. (Login sebagai user utama, bukan `hduser` kecuali disebutkan).

1.  **Update Sistem Operasi** (Sama seperti Master Node - Langkah A.1)
2.  **Instal Java Development Kit (OpenJDK 11)** (Sama seperti Master Node - Langkah A.2)
3.  **Buat User dan Grup Khusus untuk Hadoop** (Sama seperti Master Node - Langkah A.3)
4.  **Login sebagai `hduser`** (Sama seperti Master Node - Langkah A.4)
    ```bash
    su - hduser
    ```
5.  **Set Variabel Lingkungan untuk `hduser`**:
    Edit `~/.bashrc`:
    ```bash
    nano ~/.bashrc
    ```
    Tambahkan baris yang sama persis seperti di Master Node untuk `JAVA_HOME`, `HADOOP_HOME`, `PATH`, `HADOOP_OPTS`, dll.
    Terapkan perubahan:
    ```bash
    source ~/.bashrc
    ```
6.  **Buat Direktori Penyimpanan Data Hadoop di Slave1**:
    (Sebagai `hduser`)
    ```bash
    mkdir -p /home/hduser/hadoop_data/tmp_dir
    mkdir -p /home/hduser/hadoop_data/datanode_dir
    ```
7.  **Persiapan Direktori `.ssh` untuk Menerima Kunci dari Master**:
    (Sebagai `hduser`)
    ```bash
    rm -f ~/.ssh/authorized_keys 
    mkdir -p ~/.ssh
    chmod 700 ~/.ssh
    ```
    *(Perintah `rm` untuk menghapus kunci lama jika ada)*
8.  **Persiapan Direktori Target untuk Instalasi Hadoop**:
    (Kembali ke user utama Anda jika perlu, atau gunakan `sudo` sebagai `hduser` jika sudah dikonfigurasi di `sudoers`)
    ```bash
    sudo mkdir -p /usr/local/hadoop
    sudo chown hduser:hadoop /usr/local/hadoop
    ```

---

<a name="c-tugas-di-slave2-node-id"></a>
### C. Tugas di SLAVE2 NODE

Lakukan langkah-langkah yang **SAMA PERSIS** seperti pada **SLAVE1 NODE** (Bagian B).
**Perhatian**: Teliti dan pastikan tidak ada kesalahan konfigurasi antara Slave1 dan Slave2.

---

<a name="d-tugas-di-master-node--distribusi-konfigurasi-dan-menjalankan-klaster-id"></a>
### D. Tugas di MASTER NODE  - Distribusi Konfigurasi dan Menjalankan Klaster

Kembali ke **Master Node** dan pastikan Anda login sebagai `hduser`.

1.  **Salin Kunci Publik SSH Master ke Semua Slave Node**:
    ```bash
    ssh-copy-id hduser@slave1
    # Masukkan password hduser untuk slave1 jika diminta (password user hduser di slave1)
    ssh-copy-id hduser@slave2
    # Masukkan password hduser untuk slave2 jika diminta (password user hduser di slave2)
    ```
    * Jawab `yes` jika ada prompt keaslian host.
    * Contoh output saat `ssh-copy-id`:
        ```
        master@ubuntu $ ssh-copy-id hduser@slave1
        /usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/home/hduser/.ssh/id_rsa.pub"
        The authenticity of host 'slave1 (192.168.159.129)' can't be established.
        ED25519 key fingerprint is SHA256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx.
        This key is not known by any other names.
        Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
        /usr/bin/ssh-copy-id: INFO: 1 key(s)
        ```

2.  **Salin Seluruh Direktori Instalasi Hadoop ke Slave Node**:
    ```bash
    scp -rp /usr/local/hadoop hduser@slave1:/usr/local/
    scp -rp /usr/local/hadoop hduser@slave2:/usr/local/
    ```
    * Opsi `-p` menjaga izin file.

3.  **Format HDFS NameNode (Tindakan Sekali Pakai!)**:
    ```bash
    hdfs namenode -format
    ```
    * **PENTING**: Periksa outputnya! Pastikan tidak ada error dan NameNode mengidentifikasi dirinya dengan IP LAN Master yang benar (bukan `127.0.0.1` atau `127.0.1.1`). Jika salah, periksa konfigurasi IP dan `/etc/hosts`.

4.  **Mulai Layanan HDFS**:
    ```bash
    start-dfs.sh
    ```

5.  **Mulai Layanan YARN**:
    ```bash
    start-yarn.sh
    ```

6.  **Mulai MapReduce JobHistory Server (Opsional)**:
    ```bash
    mr-jobhistory-daemon.sh start historyserver
    ```

---

<a name="e-verifikasi-klaster-id"></a>
### E. Verifikasi Klaster (Dilakukan di Node yang Sesuai)

1.  **Cek Proses Java yang Berjalan (`jps`)**:
    * **Di Master Node**: Jalankan `jps`. Harus ada: `NameNode`, `SecondaryNameNode`, `ResourceManager`, `JobHistoryServer` (jika dimulai), `Jps`.
    * **Di Slave1 Node**: Login (`ssh hduser@slave1`) dan jalankan `jps`. Harus ada: `DataNode`, `NodeManager`, `Jps`.
    * **Di Slave2 Node**: Login (`ssh hduser@slave2`) dan jalankan `jps`. Harus ada: `DataNode`, `NodeManager`, `Jps`.

2.  **Akses Web UI Hadoop**:
    Buka browser di komputer Anda dan akses URL berikut (ganti `<IP_MASTER>` dengan IP statis Master Node Anda):
    * **HDFS NameNode Overview**: `http://<IP_MASTER>:9870`
        * Periksa bagian "Datanode Information" -> "Live Nodes". Jumlahnya harus `2`.
    * **YARN ResourceManager Overview**: `http://<IP_MASTER>:8088`
        * Klik "Nodes" di menu. Harus ada `2` node aktif (slave1 dan slave2) dengan status `RUNNING`.
    * **MapReduce JobHistory Server**: `http://<IP_MASTER>:19888`
    * **Firewall**: Pastikan firewall di Master Node mengizinkan port-port ini.
        Cek status firewall:
        ```bash
        sudo ufw status verbose
        ```
        Jika tidak aktif dan ingin diaktifkan:
        ```bash
        sudo ufw enable
        ```
        Allow port yang dibutuhkan (jika belum ada di list):
        ```bash
        sudo ufw allow 22/tcp   # SSH (PENTING!)
        sudo ufw allow 9870/tcp # HDFS NameNode UI
        sudo ufw allow 8088/tcp # YARN ResourceManager UI
        sudo ufw allow 19888/tcp# MapReduce JobHistory Server UI
        sudo ufw allow 8030/tcp # YARN RPC
        sudo ufw allow 8031/tcp # YARN RPC
        sudo ufw allow 8032/tcp # YARN RPC
        sudo ufw reload
        ```

3.  **Tes Operasi Dasar HDFS**:
    Dari Master Node, sebagai `hduser`:
    ```bash
    hdfs dfs -mkdir -p /user/hduser/test_rekap
    echo "Tes rekap HDFS berhasil pada $(date)" > ~/file_rekap.txt
    hdfs dfs -put ~/file_rekap.txt /user/hduser/test_rekap/
    hdfs dfs -ls /user/hduser/test_rekap/
    hdfs dfs -cat /user/hduser/test_rekap/file_rekap.txt
    ```

4.  **Jalankan Contoh Job MapReduce (WordCount)**:
    Siapkan file input (misalnya `buku.txt`) di HDFS. Contoh path: `/user/hduser/wordcount_rekap/input/buku.txt`.
    Buat direktori input di HDFS dan upload file:
    ```bash
    hdfs dfs -mkdir -p /user/hduser/wordcount_rekap/input
    # Misal Anda punya file buku.txt di home directory hduser
    hdfs dfs -put ~/buku.txt /user/hduser/wordcount_rekap/input/
    ```
    Hapus direktori output lama jika ada:
    ```bash
    hdfs dfs -rm -r /user/hduser/wordcount_rekap/output
    ```
    Jalankan job:
    ```bash
    hadoop jar $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-examples-*.jar wordcount /user/hduser/wordcount_rekap/input /user/hduser/wordcount_rekap/output
    ```
    Pantau progres di terminal dan YARN UI. Setelah selesai (`SUCCEEDED`), periksa hasilnya:
    ```bash
    hdfs dfs -cat /user/hduser/wordcount_rekap/output/part-r-00000
    ```

---

<a name="catatan-penting-notes-id"></a>
### Catatan Penting (Notes)

* `sudo` digunakan jika file environment tidak bisa ditulis atau `unwriteable`.
* Perhatikan **IP ADDRESS** setiap komputer atau VM. Tabrakan IP akan menyebabkan komunikasi gagal.
* Command bisa dieksekusi sekaligus (dipisah `&&`), tapi hati-hati.
* Jika menemukan command `rm -rf`, perhatikan baik-baik direktori atau file yang akan dihapus. Kesalahan bisa fatal.
* Saat mengedit file konfigurasi (`<configuration>...</configuration>`), disarankan menghapus blok `<configuration>...</configuration>` yang ada (jika ada) lalu copas seluruh konten XML yang baru.
* Untuk copy-paste di terminal Linux, gunakan `CTRL+SHIFT+C` (copy) dan `CTRL+SHIFT+V` (paste). Jangan gunakan `CTRL+C`/`CTRL+V` biasa karena `CTRL+C` adalah perintah interupsi.
* File konfigurasi XML sangat sensitif terhadap syntax (titik, koma, spasi, tag pembuka/penutup). Teliti!
* Tidak disarankan menggunakan 3 VM di laptop pribadi jika spesifikasinya terbatas. Hadoop lumayan berat. Pemahaman dasar networking (NAT, IPv4) juga penting.
* Linux itu seni, perlu ketelitian dan perhatian lebih. Terlena sedikit bisa merusak banyak.
  
