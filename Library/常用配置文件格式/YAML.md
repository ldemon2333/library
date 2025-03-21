它的基本语法规则如下。
- 大小写敏感
- 使用缩进表示层级关系
- 缩进时不允许使用 Tab 键，只允许使用空格
- 缩进的空格数目不重要，只要相同层级的元素左侧对齐即可

`#` 表示注释，从这个字符一直到行尾，都会被解析器忽略。

YAML 支持的数据结构有三种。
- 对象：键值对的集合，又称为映射（mapping）/ 哈希（hashes） / 字典（dictionary）
- 数组：一组按次序排列的值，又称为序列（sequence） / 列表（list）
- 纯量（scalars）：单个的、不可再分的值

# 对象
对象的一组键值对，使用冒号结构表示。

```yaml
animal: pets
```

Yaml 也允许另一种写法，将所有键值对写成一个行内对象。
```yaml
hash: {name: Steve, foo: bar}
```


# 数组
一组连词线开头的行，构成一个数组
```yaml
- Cat
- Dog
- Goldfish
```
转为 JavaScript 如下。
```javascript
[ 'Cat', 'Dog', 'Goldfish' ]
```
数据结构的子成员是一个数组，则可以在该项下面缩进一个空格。
```yaml
-
 - Cat
 - Dog
 - Goldfish
```
转为 JavaScript 如下。
```javascript
[ [ 'Cat', 'Dog', 'Goldfish' ] ]
```
数组也可以采用行内表示法。
```yaml
animal: [Cat, Dog]
```

# 复合结构
对象和数组可以结合使用，形成复合结构。
```yaml
languages:
 - Ruby
 - Perl
 - Python 
websites:
 YAML: yaml.org 
 Ruby: ruby-lang.org 
 Python: python.org 
 Perl: use.perl.org 
```
转为 JavaScript 如下。
```javascript
{ languages: [ 'Ruby', 'Perl', 'Python' ],
  websites: 
   { YAML: 'yaml.org',
     Ruby: 'ruby-lang.org',
     Python: 'python.org',
     Perl: 'use.perl.org' } }
```

# 纯量
`null`用`~`表示。

```yaml
parent: ~
```

YAML 允许使用两个感叹号，强制转换数据类型。
```yaml
e: !!str 123
f: !!str true
```





