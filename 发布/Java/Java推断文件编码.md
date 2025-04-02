---
title: Java推断文件编码
slug: java-file-encoding-inference
categories:
  - Lang
tags:
  - Java
featuredImage: https://sonder.vitah.me/ryze/71b535771df3936045f62df02ed50c76.webp
desc: 本文介绍了如何使用com.ibm.icu库来推断文件的编码
draft: false
---

使用的maven库：
```xml
<dependency>
    <groupId>com.ibm.icu</groupId>
    <artifactId>icu4j</artifactId>
    <version>77.1</version>
</dependency>
```

示例代码：
```Java
import com.ibm.icu.text.CharsetDetector;  
import com.ibm.icu.text.CharsetMatch;
import java.io.*;

public static String detectFileCharset(InputStream inputStream) {  
    try {  
        BufferedInputStream bufferedInputStream = new BufferedInputStream(inputStream);  
        CharsetDetector detector = new CharsetDetector();  
        detector.setText(bufferedInputStream);  
        CharsetMatch match = detector.detect();  
        return match.getName();  
    } catch (IOException e) {  
        e.printStackTrace();  
        return "";  
    }
}
```