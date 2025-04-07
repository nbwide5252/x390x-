#!/bin/bash

# 脚本用途：在s390x服务器上自动编译并运行Xray
# 当前日期：2025年4月7日

# 设置变量
XRAY_VERSION="v1.8.23"  # 可根据需要修改为最新版本
INSTALL_DIR="/usr/local/xray"
CONFIG_DIR="/etc/xray"
GO_VERSION="1.22.2"     # Go语言版本，确保兼容Xray编译

# 检查是否以root权限运行
if [ "$EUID" -ne 0 ]; then
    echo "请以root权限运行此脚本：sudo bash $0"
    exit 1
fi

# 更新系统并安装基本依赖
echo "正在更新系统并安装依赖..."
apt-get update -y || yum update -y
apt-get install -y git curl unzip build-essential || yum install -y git curl unzip gcc make

# 检查并安装Go语言环境
if ! command -v go &> /dev/null; then
    echo "未检测到Go环境，正在安装Go $GO_VERSION..."
    curl -LO "https://golang.org/dl/go${GO_VERSION}.linux-s390x.tar.gz"
    tar -C /usr/local -xzf "go${GO_VERSION}.linux-s390x.tar.gz"
    echo "export PATH=\$PATH:/usr/local/go/bin" >> /etc/profile
    source /etc/profile
    rm "go${GO_VERSION}.linux-s390x.tar.gz"
else
    echo "Go已安装，版本：$(go version)"
fi

# 创建安装目录
echo "创建安装目录：$INSTALL_DIR"
mkdir -p "$INSTALL_DIR" "$CONFIG_DIR"

# 下载Xray源码
echo "正在下载Xray源码（版本：$XRAY_VERSION）..."
cd /tmp
git clone https://github.com/XTLS/Xray-core.git
cd Xray-core
git checkout "$XRAY_VERSION"

# 编译Xray
echo "正在编译Xray for s390x..."
GOOS=linux GOARCH=s390x go build -o xray ./main
if [ $? -ne 0 ]; then
    echo "编译失败，请检查依赖或网络连接！"
    exit 1
fi

# 移动编译结果到安装目录
mv xray "$INSTALL_DIR/"
cd .. && rm -rf Xray-core

# 生成简单的配置文件
echo "生成Xray配置文件：$CONFIG_DIR/config.json"
cat > "$CONFIG_DIR/config.json" <<EOF
{
  "log": {
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "port": 1080,
      "protocol": "vless",
      "settings": {
        "clients": [
          {
            "id": "your-uuid-here",
            "level": 0
          }
        ],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "tcp"
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "settings": {}
    }
  ]
}
EOF

# 设置权限
chmod +x "$INSTALL_DIR/xray"
chown root:root "$INSTALL_DIR/xray" "$CONFIG_DIR/config.json"

# 创建systemd服务文件
echo "创建Xray服务..."
cat > /etc/systemd/system/xray.service <<EOF
[Unit]
Description=Xray Service
After=network.target

[Service]
Type=simple
ExecStart=$INSTALL_DIR/xray -c $CONFIG_DIR/config.json
Restart=on-failure
User=nobody
Group=nogroup

[Install]
WantedBy=multi-user.target
EOF

# 启动并启用服务
systemctl daemon-reload
systemctl enable xray
systemctl start xray

# 检查服务状态
if systemctl is-active xray > /dev/null; then
    echo "Xray已成功启动！监听端口：1080"
    echo "请编辑 $CONFIG_DIR/config.json 修改UUID和其他配置。"
    echo "运行命令查看状态：systemctl status xray"
else
    echo "Xray启动失败，请检查日志：journalctl -u xray"
fi
