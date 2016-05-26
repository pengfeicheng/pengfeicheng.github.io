---
layout: post
title:  "Android的BLE通信实现" 
author: 程鹏飞
tags: Android BLE 通信
category: Android
---

####1、关于Android平台的BLE

蓝牙4.0于2010年发布，相对于上个版本3.0，它的特点是更省电、成本低
延迟低等特点，现在最新的蓝牙协议是2013年底发布的蓝牙4.1，蓝牙4.1在4.0
基础上进行升级，使得可穿戴设备的批量数据传输速度更高。Android是从4.3
才开始提供BLE API,这也就限定了BLE的应用只能运行在Android 4.3及其以上
的系统。在Android平台上的蓝牙4.0主要有两种工作模式:经典蓝牙(classic bluetooth)
、低功耗蓝牙(bluetooth low energy,缩写为BLE)。

####2、角色与职责
当一个Android设备与一个BLE设备进行交互通信时，主要存在以下两种关系

* 中心设备与外围设备：中心设备扮演扫描的角色，寻找外围设备的广播消息。Android设备
  作为中心设备，与之连接通信的设备作为外围设备。
* GATT服务器与GATT客户端：这种关系决定了当连接建立后两个设备如何通信。


    注：目前Android系统提供的API使得Android设备只能作为中心设备

####3、组成部分
BLE分为三个部分Service、Characteristic、Descriptor，每个部分都拥有不同的
UUID来标识。一个BLE设备可以拥有多个Service，一个Service可以包含多个Characteristic，
一个Characteristic包含一个Value和多个Descriptor，一个Descriptor包含一个Value。
通信数据一般存储在Characteristic内，目前一个Characteristic中存储的数据最大为20 byte。
与Characteristic相关的权限字段主要有READ、WRITE、WRITE_NO_RESPONSE、NOTIFY。
Characteristic具有的权限属性可以有一个或者多个。

####4、核心代码

{% highlight java %}

// BLEDemo主要代码
private BluetoothAdapter mBtAdapter = null;
private BluetoothGatt mBtGatt = null;
private int mState = 0;
private Context mContext;
private BluetoothGattCharacteristic mWriteCharacteristic = null;
private BluetoothGattCharacteristic mReadCharacteristric = null;

private final String TAG = "BLE_Demo";

// 设备连接状态
private final int CONNECTED = 0x01;
private final int DISCONNECTED = 0x02;
private final int CONNECTTING = 0x03;

// 读写相关的Service、Characteristic的UUID
public static final UUID TRANSFER_SERVICE_READ = UUID.fromString("34567817-2432-5678-1235-3c1d5ab44e17");
public static final UUID TRANSFER_SERVICE_WRITE = UUID.fromString("34567817-2432-5678-1235-3c1d5ab44e18");
public static final UUID TRANSFER_CHARACTERISTIC_READ = UUID.fromString("23487654-5678-1235-2432-3c1d5ab44e94");
public static final UUID TRANSFER_CHARACTERISTIC_WRITE = UUID.fromString("23487654-5678-1235-2432-3c1d5ab44e93");

// BLE设备连接通信过程中回调
private BluetoothGattCallback mBtGattCallback = new BluetoothGattCallback() {

	// 连接状态发生改变时的回调
	@Override
	public void onConnectionStateChange(BluetoothGatt gatt, int status,
				int newState) {

		if (status == BluetoothGatt.GATT_SUCCESS) {
			mState = CONNECTED;
			Log.d(TAG, "connected OK");
			mBtGatt.discoverServices();
		} else if (newState == BluetoothGatt.GATT_FAILURE) {
			mState = DISCONNECTED;
			Log.d(TAG, "connect failed");
		}
	}

	// 远端设备中的服务可用时的回调
	@Override
	public void onServicesDiscovered(BluetoothGatt gatt, int status) {

		if (status == BluetoothGatt.GATT_SUCCESS) {
			BluetoothGattService btGattWriteService = mBtGatt
					.getService(TRANSFER_SERVICE_WRITE);
			BluetoothGattService btGattReadService = mBtGatt
					.getService(TRANSFER_SERVICE_READ);
			if (btGattWriteService != null) {
				mWriteCharacteristic = btGattWriteService
						.getCharacteristic(TRANSFER_CHARACTERISTIC_WRITE);
			}
			if (btGattReadService != null) {
				mReadCharacteristric = btGattReadService
						.getCharacteristic(TRANSFER_CHARACTERISTIC_READ);
				if (mReadCharacteristric != null) {
					mBtGatt.readCharacteristic(mReadCharacteristric);
				}
			}
		}
	}

	// 某Characteristic的状态为可读时的回调
	@Override
	public void onCharacteristicRead(BluetoothGatt gatt,
			BluetoothGattCharacteristic characteristic, int status) {

		if (status == BluetoothGatt.GATT_SUCCESS) {
			readCharacterisricValue(characteristic);

			// 订阅远端设备的characteristic，
			// 当此characteristic发生改变时当回调mBtGattCallback中的onCharacteristicChanged方法
			mBtGatt.setCharacteristicNotification(mReadCharacteristric,
					true);
			BluetoothGattDescriptor descriptor = mReadCharacteristric
					.getDescriptor(UUID
							.fromString("00002902-0000-1000-8000-00805f9b34fb"));
			if (descriptor != null) {
				byte[] val = BluetoothGattDescriptor.ENABLE_NOTIFICATION_VALUE;
				descriptor.setValue(val);
				mBtGatt.writeDescriptor(descriptor);
			}
		}
	}

	// 写入Characteristic成功与否的回调
	@Override
	public void onCharacteristicWrite(BluetoothGatt gatt,
			BluetoothGattCharacteristic characteristic, int status) {

		switch (status) {
		case BluetoothGatt.GATT_SUCCESS:
			Log.d(TAG, "write data success");
			break;// 写入成功
		case BluetoothGatt.GATT_FAILURE:
			Log.d(TAG, "write data failed");
			break;// 写入失败
		case BluetoothGatt.GATT_WRITE_NOT_PERMITTED:
			Log.d(TAG, "write not permitted");
			break;// 没有写入的权限
		}
	}

	// 订阅了远端设备的Characteristic信息后，
	// 当远端设备的Characteristic信息发生改变后,回调此方法
	@Override
	public void onCharacteristicChanged(BluetoothGatt gatt,
			BluetoothGattCharacteristic characteristic) {
		readCharacterisricValue(characteristic);
	}

};

/**
 * 读取BluetoothGattCharacteristic中的数据
 * 
 * @param characteristic
 */
private void readCharacterisricValue(
		BluetoothGattCharacteristic characteristic) {
	byte[] data = characteristic.getValue();
	StringBuffer buffer = new StringBuffer("0x");
	int i;
	for (byte b : data) {
		i = b & 0xff;
		buffer.append(Integer.toHexString(i));
	}
	Log.d(TAG, "read data:" + buffer.toString());
}

/**
 * 与指定的设备建立连接
 * 
 * @param device
 */
public void connect(BluetoothDevice device) {

	mBtGatt = device.connectGatt(mContext, false, mBtGattCallback);
	mState = CONNECTTING;
}

/**
 * 初始化
 * 
 * @param context
 * @return 如果初始化成功则返回true
 */
public boolean init(Context context) {
	BluetoothManager btMrg = (BluetoothManager) context
			.getSystemService(Context.BLUETOOTH_SERVICE);
	if (btMrg == null)
		return false;
	mBtAdapter = btMrg.getAdapter();
	if (mBtAdapter == null)
		return false;
	mContext = context;
	return true;
}

// BLE设备搜索过程中的回调，在此可以根据外围设备广播的消息来对设备进行过滤
private BluetoothAdapter.LeScanCallback mLeScanCallback = new BluetoothAdapter.LeScanCallback() {
	
	@Override
	public void onLeScan(final BluetoothDevice device, int rssi,
			byte[] scanRecord) {
		
		ArrayUtils.reverse(scanRecord);// 数组反转
		// 将Byte数组的数据以十六进制表示并拼接成字符串
		StringBuffer str = new StringBuffer();
		int i = 0;
		for (byte b : scanRecord) {
			i = (b & 0xff);
			str.append(Integer.toHexString(i));
		}
		String discoveryServceID = str.toString();
		Log.d(TAG, device.getName() + " scanRecord:\n" + discoveryServceID);
		
		// 查询是否含有指定的Service UUID信息
		if (discoveryServceID.indexOf(TRANSFER_SERVICE_WRITE.toString()
				.replace("-", "")) != -1) {

			Log.d(TAG, device.getName() + " has available service UUID");

			// 在这是处理匹配的设备……

		}

	}

};

/**
 * 开始BLE设备扫描
 */
public void startScan() {
	mBtAdapter.startLeScan(mLeScanCallback);
}

/**
 * 停止BLE设备扫描
 */
public void stopScan() {
	mBtAdapter.stopLeScan(mLeScanCallback);
}

/**
 * 发送数据
 * 
 * @param data
 *            待发送的数据,最大长度为20
 */
private void sendData(byte[] data) {

	if (data != null && data.length > 0 && data.length < 21) {
		if (mWriteCharacteristic.setValue(data)
				&& mBtGatt.writeCharacteristic(mWriteCharacteristic)) {
			Log.d(TAG, "send data OK");
		}
	}
}
    
{% endhighlight %}

####5、目前仍存在的问题
经过测试，在使用IOS的设备作为一个外围设备，使用Android设备通过BLE与之连接进行通信时，mBtGatt.writeCharacteristic写入成功，但是在mBtGattCallback.onCharacteristicWrite
回调中经常会出现status值为3即BluetoothGatt.GATT_WRITE_NOT_PERMITTED，从表面上看好像是因为IOS平台上的测试程序没有给予write的权限，但是实际是已经给予了write的权限。