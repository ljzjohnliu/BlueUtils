# BlueUtils
经典蓝牙搜索，连接，数据传输小DEMO

通过经典模式 搜索 蓝牙应用。蓝牙有蓝牙1.0、蓝牙2.0、蓝牙3.0、蓝牙4.0之类的以数字结尾的蓝牙版本号，而实际上，在最新
的标准中，已经不再使用数字版本号作为蓝牙版本的区分了，取而代之的是经典蓝牙与低功耗蓝牙（BLE）这两种区别。BLE 蓝牙不
做过多讲解。具体的信息大家可以参考。
http://www.loverobots.cn/the-analysis-is-simple-compared-with-the-classic-bluetooth-and-bluetooth-low-energy-in-android.html

# 流程
  发现设备->配对/绑定设备->建立连接->数据通信
  经典蓝牙和低功耗蓝牙除了配对/绑定这个环节是一样的之外，其它三个环节都是不同的。
  
# 截图
![image](https://github.com/moruoyiming/BlueUtils/blob/master/pics/Screenshot_2017-12-29-15-58-50-172_com.calypso.bu.png) 
![image](https://github.com/moruoyiming/BlueUtils/blob/master/pics/Screenshot_2017-12-29-15-59-22-173_com.calypso.bu.png) 
![image](https://github.com/moruoyiming/BlueUtils/blob/master/pics/Screenshot_2017-12-29-15-59-33-044_com.calypso.bu.png) 
![image](https://github.com/moruoyiming/BlueUtils/blob/master/pics/Screenshot_2017-12-29-15-59-44-944_com.calypso.bu.png) 
  
  
# 详解
  公司最近在要做一个蓝牙与串口通讯的项目，然后就涉及到手机端与蓝牙的连接及数据交互。大致需求就是通过手机搜索硬件蓝牙
  设备，然后连接上蓝牙，通过手机端的指令消息来获取串口信息，在通过蓝牙返回数据到手机端。在这之前看了一些开源的项目，
  包括BluetoothKit，FastBle，BluetoothHelper等其中BluetoothKit和FastBle只支持BLE 模式蓝牙，因为硬件的模式是
  经典模式，后来自己在两个项目的基础上做了一些修改，然后可以搜索到经典蓝牙。但是怎么也是连接不上我们的硬件设备。（应
  该是底层不是经典蓝牙连接导致。）后来发现了BluetoothHelper项目。在这个项目的基础上做了一些修改及优化 ，能够满足
  项目需求，现在将这个项目做了分包及优化。然后在这分享自己的一些踩坑心得。



  在页面首先初始化一个BlueManager

  private BlueManager bluemanage;

  bluemanage = BlueManager.from(MainActivity.this);

  然后通过 调用 searchDevices 获取蓝牙设备，有些手机搜索开始之后 一直不走onSearchCompleted。

    bluemanage.searchDevices(new OnSearchDeviceListener() {
                    @Override
                    public void onStartDiscovery() {
                        Log.d(TAG, "onStartDiscovery()");
                    }

                    @Override
                    public void onNewDeviceFound(BluetoothDevice device) {
                        Log.d(TAG, "new device: " + device.getName() + " " + device.getAddress());
                    }

                    @Override
                    public void onSearchCompleted(List<BluetoothDevice> bondedList, List<BluetoothDevice> newList) {
                        Log.d(TAG, "SearchCompleted: bondedList" + bondedList.toString());
                        Log.d(TAG, "SearchCompleted: newList" + newList.toString());
                    }

                    @Override
                    public void onError(Exception e) {
                        e.printStackTrace();
                    }
                });

   通过 BlueManager里的searchDevices方法，里边其实就是获取了一个BluetoothAdapter然后，通过调用mBluetoothAda
   pter.startDiscovery();来搜索经典蓝牙设备。这里如果调用 mBluetoothAdapter.startLeScan(mLeScanCallback);
   搜索的就是BLE蓝牙。然后在这之前需要动态注册一个BroadcastReceiver来监听 蓝牙的搜索情况，在通过onReceive中去判
   断设备的类型，是不是新设备，是不是已经链接过。搜索完成同样也会被监听到。

   搜索代码如下

      public void searchDevices(OnSearchDeviceListener listener) {

            checkNotNull(listener);
            if (mBondedList == null) mBondedList = new ArrayList<>();
            if (mNewList == null) mNewList = new ArrayList<>();

            mOnSearchDeviceListener = listener;

            if (mBluetoothAdapter == null) {
                mOnSearchDeviceListener.onError(new NullPointerException(DEVICE_HAS_NOT_BLUETOOTH_MODULE));
                return;
            }

            if (mReceiver == null) mReceiver = new Receiver();//注册receiver监听回调

            // ACTION_FOUND
            IntentFilter filter = new IntentFilter(BluetoothDevice.ACTION_FOUND);
            mContext.registerReceiver(mReceiver, filter);

            // ACTION_DISCOVERY_FINISHED
            filter = new IntentFilter(BluetoothAdapter.ACTION_DISCOVERY_FINISHED);
            mContext.registerReceiver(mReceiver, filter);

            mNeed2unRegister = true;

            mBondedList.clear();
            mNewList.clear();

            if (mBluetoothAdapter.isDiscovering())    //先判断是否在扫描
                mBluetoothAdapter.cancelDiscovery();  //取消扫描
            mBluetoothAdapter.startDiscovery();       //开始扫描蓝牙

            if (mOnSearchDeviceListener != null)
                mOnSearchDeviceListener.onStartDiscovery();

        }

   到这里搜索的大概流程就是走完了。接下来说下配对连接。

      bluemanage.connectDevice("00:21:13:02:9B:F1", new OnConnectListener() {
                         @Override
                         public void onConnectStart() {
                             Log.i("blue", "onConnectStart");
                         }

                         @Override
                         public void onConnectting() {
                             Log.i("blue", "onConnectting");
                         }

                         @Override
                         public void onConnectFailed() {
                             Log.i("blue", "onConnectFailed");
                         }

                         @Override
                         public void onConectSuccess() {
                             Log.i("blue", "onConectSuccess");
                         }

                         @Override
                         public void onError(Exception e) {
                             Log.i("blue", "onError");
                         }
                     });

    就是开启一个线程去连接远程蓝牙


        public void connectDevice(String mac, OnConnectListener listener) {
            if (mCurrStatus != STATUS.CONNECTED) {
                if (mac == null || TextUtils.isEmpty(mac))
                    throw new IllegalArgumentException("mac address is null or empty!");
                if (!BluetoothAdapter.checkBluetoothAddress(mac))
                    throw new IllegalArgumentException("mac address is not correct! make sure it's upper case!");
                ConnectDeviceRunnable connectDeviceRunnable = new ConnectDeviceRunnable(mac, listener);
                checkNotNull(mExecutorService);
                mExecutorService.submit(connectDeviceRunnable);
            } else {
                Log.i("blue", "the blue is connected !");
            }
        }

   在连接的线程run方法中，通过调用mBluetoothAdapter.getRemoteDevice 获取远程蓝牙信息，通过createInsecureRfcommSocketToServiceRecord
   获得一个与远程蓝牙的socket连接。通过这个进行连接及数据的读写。

         BluetoothDevice remoteDevice = mBluetoothAdapter.getRemoteDevice(mac);
                   mBluetoothAdapter.cancelDiscovery();
                   mCurrStatus = STATUS.FREE;
                   try {
                       Log.d(TAG, "prepare to connect: " + remoteDevice.getAddress() + " " + remoteDevice.getName());
                       mSocket = remoteDevice.createInsecureRfcommSocketToServiceRecord(UUID.fromString(Constants.STR_UUID));
                       listener.onConnectting();
                       mSocket.connect();
                       mInputStream = mSocket.getInputStream();
                       mOutputStream = mSocket.getOutputStream();
                       mCurrStatus = STATUS.CONNECTED;
                       if (listener != null) {
                           listener.onConectSuccess();
                       }
                   } catch (Exception e) {
                       e.printStackTrace();
                       if (listener != null)
                           listener.onConnectFailed();
                       try {
                           mInputStream.close();
                           mOutputStream.close();
                       } catch (IOException closeException) {
                           closeException.printStackTrace();
                       }
                       mCurrStatus = STATUS.FREE;
                   }

   当设备连接成功之后，就可以给蓝牙设备发送消息了。 通过调用bluemanage.sendMessage(MessageBean mesaage，
   needResponse，OnSendMessageListener listener，OnReceiveMessageListener listener);在bluemange里会开
   起一个WriteRunnable写线程和一个ReadRunnable，读线程只会在第一次发消息时初始化一次。以后都是用这个线程
   去读从蓝牙返回的数据。

        写数据
          writer.write(item.text);
          writer.newLine();
          writer.flush();

   在WriteRunnable 的run方法中通过mOutputStream流将数据传送给蓝牙设备,当蓝牙接受到消息也就会返回数据，
   在ReadRunnable中从mInputStream里不断的读取数据。这里有一个问题，就是有的时候从蓝牙口都的数据并不是一
   个完整的数据，这里就是一个坑。首先你需要知道你需要什么数据，什么格式，数据的长度。这里我们的数据的格式
   类似是一帧一帧，而且我们的帧长度固定大小是10. 那么我们就可以在这里做一些你想做的事了。

 # 坑 有时候从蓝牙socket 中读取的数据不完整
    读数据不完整，是因为我们开启线程之后会一直读，有时候蓝牙并没有返回数据，或者没有返回完整数据，这个时候
    我们需要在这做一些特殊处理。

            int count = 0;
            while (count == 0) {
                count = stream.available();//输入流中的数据个数。
            }

    通过以上代码可以确保读的数据不会是0。通过下边的代码可以确保读到完整数据之后才会走我的回调，保证了数据
    的完整性。这里的what只是我用来区分当前读到的数据是进度信息，还是真正想要的信息。

             if (count == 10 && what) {
                  int num = stream.read(buffer);
                  String progress = TypeConversion.bytesToHexStrings(buffer);
                  Log.i("progress", progress);
                  if (mListener != null) {
                      mListener.onProgressUpdate(progress, 0);
                  }
              } else if (count >= 10) {
                  what = false;
                  int num = stream.read(buffer);
                  String detect = TypeConversion.bytesToHexStrings(buffer);
                  builder.append(detect);
                  Log.i("detect", detect);
                  if (detect.endsWith("04 ")) {
                      number++;//这个number也是一个标记，用来标记当前读了多少信息，当读完所有的信息就
                               //回调接口。通知界面信息读取完整。
                  }
                  if (number == 5) {
                      if (mListener != null) {
                          mListener.onDetectDataFinish();
                          mListener.onNewLine(builder.toString().trim());
                          builder.delete(0, builder.length());
                      }
                  } else {
                      if (mListener != null) {
                          mListener.onDetectDataUpdate(detect);
                      }
                  }
              }


    下边是BlueManager提供的一些方法：

     requestEnableBt()    开启蓝牙

     searchDevices()      搜索蓝牙设备

     connectDevice()      连接蓝牙设备

     closeDevice()        断开蓝牙连接

     sendMessage()        发送消息

     close()              关闭销毁蓝牙

     BlueManager大概的使用流程及大致原理就说到这里，口才不是很好，平常也不怎么写博客，有什么问题大家
     探讨一下。项目代码部分参考BluetoothHelper 项目，在此基础上做了一些分包优化。如有雷同，不属巧合，
     我就是抄的你的。哈哈哈哈~~  希望对那些在踩蓝牙坑的小伙伴有帮助~~~

     #Contact Me
     QQ: 798774875
     Email: moruoyiming123@gmail.com
     GitHub: https://github.com/moruoyiming