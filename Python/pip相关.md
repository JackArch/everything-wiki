## 清华大学镜像源
pypi 镜像每 5 分钟同步一次。

临时使用
```bash
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple some-package
```

注意，simple 不能少, 是 https 而不是 http

设为默认
修改 ``~/.config/pip/pip.conf` (Linux), ``%APPDATA%\pip\pip.ini` (Windows 10) 或 ``$HOME/Library/Application Support/pip/pip.conf` (macOS) (没有就创建一个)， 修改 index-url至tuna，例如
```
[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
```
pip 和 pip3 并存时，只需修改 ~/.pip/pip.conf。
参考: https://mirrors.tuna.tsinghua.edu.cn/help/pypi/


## 公司内代理
开发机
```bash
pip3 install package_name --proxy dev-proxy.oa.com:8080
```
