## YAML 语法

### 1、基本语法

k:(空格)v：表示一对键值对，空格不能舍去。

以缩进控制层级关系，属性和值是大小写敏感的。

### 2、值的写法

#### 字面量：普通的值（基本数据类型）：

k: v，字面直接写；

注意：字符串默认不用加上单引号或者双引号。

+ ""：双引号不会转义字符串里面的特殊字符，比如

  ```yaml
  name: "zpffly\nscnu"
  输出
  zpffly
  scnu
  ```

  

+ ''：单引号会转义特殊字符，如果字符串包含空格也要使用单引号。

  ```yaml
  name: '  zpffly\nscnu'
  输出
    zpffly\nscnu
  ```

#### 对象（属性和值）、Map（键值对）：

普通写法：

```yaml
person:
    name: zpffly
    age: 20
```

行内写法：

```yaml
person: {name: zpffly,age: 20}
```



#### 数组（List、Set）：

用 `- 值`表示元素（- 和值中间有空格）

```yaml
pets:
    - cat
    - dog
    - pig
```

行内写法

```yaml
pets: [cat, dog, pig]
```

