
---
tags: Snippet
---


```java
public static String detectFileType(InputStream inputStream) {  
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