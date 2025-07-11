+++
date = '2025-07-11T10:25:29+08:00'
draft = false
title = 'Ubuntu仓库构建'
+++
# 构建Ubuntu/ Debian软件仓库 

## 安装依赖项

```bash
sudo apt install gcc dpkg-dev gpg nginx
```

## 构建创库

创建存放软件包的仓库目录

```bash
mkdir -p  ~/example/apt_repo/pool/main
```

将对应软件包存放到该目录下：

*大的仓库管理通常在下面创建子文件夹存放对应分类或项目的软件包*

```bash
cd  ~/example/apt_repo/pool/main && mkdir [target_sub_direcotry]
```

将软件包存放到对应的子目录:

```bash
cp [all_deb_files or holding directories]  [target_sub_direcotry]
```

依据架构支持创建存放包元数据的目录

```bash
cd ~/example/apt-repo &&
```

返回到apt-repo层目录,依据架构创建存放元包数据的目录

```bash
cd ~/example/apt-repo && mkdir -p ./dists/stable/main/binary-arm64
```

使用**dpkg-scanpackages** 生成**Packages**记录文件

```bash
dpkg-scanpackages --arch arm64 pool/ > dists/stable/main/binary-arm64/Packages
```

对Packages文件进行压缩

```bash
cat dists/stable/main/binary-arm64/Packages|gzip -9 >dists/stable/main/binary-a
rm64/Packages.gz 
```

*Note:支持的压缩格式：bzip2,lzma,*

创建Release文件

脚本程序，放到example目录下

generate_release.sh

```bash
#!/bin/sh
set -e

do_hash() {
    HASH_NAME=$1
    HASH_CMD=$2
    echo "${HASH_NAME}:"
    for f in $(find -type f); do
        f=$(echo $f | cut -c3-) # remove ./ prefix
        if [ "$f" = "Release" ]; then
            continue
        fi
        echo " $(${HASH_CMD} ${f}  | cut -d" " -f1) $(wc -c $f)"
    done
}

cat << EOF
Origin: Example Repository
Label: Example
Suite: stable
Codename: stable
Version: 1.0
Architectures: amd64 arm64 arm7
Components: main
Description: An example software repository
Date: $(date -Ru)
EOF
do_hash "MD5Sum" "md5sum"
do_hash "SHA1" "sha1sum"
do_hash "SHA256" "sha256sum"
```

运行脚本

```bash
cd ~/example/apt_repo/dists/stable
~/example/generate_release.sh > Release
```

## 仓库Release file签名

```bash
echo "%echo Generating an example PGP key
Key-Type: RSA
Key-Length: 4096
Name-Real: example
Name-Email: yongqi.chen@skysys.cn
Expire-Date: 0
%no-ask-passphrase
%no-protection
%commit" > /tmp/example-pgp-key.batch
```

```bash
export GNUPGHOME="$(mktemp -d ~/example/pgpkeys-XXXXXX)"
gpg --no-tty --batch --gen-key /tmp/example-pgp-key.batch
```

查看生成的密匙

```bash
gpg --list-keys
```

导出公钥

```bash
gpg --armor --export example > ~/example/pgp-key.public
```

可通过如下指令查看确保导出单个公钥

```bash
cat ~/example/pgp-key.public | gpg --list-packets
```

导出私钥

```bash
gpg --armor --export-secret-keys example > ~/example/pgp-key.private
```

对release 文件进行签名，在开始用生成的密钥进行签名前，需要确保可以导出备份。定义新的GPG密钥位置

```bash
export GNUPGHOME="$(mktemp -d ~/example/pgpkeys-XXXXXX)"
```

查看密钥

```bash
gpg --list-keys
```

显示如下：

-----------------------------------------------------------------------------

gpg: keybox '/home/ubuntu/example/pgpkeys-bJjHJv/pubring.kbx' created
gpg: /home/ubuntu/example/pgpkeys-bJjHJv/trustdb.gpg: trustdb created

导入备份的私钥

```bash
cat ~/example/pgp-key.private | gpg --import
```

现在查看密钥显示应如下：

/home/ubuntu/example/pgpkeys-bJjHJv/pubring.kbx                                            
-----------------------------------------------                                            
pub   rsa4096 2024-04-08 [SCEA]                                                            
      51E07666AB32B2F6E1996DAABC73497279AA9C24                                             
uid           [ unknown] example                               

对release file 进行签名

```bash
cat ~/example/apt_repo/dists/stable/Release | gpg --default-key example -abs > ~/example/apt_repo/dists/stable/Release.gpg
```

创建InRelease 文件加速apt 获取速度

```bash
cat ~/example/apt_repo/dists/stable/Release | gpg --default-key example -abs --clearsign > ~/example/apt_repo/dists/stable/InRelease
```

生成apt仓库源

```bash
echo "deb [arch=arm64] http://[ip]:[port]/apt_repo stable main" | sudo tee /etc/apt/sources.list.d/example.list
```

## 配置NGINX 代理

配置文件在  **/etc/nginx/sites-available/default**

根据提示配置http服务

## 软件包更新自动维护程序

```
from hashlib import sha256
import os
import subprocess
from rich.console import Console
from datetime import datetime
from time import sleep
DIRECTORY="/home/ubuntu/example/apt_repo/pool"
OUTFILE="hashes.txt"

ROOT_DIRECTORY= "/home/ubuntu/example/apt_repo"

LOG_DIRECTORY="/tmp/log/maintaince_tool/"

log_file ="log-"+datetime.strftime(datetime.now(),"%YY-%m-%d-%H-%M-%S")+".txt"

log_file =os.path.join(LOG_DIRECTORY,log_file)

logger=Console(file=log_file)

private_key= '''
-----BEGIN PGP PRIVATE KEY BLOCK-----
xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
-----END PGP PRIVATE KEY BLOCK-----
'''  #replace your private key.

def calculate_hash(file_path, hash_algorithm="sha256"):
    """Calculate hash for a file."""
    hasher = sha256()
    with open(file_path, "rb") as f:
        while True:
            chunk = f.read(4096)
            if not chunk:
                break
            hasher.update(chunk)
    return hasher.hexdigest()

def generate_hashes(directory, output_file, hash_algorithm="sha256"):
    """Generate hashes for all files in a directory and save to output file."""
    hashes=[]
    with open(output_file, "w") as f:
        for root, dirs, files in os.walk(directory):
            for file in files:
                file_path = os.path.join(root, file)
                file_hash = calculate_hash(file_path, hash_algorithm)
                hashes.append(file_hash)
    return hashes

def save_hash_to_file(hash_file,hashes):
    with open(hash_file,"w")as fp:
        fp.writelines(hashes)
        

def load_hashes(hash_file):
    with open(hash_file,"r")as fp:
        hashes=fp.readlines()
                
    return hashes



if __name__=="__main__":
    # create log directory
    if not os.path.exists(LOG_DIRECTORY):
        os.makedirs(LOG_DIRECTORY)
        
    # load hashes
    hash_file=os.path.join(DIRECTORY,OUTFILE)
    hashes=[]
    # 
    if not os.path.exists(hash_file):
        hashes=generate_hashes(DIRECTORY,hash_file)
        save_hash_to_file(hash_file,hashes)
    else:
        hashes=load_hashes(hash_file)
    
    # maintaining progress.
    while True:
        new_hashes=generate_hashes(DIRECTORY,hash_file)
        if hashes != new_hashes: # repository has been updated
            # 1.Regenerate Packages File
                shell_scripts=f"/usr/bin/bash -c ' cd {ROOT_DIRECTORY} &&  dpkg-scanpackages --arch arm64 pool/ > ./dists/stable/main/binary-arm64/Packages && cat dists/stable/main/binary-arm64/Packages|gzip -9 > dists/stable/main/binary-arm64/Packages.gz '"
                ret=subprocess.call(shell_scripts,shell=True)
                if ret!=0:
                    # log errors
                    logger.log("[#ff0000]Regenerate Packages file failed.[ff0000]")
                    pass
            # 2. Regenerate Release file
                shell_scripts=f"/usr/bin/bash -c 'cd {os.path.join(ROOT_DIRECTORY,'''dists/stable''')} && ~/example/generate_release.sh > Release'"
                ret=subprocess.call(shell_scripts,shell=True)
                if ret!=0:
                    logger.log("[#ff0000]Regenerate Release file failed.[ff0000]")
                    pass
            # 3. sign the release file with private-key
                shell_scripts=f"/usr/bin/bash -c 'gpg --import << f{private_key}  && cat {os.path.join(ROOT_DIRECTORY,'''dists/stable/Release''')} | gpg --default --key example -abs > {os.path.join(ROOT_DIRECTORY,'''dists/stable/Release.gpg''')}' "
                ret=subprocess.call(shell_scripts,shell=True)
                if ret!=0:
                     logger.log("[#ff0000]Sign Release file failed.[ff0000]")
            # 4. create InRelease file
                shell_scripts=f"/usr/bin/bash -c 'gpg --import << f{private_key} && cat {os.path.join(ROOT_DIRECTORY,'''dists/stable/Release''')} | gpg --default --key example -abs --clearsign > {os.path.join(ROOT_DIRECTORY,'''dists/stable/InRelease''')}' "
                ret=subprocess.call(shell_scripts,shell=True)
                if ret!=0:
                    logger.log("[#ff0000]Create InRelease file failed.[ff0000]")
            #save new hash to file
                hashes=new_hashes
                save_hash_to_file(hash_file,hashes)
        sleep(10)
```
