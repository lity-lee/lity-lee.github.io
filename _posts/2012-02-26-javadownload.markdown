---
layout: post
category: "program"
title:  "java多线程下载"
tags: [编程,java,http]
---

### java多线程下载http协议的资源

1. 文件输出流FileOutStream，可能是之前学习数据库或者学习c语言对文件操作的原因，一直觉得一个文件只能被一个对象或指针作写操作。而java语言上针对同一个文件可以有多个输出流，每个输出流相对其它输出流是相对独立的，即每个输出流对文件操作后都记录当前的位置，为此对象下次对文件操作提供依据。当然，文件最终的内容，指某段数据，一定是有输出流中最后对此段数据写操作的输出流写入的内容。

1. 此小程序设计的思路是，在main线程中请求http资源，获取出来长度，然后用RandomAccessFile对象创建本地文件，包括文件的大小，这时一个空的文件就创建了出来。由获取出来的总长度平均分成5份，最后的一份要把余下来的加上。这5份分别由5个线程负责请求并下载保存。http协议可以只请求文件的一部分。

1. 尽管http资源在网上很多，很容易找到，但找到无效的资源链接更容易，一个无效的链接用写好的程序测试时会超时。这时，如果你想用迅雷下载一下来测证此资源是否有效，很多情况下是能下载的，这是因为迅雷使用了替代资源，可以在使用迅雷下载时勾选“只从原始地址下载”来验证资源确实有效。

1. 用到了多线程问题，不得不说线程对象Thread是不能够复用的，就是当一个线程对象调用了start()方法后，无论是否有线程已启动，都不能再用这个线程对象启动新的线程，而线程对象也没有类似reset()这样的方法。

1. 下载完成后，怎样确保保存的这个文件就是网上那个资源文件呢。用迅雷或其实下载工具下载此资源，然后对这两个文件比对当，当然你可以自己写程序来比对。有一简便的方法是下载一个md5算法工具，分别计算出这两个文件的md5值进行比对（尽管有人证明出来md5算法是有碰撞的，但目前为止可以忽略）。

```
package com.lity.java.apis.net.download;
import java.io.BufferedInputStream;
import java.io.RandomAccessFile;
import java.net.HttpURLConnection;
import java.net.URL;
public class HttpMultiThreadDownload {
  public static final int THREAD_COUNT = 5;

  public static final int REFRESH_SPEED_TIME = 1 * 1000;
  /*
  * 以下是测试资源：
  * http://dlc2.sdo.com/FTP/AION/20120216/2/data136.zip
  * 如果上面的连接失效,可以找游戏资源的下载
  *
  */
  public static void main(String[] args) throws Exception {
    final String sourceFile = "http://dlc2.sdo.com/FTP/AION/20120216/1/AION_Setup2.1.0.22.exe";
    URL url = new URL(sourceFile);
    HttpURLConnection conn = (HttpURLConnection)url.openConnection();

    // 添加这句,可能获取不到长度
    //  conn.setInstanceFollowRedirects(false);
    // 文件长度
    long fileLength = conn.getContentLength();
    System.out.println("fileLength:" + fileLength);

    // 每个线程负责的长度
    long threadLength = fileLength / THREAD_COUNT;


    // 最后一个线程多负的一部分,可能为0
    long retain = fileLength % THREAD_COUNT;

    // 线程1
    long start = 0;
    long length = threadLength;

    // 创建本地临时文件
    String savePath = "down/AION_Setup2.1.0.22.exe";
    RandomAccessFile raf = new RandomAccessFile(savePath, "rw");
    raf.seek(fileLength);
    raf.close();

    // 创建线程对象
    SubThreadDownload[] threads = new SubThreadDownload[THREAD_COUNT];
    for (int i = 0; i < THREAD_COUNT; i++) {
      start = i * threadLength;
      length = start + threadLength;
      if (THREAD_COUNT - 1 == i) {
        length += retain;
      }
      SubThreadDownload thread = new SubThreadDownload(sourceFile, start, length, savePath, "thread" + i);
      threads[i] = thread;
    }

    // 开始下载
    System.out.println("开始下载");
    for (int i = 0; i < THREAD_COUNT; i++) {
      threads[i].start();
    }
  }
}
class SubThreadDownload extends Thread {

  /*
  * 每个线程负责请求并保存下载文件的一部分
  * 由于java上,多个文件输入流可以同时写入一个文件
  * 并且主线程负责创建一个临时文件,子线程每个负责读取、存储这个文件的部分流
  * 所以它们不会出现,不同步的情况。
  */

  /** 线程负责下载部分的起点 **/
  private long start;

  /** 线程负责下载部分的长度 **/
  private long length;

  /** 下载后保存的文件 **/
  private String saveFile;

  /** 下载的源文件 **/
  private String sourceFile;
  public SubThreadDownload(String sourceFile, long start, long length, String saveFile, String threadName) {
    super(threadName);
    this.start = start;
    this.length = length;
    this.saveFile = saveFile;
    this.sourceFile = sourceFile;
    System.out.println(getName() + ", start:" + start + ", end:" + length);
  }

  @Override
  public void run() {
    super.run();
    try {
      URL url = new URL(sourceFile);
      HttpURLConnection conn = (HttpURLConnection)url.openConnection();
      conn.addRequestProperty("Range", "bytes=" + start + "-" + length);
      BufferedInputStream bis = new BufferedInputStream(conn.getInputStream());
      byte[] buffer = new byte[2048];
      int length = -1;
      RandomAccessFile raf = new RandomAccessFile(saveFile, "rw");
      raf.seek(start);
      long startTime = System.currentTimeMillis();
      long tempTime = 0;
      long tempTotal = 0;
      while (-1 != (length = bis.read(buffer, 0, buffer.length))) {
        raf.write(buffer, 0, length);
        tempTime += System.currentTimeMillis() - startTime;
        tempTotal += length;
        if (tempTime > 1 * 1000) {
          System.out.println(getName() + ", current speed:" + tempTotal / 1024 / (tempTime / 1000) + "kb/s");
          tempTotal = 0;
          tempTime = 0;
        }

        // 最后记录开始时间
        startTime = System.currentTimeMillis();
      }
      System.out.println(getName() + " is over");
    } catch (Exception e) {
      System.err.println(getName() + " has an error:" + e);
    }
  }
}
```
