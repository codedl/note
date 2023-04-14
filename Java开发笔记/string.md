trim()：将字符串首尾的空格去掉
```java
    public String trim() {
        int len = value.length;
        int st = 0;
        char[] val = value;    /* avoid getfield opcode */
//找到开头不是字符串的位置，发现是字符是st++停止
        while ((st < len) && (val[st] <= ' ')) {
            st++;
        }
//找到末尾不是字符的位置
        while ((st < len) && (val[len - 1] <= ' ')) {
            len--;
        }
        //截取有效字符(非空字符)
        return ((st > 0) || (len < value.length)) ? substring(st, len) : this;
    }
```