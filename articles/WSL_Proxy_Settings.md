---
title: "WSLの起動時にWindowsのProxy設定をコピーする方法"
emoji: "🐱‍🏍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Windows", "WSL", "Proxy"]
published: false
---

## 課題感

あるProxy環境でWSLを立ち上げるときと、まっさらな状態で立ち上げたいときが複数あり、かつWindowsの設定も同じように変更させなければならなかったです。  
なのでWSLの起動時にWindowsのProxy設定をコピーするようにしました。  
その対象は  

- WSLのProxy設定
- Dockerクライアント側のProxy設定
- Dockerデーモン側のProxy設定

で、起動時に全部Proxy設定を揃えるようにしました。

## やり方

.bashrcに以下のように記述します。  

```bash
# proxy setting
# プロキシ設定はwindows側の設定と同期させる
# WSLの設定とDockerの設定(Dockerクライアント/Dockerデーモン)
function setup_proxy_from_windows() {
  # PowerShellから各設定を個別に取得する
  # 改行と空白が存在する場合があるため、取得時にその2つを削除する
  proxy_enable=$(powershell.exe -command "Get-ItemProperty -Path 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings' | Select-Object -ExpandProperty ProxyEnable" | tr -d '\r\n ')
  proxy_server=$(powershell.exe -command "Get-ItemProperty -Path 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings' | Select-Object -ExpandProperty ProxyServer" | tr -d '\r\n ')
  bypass_list=$(powershell.exe -command "Get-ItemProperty -Path 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings' | Select-Object -ExpandProperty ProxyOverride | ForEach-Object { \$_ -replace '\r?\n', ' ' }" | tr -d '\>
  # Windowsのproxy設定が存在しているか
  if [ "$proxy_enable" -eq 1 ] && [ -n "$proxy_server" ]; then
    # httpが含まれていない場合は追加
    if [[ "$proxy_server" != http://* ]] && [[ "$proxy_server" != https://* ]]; then
      proxy_server="http://$proxy_server"
    fi

    export http_proxy="$proxy_server"
    export https_proxy="$proxy_server"
    export HTTP_PROXY="$proxy_server"
    export HTTPS_PROXY="$proxy_server"

    # セミコロンをカンマに変換(WSLの期待する形)
    bypass_list=${bypass_list//;/,}

    export no_proxy="$bypass_list"
    export NO_PROXY="$bypass_list"

    echo "プロキシ設定を更新しました:"
    echo "http_proxy=$http_proxy"
    echo "https_proxy=$https_proxy"
    echo "no_proxy=$no_proxy"

    # Dockerのプロキシ設定を更新
    setup_docker_proxy

    return 0
  else
    echo "Windows側でプロキシが設定されていないか、直接接続になっています"
    echo "デバッグ: proxy_enable=$proxy_enable (期待値: 1), proxy_server長さ=${#proxy_server} (期待値: >0)"
    unset http_proxy https_proxy no_proxy HTTP_PROXY HTTPS_PROXY NO_PROXY

    # Dockerのプロキシ設定をクリア
    clear_docker_proxy

    return 1
  fi
}

function setup_docker_proxy() {
  # Dockerがインストールされているか確認
  if ! command -v docker &> /dev/null; then
    echo "Dockerがインストールされていないため、Docker用プロキシ設定はスキップします"
    return 1
  fi

  # Dockerデーモン設定ディレクトリの作成
  sudo mkdir -p /etc/systemd/system/docker.service.d

  # プロキシ設定ファイルの作成
  sudo tee /etc/systemd/system/docker.service.d/http-proxy.conf > /dev/null << EOF
[Service]
Environment="HTTP_PROXY=${HTTP_PROXY}"
Environment="HTTPS_PROXY=${HTTPS_PROXY}"
Environment="NO_PROXY=${NO_PROXY}"
EOF

  # Docker個人設定ファイルの作成
  mkdir -p ~/.docker
  cat > ~/.docker/config.json << EOF
{
  "proxies": {
    "default": {
      "httpProxy": "${http_proxy}",
      "httpsProxy": "${https_proxy}",
      "noProxy": "${no_proxy}"
    }
  }
}
EOF

  # Dockerサービスの再起動（systemctlが利用可能な場合）
  if command -v systemctl &> /dev/null; then
    sudo systemctl daemon-reload
    sudo systemctl restart docker
    echo "Dockerプロキシ設定を更新し、サービスを再起動しました"
  else
    # systemctlが使えない場合（WSL1など）の代替手段
    if command -v service &> /dev/null; then
      sudo service docker restart
      echo "Dockerプロキシ設定を更新し、サービスを再起動しました"
    else
      echo "Dockerプロキシ設定を更新しました。Dockerサービスの手動再起動が必要かもしれません"
    fi
  fi
}

function clear_docker_proxy() {
  # Dockerがインストールされているか確認
  if ! command -v docker &> /dev/null; then
    return 1
  fi

  # プロキシ設定ファイルを削除
  if [ -f /etc/systemd/system/docker.service.d/http-proxy.conf ]; then
    sudo rm /etc/systemd/system/docker.service.d/http-proxy.conf
  fi

  # 個人設定ファイルを更新（プロキシ設定を削除）
  if [ -f ~/.docker/config.json ]; then
    # 一時的な方法として新しい空の設定を作成
    cat > ~/.docker/config.json << EOF
{
  "proxies": {
    "default": {}
  }
}
EOF
  fi

  # Dockerサービスの再起動
  if command -v systemctl &> /dev/null; then
    sudo systemctl daemon-reload
    sudo systemctl restart docker
    echo "Dockerプロキシ設定をクリアし、サービスを再起動しました"
  else
    if command -v service &> /dev/null; then
      sudo service docker restart
      echo "Dockerプロキシ設定をクリアし、サービスを再起動しました"
    else
      echo "Dockerプロキシ設定をクリアしました。Dockerサービスの手動再起動が必要かもしれません"
    fi
  fi
}

# WSL起動時にプロキシ設定を実行
setup_proxy_from_windows
```

## まとめと反省点

`sudo`が必要なので起動時にパスワード聞かれるのがちょっと面倒くさいですが、いちいち設定を変えるより楽になりました。
