# sardine源码修改

标签（空格分隔）： sardine

---
[TOC]

敏捷分支项目，在使用开源软件`sardine`的过程中，对其源码进行了小部分修改，以满足项目需求。

当时使用的`sardine`版本是5.4.

`maven`库依赖信息如下：

```xml
<dependency>
	<groupId>com.github.lookfirst</groupId>
	<artifactId>sardine</artifactId>
	<version>5.4</version>
</dependency>
```

## 修改代码内容

### 1. 增加`ExistsFolderResponseHandler.java`文件
文件所在的包路径为：
```
com.github.sardine.impl.handler
```
代码如下：
```java
package com.github.sardine.impl.handler;

import org.apache.http.HttpResponse;
import org.apache.http.HttpStatus;
import org.apache.http.StatusLine;

import com.github.sardine.impl.SardineException;

/**
 * 判断目录是否存在
 * @author l00210728
 *
 */
public class ExistsFolderResponseHandler extends
        ValidatingResponseHandler<Boolean>
{
    @Override
    public Boolean handleResponse(HttpResponse response)
            throws SardineException
    {
        StatusLine statusLine = response.getStatusLine();
        int statusCode = statusLine.getStatusCode();
        if (statusCode < HttpStatus.SC_MULTIPLE_CHOICES)
        {
            return true;
        }
        if (statusCode == HttpStatus.SC_NOT_FOUND)
        {
            return false;
        }
        if (statusCode == HttpStatus.SC_FORBIDDEN)
        {
            return true;
        }
        if (statusCode == HttpStatus.SC_METHOD_NOT_ALLOWED)
        {
            return true;
        }
        throw new SardineException("Unexpected response", statusCode,
                statusLine.getReasonPhrase());
    }
}
```

### 2. 增加`PutResponseHandler.java`文件
文件所在的包路径为：
```
com.github.sardine.impl.handler
```
代码为：
```java
package com.github.sardine.impl.handler;

import org.apache.http.Header;
import org.apache.http.HttpResponse;
import org.apache.http.HttpStatus;
import org.apache.http.StatusLine;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.github.sardine.impl.SardineException;

/**
 * 判断是否上传成功
 * @author l00210728
 *
 */
public class PutResponseHandler extends ValidatingResponseHandler<Boolean>
{
    private static final Logger logger = LoggerFactory
            .getLogger(PutResponseHandler.class);

    /**
     * 文件md5
     */
    private String md5;
    
    private int statusCode;

    public String getMd5()
    {
        return md5;
    }

    public int getStatusCode()
    {
        return statusCode;
    }

    @Override
    public Boolean handleResponse(HttpResponse response)
            throws SardineException
    {
        StatusLine statusLine = response.getStatusLine();

        logger.info("-----------------------------------");
        logger.info(statusLine.toString());

        boolean ret = false;

        statusCode = statusLine.getStatusCode();

        if(statusCode == HttpStatus.SC_OK)
        {
            ret = true;
        }

        else if (statusCode == HttpStatus.SC_CREATED)
        {
            //获得md5
            Header[] headers = response.getAllHeaders();
            for(Header h : headers)
            {
                logger.info("  {} : {}", h.getName(), h.getValue());

                if(h.getName().equals("Content-MD5"))
                {
                    md5 = h.getValue();
                }
            }

            ret = true;
        }

        logger.info("-----------------------------------");

        return ret;
    }
}
```

### 3. 修改`SardineImpl.java`文件
文件所在的包路径为：
```
com.github.sardine.impl
```
修改的代码如下：
```
//增加Import
import com.github.sardine.impl.handler.ExistsFolderResponseHandler;
import com.github.sardine.impl.handler.PutResponseHandler;

//由于部分接口在Sardine.java文件中声明，所以需要加上@Override
@Override
public void put(String url, InputStream dataStream, List<Header> headers) throws IOException
{
    //...
}

@Override
public void put(String url, HttpEntity entity, String contentType, boolean expectContinue) throws IOException
{
    //...
}

@Override
public void put(String url, HttpEntity entity, List<Header> headers) throws IOException
{
    //...
}

//此方法有改动
@Override
public <T> T put(String url, HttpEntity entity, List<Header> headers, ResponseHandler<T> handler) throws IOException
{
    /**
     * 由VoidResponseHandler改成PutResponseHandler
     * @author l00210728
     */
    this.put(url, entity, headers, new PutResponseHandler());
}

//此方法有改动
@Override
public boolean exists(String url) throws IOException
{
    /**
     * 增加对文件夹是否存在的判断
     * @author l00210728
     */
    HttpHead head = new HttpHead(url);
    if (url.endsWith("/"))
    {
        /**
         * folder
         */
        return this.execute(head, new ExistsFolderResponseHandler());
    }
    else
    {
        /**
         * file
         */
        return this.execute(head, new ExistsResponseHandler());
    }
}
```
### 4. 修改`SardineUtil.java`文件
文件所在的包路径为：
```
com.github.sardine.util
```
增加`parseDate`方法，代码如下：
```java
/**
 * Loops over all the possible date formats and tries to find the right one.
 *
 * @param value ISO date string
 * @return Null if there is a parsing failure
 */
public static Date parseDate(String value)
{
    if (value == null)
    {
        return null;
    }
    Date date = null;
    for (int i = 0; i < DATETIME_FORMATS.size(); i++)
    {
        ThreadLocal<SimpleDateFormat> format = DATETIME_FORMATS.get(i);
        SimpleDateFormat sdf = format.get();
        if (sdf == null)
        {
            sdf = new SimpleDateFormat(SUPPORTED_DATE_FORMATS[i], Locale.US);
            sdf.setTimeZone(TimeZone.getTimeZone("UTC"));
            format.set(sdf);
        }
        try
        {
            date = sdf.parse(value);
            break;
        }
        catch (ParseException e)
        {
            // We loop through this until we found a valid one.
            ;
        }
    }
    return date;
}
```

### 5. 修改`Sardine.java`文件
文件所在的包路径为：
```
com.github.sardine
```
增加`put`接口，代码如下：
```java
import org.apache.http.Header;
import org.apache.http.HttpEntity;
import org.apache.http.client.ResponseHandler;

/**
 * <一句话功能简述>
 * <功能详细描述>
 * @param url String
 * @param dataStream InputStream
 * @param headers List<Header>
 * @throws IOException IOException
 * @author l00210728
 */
void put(String url, InputStream dataStream, List<Header> headers) throws IOException;

/**
 * <一句话功能简述>
 * <功能详细描述>
 * @param url String
 * @param entity HttpEntity
 * @param contentType String
 * @param expectContinue boolean
 * @throws IOException IOException
 * @author l00210728
 */
void put(String url, HttpEntity entity, String contentType, boolean expectContinue) throws IOException;

/**
 * <一句话功能简述>
 * <功能详细描述>
 * @param url String
 * @param entity HttpEntity
 * @param headers List<Header>
 * @throws IOException IOException
 * @author l00210728
 */
void put(String url, HttpEntity entity, List<Header> headers) throws IOException;

/**
 * <一句话功能简述>
 * <功能详细描述>
 * @param url String
 * @param entity HttpEntity
 * @param headers List<Header>
 * @param handler ResponseHandler<T>
 * @param <T> T
 * @return T <T>
 * @throws IOException IOException
 * @author l00210728
 */
<T> T put(String url, HttpEntity entity, List<Header> headers, ResponseHandler<T> handler) throws IOException;
```

### 6. 修改`Version.java`文件
文件所在的包路径为：
```
com.github.sardine
```
修改内容如下：
```java
//删除main方法

//    public static void main(String[] args)
//    {
//        System.out.println("Version: " + getSpecification());
//        System.out.println("Implementation: " + getImplementation());
//    }
```

