# VPS搭建WordPress：LEMP环境从零部署全流程，新手照做一次跑通，配置和VPS这样选

VPS搭建WordPress 这件事，拆开看其实就五步：装 LEMP 环境、建数据库、下载程序、配置 Nginx、签一张免费 SSL 证书。命令都不难，真正让人卡住的是三个前置选择——买多大内存的机器、选哪个系统镜像、用哪家的 VPS。这三个问题先解决，后面一路都是顺的。

下面这套流程基于 Ubuntu 24.04 LTS 和 PHP 8.3，从一台刚开机的裸机写到 HTTPS 上线。至于为什么值得从虚拟主机搬出来：同样是打开一个页面，调好缓存的 VPS 首字节时间能压到 200ms 以内，普通共享主机普遍在 600–1500ms，这个差距会实打实影响搜索排名和转化率。

> **赶时间版**：个人博客、作品集，选 1GB 年付套餐（$10.99/年）配 FastCGI 缓存就够用；要跑 WooCommerce 或打算一台机器多站共存，直接看 2GB（$21.99/年）或 4GB（$39.99/年）。都是 KVM 虚拟化、带完整 root 权限。👉 [查看 RackNerd 全部年付套餐价格](https://bit.ly/my-rn)

## 多大配置才够 WordPress 用？

1GB 内存就能跑 WordPress，配合服务器端缓存，个人博客和作品集完全够；WooCommerce 从 2GB 起步，想稳妥就上 4GB。

具体拆开说：

- 底线很低：1 核 CPU、1GB 内存、10GB 硬盘就装得起来，连 GitHub 上的开源一键部署方案给出的最低要求都是这个数。裸跑没问题，但缓存一定要做。
- 2GB 是稳妥的入门线：数据库查询多、插件装得多的站点（典型如 WooCommerce 商店），1GB 会经常吃紧。
- 4GB 是“一台机器养多个站”的门槛：Nginx + FastCGI 缓存的架构下，跑 10–25 个中低流量站点是常规操作，每个站一个 server block 加独立数据库就行。

PHP 版本别省事：选 Ubuntu 24.04 LTS，默认源直接就是 PHP 8.3，不用手动加仓库。有些教程还在教 Ubuntu 22.04 + PHP 8.1 的组合，能跑，但没必要；至于 PHP 7.4，安全更新停了好几年，还在推荐它的教程可以直接关掉。

这几档配置 RackNerd 都有对应的年付套餐，价格放在下一节。

## 主机怎么选？这次为什么推荐 RackNerd

同样配置下，RackNerd 的年付促销价便宜一个量级，代价是全自助运维——装系统、加固、备份都得自己来。如果你愿意照着本文敲命令，这个代价值得付。

RackNerd 是一家主打低价年付 KVM VPS 的美国商家，机房覆盖洛杉矶、圣何塞、西雅图、达拉斯、芝加哥、纽约、亚特兰大和荷兰阿姆斯特丹等节点，下单时自选。每个套餐给独立 IPv4、1Gbps 端口，SolusVM 面板里可以随时重装系统。目前促销区长期挂着这几档：

| 套餐 | 内存 | vCPU | SSD | 月流量 | 年付价（美元） |
| --- | --- | --- | --- | --- | --- |
| 1GB KVM | 1 GB | 1 核 | 20 GB | 1 TB | $10.99（ [年付购买](https://bit.ly/my-rn)） |
| 2GB KVM | 2 GB | 2 核 | 35 GB | 3 TB | $21.99（ [年付购买](https://bit.ly/my-rn)） |
| 2.5GB KVM | 2.5 GB | 2 核 | 50 GB | 3.5 TB | $24.99（ [年付购买](https://bit.ly/my-rn)） |
| 3GB KVM | 3 GB | 3 核 | 50 GB | 4 TB | $32.99（ [年付购买](https://bit.ly/my-rn)） |
| 4GB KVM | 4 GB | 4 核 | 65 GB | 5 TB | $39.99（ [年付购买](https://bit.ly/my-rn)） |
| 5GB KVM | 5 GB | 4 核 | 80 GB | 6 TB | $54.99（ [年付购买](https://bit.ly/my-rn)） |
| 6GB KVM | 6 GB | 5 核 | 90 GB | 7 TB | $64.99（ [年付购买](https://bit.ly/my-rn)） |

促销档位和价格会随活动小幅浮动，购物车里有时还有两年付选项、均价更低，下单前以实际结算价为准。这些低价套餐只提供 Linux 系统（Ubuntu、Debian、AlmaLinux 等自选），Windows 需求要走它家另一条产品线，价格完全不同。

这笔账值得算一下。同样是 2GB 内存：RackNerd 年付 $21.99，摊到每月不到 $2；DejaVuHost 的 2GB 档（VPS Start）月付 $24.99，一年就是 $300 上下。Hostinger 的 KVM 2 是另一台常被推荐的入门机，长约折合每月 $5–7，配置不差，但到期续费会明显跳价。只看持有成本的话，年付促销的差距很难被别的因素抹平。👉 [用年付价开通 RackNerd 2GB 套餐](https://bit.ly/my-rn)

也说说它不适合谁：

- 要国内访客直连秒开：RackNerd 走普通国际线路，西海岸节点（洛杉矶、圣何塞）到国内的延迟通常好于东海岸，但它本质是便宜的美国 VPS，认真做国内体验得另配 CDN。
- 完全不想碰命令行：买 Cloudways 这类托管方案，运维由商家代管，贵但省心。
- 想要工单秒回的保姆服务：低价自助 VPS 的支持以基础问题为主，别抱 managed hosting 的预期。

## 部署前，先把三件事做掉

域名解析要提前。去域名后台加一条 A 记录指向 VPS 的 IPv4 地址，DNS 生效最长可能等 24 小时，别拖到签证书那一步才想起它。

然后是服务器基础加固。SSH 登录后先更新系统、建一个普通用户，别一直用 root 干活：

bash
ssh root@你的服务器IP
apt update && apt upgrade -y
adduser deploy
usermod -aG sudo deploy


用 deploy 重新登录后开防火墙，只放行 22、80、443 三个端口：

bash
sudo ufw allow OpenSSH
sudo ufw allow 80
sudo ufw allow 443
sudo ufw enable


## 安装 LEMP 环境

一条命令装齐 Nginx、MariaDB 和 PHP 8.3 全套扩展：

bash
sudo apt install nginx mariadb-server php8.3-fpm php8.3-mysql \
  php8.3-curl php8.3-gd php8.3-mbstring php8.3-xml \
  php8.3-zip php8.3-intl -y


确认服务状态：

bash
systemctl status nginx php8.3-fpm mariadb


MariaDB 和 MySQL 对 WordPress 来说完全等价，用哪个都行。顺手改两个 PHP 参数（`/etc/php/8.3/fpm/php.ini`）：把 `upload_max_filesize` 和 `post_max_size` 提到 64M，否则以后传大点的主题或插件会被默认的 2M 上限卡住。改完重启 PHP-FPM：

bash
sudo systemctl restart php8.3-fpm


## 建数据库，给 WordPress 一个专用账号

先跑一遍安全脚本，按默认走即可——移除匿名用户、禁止 root 远程登录、删除测试库：

bash
sudo mysql_secure_installation


然后建库。两个细节：字符集用 utf8mb4，不然连 emoji 都存不进去；账号只授权给这一个库，别图省事直接让应用连 root：

sql
CREATE DATABASE wp_production CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'wp_user'@'localhost' IDENTIFIED BY '换成你自己的强密码';
GRANT ALL PRIVILEGES ON wp_production.* TO 'wp_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;


库名、用户名、密码记下来，安装向导第 2 步要用。

## 下载 WordPress 并写好配置

bash
sudo mkdir -p /var/www/example.com
cd /tmp
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
sudo cp -a wordpress/. /var/www/example.com/
sudo chown -R www-data:www-data /var/www/example.com


进入目录后把 `wp-config-sample.php` 复制为 `wp-config.php`，填入数据库三件套；文件里的认证密钥段别留默认值，WordPress 官方的 salt 生成服务会输出一整段现成密钥，整段替换进去就行。

## Nginx 站点配置与 FastCGI 缓存

先建缓存目录，在 `/etc/nginx/nginx.conf` 的 `http {}` 块里加一行：

nginx
fastcgi_cache_path /var/cache/nginx levels=1:2 keys_zone=WORDPRESS:100m inactive=60m;


站点配置写到 `/etc/nginx/sites-available/example.com`：

nginx
server {
    listen 80;
    server_name example.com www.example.com;
    root /var/www/example.com;
    index index.php;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
        fastcgi_cache WORDPRESS;
        fastcgi_cache_valid 200 301 302 60m;
        fastcgi_cache_use_stale error timeout updating;
        add_header X-Cache $upstream_cache_status;
    }

    location ~ /\.ht { deny all; }
}


启用并检查：

bash
sudo ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx


这是能跑的精简版。严谨一点还差几条 map 规则，让登录用户和评论者的请求绕过缓存——这正是后面 Nginx Helper 插件要配合的事。服务器端缓存的价值很直接：页面从内存里直接吐出去，几乎不消耗 PHP 进程，小内存 VPS 能扛住突发流量靠的就是它。

## 签一张免费的 Let's Encrypt 证书

bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d example.com -d www.example.com


Certbot 会自动改好 Nginx 配置、把 HTTP 跳转到 HTTPS，并挂上自动续期的定时任务。想确认续期没问题：`sudo certbot renew --dry-run`。前提是第一步的域名解析已经生效，否则证书签不下来。

## 跑完五分钟安装向导

浏览器打开 `https://example.com`，按向导走：选语言 → 填数据库信息 → 设站点标题和管理员账号 → 安装。后台地址是 `/wp-admin`。

装完先别急着发文章。到「插件 → 安装插件」搜 Nginx Helper，启用后在设置里勾选 Enable Purge、模式选 nginx Fastcgi map。之后每次发布或编辑文章，它都会自动清掉服务器端缓存，不需要你手动 reload 任何东西。

流量再往上涨，可以加 Redis 做对象缓存，把数据库查询结果放进内存，活跃站点的 MySQL 压力能降一大截。刚上线阶段，FastCGI 缓存加这一层就够了。

## 几个常被问到的问题

**Apache 还是 Nginx？** 小内存 VPS 选 Nginx。它的事件驱动架构扛并发更省内存，还能在服务器层直接缓存 PHP 输出；Apache 的传统模型每个访客都要占进程，同配置下余量小一些。

**还要装 WP Super Cache 这类缓存插件吗？** 服务器层已有 FastCGI 缓存的话，不用再叠页面缓存插件，留 Nginx Helper 负责清缓存就够。缓存叠太多层，排障时只会更乱。

**用 Docker 一键部署行不行？** 行。GitHub 上有现成的开源项目，把 Nginx、WordPress、MySQL 打成容器，Certbot 自动续证书，适合以后要反复部署、想把配置纳入版本管理的人。它对机器的最低要求同样是 1 核 1GB，省钱效果和手动装没区别。第一次搭，更建议手动走一遍，过程中学到的东西以后都用得上。

**RackNerd 的促销价续费会涨吗？** 这类年付促销按购入价续费，没有“首年低价、次年翻倍”那一套；下单页面能看到续费单价，付款前核对一眼即可。👉 [查看 RackNerd 当前促销与续费价](https://bit.ly/my-rn)

## 按你的情况做最后选择

只想有个自己的博客、预算卡得很死：1GB 年付（$10.99）加 FastCGI 缓存，够用很久。
要上 WooCommerce，或计划一台机器放两三个站：2GB（$21.99）起步，预算宽裕直接 4GB（$39.99）一步到位。
完全不想碰命令行：别勉强，托管型方案更合适，这笔钱花在别人替你运维上更值。

下单前核对两件事：机房位置选离访客近的（面向国内读者一般选洛杉矶或圣何塞）；购物车里确认账单周期是 Annually、续费单价没变。剩下的，就是照着上面的命令一路跑到底。👉 [挑一台 RackNerd 年付 VPS，今晚把站搭起来](https://bit.ly/my-rn)
