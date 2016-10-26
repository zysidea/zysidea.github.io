---
title: 区分EditText是代码赋值还是手动输入赋值
tags:
  - EditText
  - TextWatcher
categories:
  - Android
id: 88
date: 2016-01-06 22:07:25
---

由于某些需求，需要区分**EditText**是代码赋值**editText.setText()**还是手动输入，一开始想到的是**TextWatcher**

```java
//监听 输入改变
    TextWatcher textrListener = new TextWatcher() {
        @Override
        public void beforeTextChanged(CharSequence s, int start, int count, int after) {
        }

        @Override
        public void onTextChanged(CharSequence s, int start, int before, int count) {
           //在此修改 判断用的 boolean值
        }

        @Override
        public void afterTextChanged(Editable s) {
        }
    };
```

但是，这样很有欠缺，比如用于手动输入在代码赋值删掉再输入。。。。。，很快这个办法就不适用了！
对于这样的很头疼的的需求，想了半天最终想到了下面的办法

### 定义一个全局变量，用于判断

```java
private boolean isTextSetProgrammatically; //用于判断是否是通过 setText输入的

```
### 自定义一个赋值方法

```java
private void setTextProgrammatically(String text) {
        mEditText.removeTextChangedListener(textListener);
        mEditText.setText(text);
        isTextSetProgrammatically = true;
        mEditText.addTextChangedListener(textListener);
    }

```

### EditText输入监听事件

```java
//监听 输入改变
    TextWatcher textListener = new TextWatcher() {
        @Override
        public void beforeTextChanged(CharSequence s, int start, int count, int after) {
        }

        @Override
        public void onTextChanged(CharSequence s, int start, int before, int count) {
            isTextSetProgrammatically = false;
        }

        @Override
        public void afterTextChanged(Editable s) {
        }
    };

```

在需求用到的地方的 所有的 代码赋值都调用而不是这样通过*isTextSetProgrammatically **就可以区分了


