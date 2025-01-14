## 隐藏后门

### attrib

显示、设置或删除分配给文件或目录的属性

+ `{+\|-}s `：设置 (+) 或清除 (-) 系统文件属性。 如果文件使用此属性集，则必须先清除 该属性，然后才能更改该文件的任何其他属性。

+ `{+\|-}h `：设置 (+) 或清除 (-) 隐藏文件属性。 如果文件使用此属性集，则必须先清除 该属性，然后才能更改该文件的任何其他属性。

```cmd
attrib +h +s "<path to executable>"
```
