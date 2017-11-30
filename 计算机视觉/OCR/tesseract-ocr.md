
从`https://github.com/UB-Mannheim/tesseract/wiki`下载windows安装包。安装时勾选中文(简体和繁体)语言包。

设置环境变量，将安装路径加入`PATH`变量中。
设置`TESSDATA_PREFIX`环境变量为tessdata所在目录。

测试:
```bash
tesseract sample.jpg -l chi_sim result
```
结果:
```
读 书 不 是 为 了 拿 文 凭 或 者
发 财 , 而 是 成 为 一 个 有 温
度 懂 情 趣 会 思 考 的 人 。

```
