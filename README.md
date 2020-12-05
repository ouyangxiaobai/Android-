# Android-
Android应用源码手机实时视频监控项目
<img src="https://img-blog.csdnimg.cn/20201205234836531.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMxMjkzNTc1,size_16,color_FFFFFF,t_70">
<img src="https://img-blog.csdnimg.cn/20201205234836569.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMxMjkzNTc1,size_16,color_FFFFFF,t_70">
首先，简单介绍一下原理。主要是在手机客户端（Android）通过实现Camera.PreviewCallback接口，在其onPreviewFrame重载函数里面获取摄像头当前图像数据，然后通过Socket将图像数据和相关的用户名、命令等数据传输到服务器程序中。服务器端（PC端）采用C#编写，通过监听相应的端口，在获取数据后进行相应的命令解析和图像数据还原，然后将图像数据传递至PictureBox控件中用于显示，这样就实现了手机摄像头的视频数据实时传输到服务器上。如果需要将这些视频进行转发，通过服务器再将这些数据复制转发即可。效果如下：



对于Android客户端上主要有几个地方需要注意，第一个就是Socket通信。Socket通信可以通过Socket类来实现，直接结合PrintWriter来写入命令，如下定义的一个专门用于发送命令的线程类，当要连接到服务器和与服务器断开时，都需要发送命令通知服务器，此外在进行其他文字传输时也可以采用该方法，具体代码如下：

    /**发送命令线程*/

    class MySendCommondThread extends Thread{

       private String commond;

       public MySendCommondThread(String commond){

           this.commond=commond;

       }

       public void run(){

           //实例化Socket 

            try {

              Socket socket=new Socket(serverUrl,serverPort);

              PrintWriter out = new PrintWriter(socket.getOutputStream());

              out.println(commond);

              out.flush();

           } catch (UnknownHostException e) {

           } catch (IOException e) {

           } 

       }

    }

如果是采用Socket发送文件，则可以通过OutputStream将ByteArrayInputStream数据流读入，而文件数据流则转换为ByteArrayOutputStream。如果需要在前面添加文字，同样也需要转换为byte，然后写入OutputStream。同样也可以通过定义一个线程类发送文件，如下：

   /**发送文件线程*/

    class MySendFileThread extends Thread{ 

       private String username;

       private String ipname;

       private int port;

       private byte byteBuffer[] = new byte[1024];

       private OutputStream outsocket;  

       private ByteArrayOutputStream myoutputstream;

      

       public MySendFileThread(ByteArrayOutputStream myoutputstream,String username,String ipname,int port){

           this.myoutputstream = myoutputstream;

           this.username=username;

           this.ipname = ipname;

           this.port=port;

            try {

              myoutputstream.close();

           } catch (IOException e) {

              e.printStackTrace();

           }

       }

      

        public void run() {

            try{

              //将图像数据通过Socket发送出去

                Socket tempSocket = new Socket(ipname, port);

                outsocket = tempSocket.getOutputStream();

                //写入头部数据信息

              String msg=java.net.URLEncoder.encode("PHONEVIDEO|"+username+"|","utf-8");

                byte[] buffer= msg.getBytes();

                outsocket.write(buffer);

               

                ByteArrayInputStream inputstream = new ByteArrayInputStream(myoutputstream.toByteArray());

                int amount;

                while ((amount = inputstream.read(byteBuffer)) != -1) {

                    outsocket.write(byteBuffer, 0, amount);

                }

                myoutputstream.flush();

                myoutputstream.close();

                tempSocket.close();                  

            } catch (IOException e) {

                e.printStackTrace();

            }

        }

    }

而获取摄像头当前图像的关键在于onPreviewFrame()重载函数里面，该函数里面有两个参数，第一个参数为byte[]，为摄像头当前图像数据，通过YuvImage可以将该数据转换为图片文件，同时还可用对该图片进行压缩和裁剪，将图片进行压缩转换后转换为    ByteArrayOutputStream数据，即前面发送文件线程类中所需的文件数据，然后采用线程发送文件，如下代码：

    @Override

    public void onPreviewFrame(byte[] data, Camera camera) {

       // TODO Auto-generated method stub

       //如果没有指令传输视频，就先不传

       if(!startSendVideo)

           return;

       if(tempPreRate<VideoPreRate){

           tempPreRate++;

           return;

       }

       tempPreRate=0;      

       try {

             if(data!=null)

             {

               YuvImage image = new YuvImage(data,VideoFormatIndex, VideoWidth, VideoHeight,null);

               if(image!=null)

               {

                  ByteArrayOutputStream outstream = new ByteArrayOutputStream();

                 //在此设置图片的尺寸和质量

                 image.compressToJpeg(new Rect(0, 0, (int)(VideoWidthRatio*VideoWidth),

                    (int)(VideoHeightRatio*VideoHeight)), VideoQuality, outstream); 

                 outstream.flush();

                 //启用线程将图像数据发送出去

                 Thread th = new MySendFileThread(outstream,pUsername,serverUrl,serverPort);

                 th.start(); 

               }

             }

         } catch (IOException e) {

             e.printStackTrace();

         }

    }

值得注意的是，在调试中YuvImage可能找不到，在模拟机上无法执行该过程，但是编译后在真机中可以通过。此外，以上传输文字字符都是采用UTF编码，在服务器端接收时进行解析时需要采用对应的编码进行解析，否则可能会出现错误解析。

Android客户端中关键的部分主要就这些，新建一个Android项目（项目名称为SocketCamera），在main布局中添加一个SurfaceView和两个按钮，如下图所示：



然后在SocketCameraActivity.java中添加代码，具体如下：

package com.xzy;

 

import java.io.ByteArrayInputStream;

import java.io.ByteArrayOutputStream;

import java.io.IOException;

import java.io.OutputStream;

import java.io.PrintWriter;

import java.net.Socket;

import java.net.UnknownHostException;

import android.app.Activity;

import android.app.AlertDialog;

import android.content.DialogInterface;

import android.content.Intent;

import android.content.SharedPreferences;

import android.graphics.Rect;

import android.graphics.YuvImage;

import android.hardware.Camera;

import android.hardware.Camera.Size;

import android.os.Bundle;

import android.preference.PreferenceManager;

import android.view.Menu;

import android.view.MenuItem;

import android.view.SurfaceHolder;

import android.view.SurfaceView;

import android.view.View;

import android.view.WindowManager;

import android.view.View.OnClickListener;

import android.widget.Button;

 

public class SocketCameraActivity extends Activity implements SurfaceHolder.Callback,

Camera.PreviewCallback{    

    private SurfaceView mSurfaceview = null; // SurfaceView对象：(视图组件)视频显示

    private SurfaceHolder mSurfaceHolder = null; // SurfaceHolder对象：(抽象接口)SurfaceView支持类

    private Camera mCamera = null; // Camera对象，相机预览

   

    /**服务器地址*/

    private String pUsername="XZY";

    /**服务器地址*/

    private String serverUrl="192.168.1.100";

    /**服务器端口*/

    private int serverPort=8888;

    /**视频刷新间隔*/

    private int VideoPreRate=1;

    /**当前视频序号*/

    private int tempPreRate=0;

    /**视频质量*/

    private int VideoQuality=85;

   

    /**发送视频宽度比例*/

    private float VideoWidthRatio=1;

    /**发送视频高度比例*/

    private float VideoHeightRatio=1;

   

    /**发送视频宽度*/

    private int VideoWidth=320;

    /**发送视频高度*/

    private int VideoHeight=240;

    /**视频格式索引*/

    private int VideoFormatIndex=0;

    /**是否发送视频*/

    private boolean startSendVideo=false;

    /**是否连接主机*/

    private boolean connectedServer=false;

   

    private Button myBtn01, myBtn02;

    /** Called when the activity is first created. */

    @Override

    public void onCreate(Bundle savedInstanceState) {

        super.onCreate(savedInstanceState);

        setContentView(R.layout.main);
