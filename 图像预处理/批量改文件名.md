## 在windows中的shellpower中运行
## 先进入指定目录
```
$i = 1
Get-ChildItem '.\*.jpg' -File | ForEach-Object {
    Rename-Item -LiteralPath $_.FullName -NewName ("{0}.jpg" -f $i++)
}

$i = 1
Get-ChildItem '.\*.png' -File | ForEach-Object {
    Rename-Item -LiteralPath $_.FullName -NewName ("{0}.png" -f $i++)
}

```
