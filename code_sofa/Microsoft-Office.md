# Microsoft-Office

<br />

## 2013 Version

<br />

### 激活步骤

```text
下载 Microsoft-Office 2013, 在下载激活软件 (F:\Installation Package\MicrosoftOffice)

检查激活:
    cmd -> cscript "F:\Program Files\Microsoft Office\Office15\ospp.vbs" /dstatus
    复制 SKU ID, 再执行命令: slmgr /xpr ***(复制的 SKU ID)
    会弹窗提示激活到期时间, 或者永久激活
```