---
layout: post
title:  "关于android平台BLE设备的过滤搜索遇到的问题及解决方法" 
author: 程鹏飞
tags: site tips-share
category: Android
---

###遇到的问题
在一般的情况下，根据官方提供的API，使用如下的方法：

{% highlight java %}
UUID serviceUUID = UUID.fromString("1a664749-82ba-4638-8b54-3c1d5ab44e93");
BluetoothAdapter.LeScanCallback mLeScanCallback = new BluetoothAdapter.LeScanCallback() {
			
	@Override
	public void onLeScan(BluetoothDevice device, int rssi, byte[] scanRecord) {
					
	}
};
BluetoothAdapter mBtAdapter = BluetoothAdapter.getDefaultAdapter();
mBtAdapter.startLeScan(new UUID[]{serviceUUID}, mLeScanCallback);
{% endhighlight %}

在onLeScan的回方法中，正常应该只要是外设设备广播的service的UUID与serviceUUID存在相同的时候，就可以通过此回调方法读取到发现的外设设备。但是实际的结果不能发现任何的设备。

###网络中的解决方法
网络中提供的问题解决参考方案：<http://stackoverflow.com/questions/18019161/startlescan-with-128-bit-uuids-doesnt-work-on-native-android-ble-implementation>

{% highlight java %}
	
//在搜索的时候去掉UUID过滤，即mBtAdapter.startLeScan(mLeScanCallback);
public static boolean hasMyService(byte[] scanRecord) {

// UUID we want to filter by (without hyphens)
final String myServiceID = "0000000000001000800000805F9B34FB";

// The offset in the scan record. In my case the offset was 13; it will probably be different for you
final int serviceOffset = 13; 

try{

	// Get a 16-byte array of what may or may not be the service we're filtering for
    byte[] service = ArrayUtils.subarray(scanRecord, serviceOffset, serviceOffset + 16);

    // The bytes are probably in reverse order, so we need to fix that
    ArrayUtils.reverse(service);

    // Get the hex string
    String discoveredServiceID = bytesToHex(service);

    // Compare against our service
    return myServiceID.equals(discoveredServiceID);

 } catch (Exception e){
        return false;
 }
 
}

{% endhighlight %}

###改进的解决方案
	
因为在测试的时候发现对于上述中提到的serviceOffset的值并不是13,我将将其修改为5时，才可以正确得到UUID的信息。所以改进的方法不去截取其中部分的字符串，而是使用indexOf方法来检测是否存在的方法，来判断是否为所需要的BLE设备。

{% highlight java %}

ArrayUtils.reverse(scanRecord);	 //数组反转
			
//将Byte数组的数据以十六进制表示并拼接成字符串
StringBuffer str = new StringBuffer();
int i = 0;
for (byte b : scanRecord) {
	i = (b & 0xff);
	str.append(Integer.toHexString(i));
}
String discoveryServceID = str.toString();
Log.i(TAG, device.getAddress()+" scanRecord:"+discoveryServceID);
Log.i(TAG, "local serviceID:"+ConstantValues.TRANSFER_SERVICE_READ.toString());
			
//查询是否含有指定的Service UUID信息
if(discoveryServceID.indexOf(ConstantValues.TRANSFER_SERVICE_READ.toString().replace("-", "")) != -1){
	Log.i(TAG, device.getAddress() +" has available service UUID");
	for (BluetoothDevice c : mFoundDevices) {
		if(c.getAddress().equals(device.getAddress())) return;
	}
	mFoundDevices.add(device);
	Log.i(TAG, "add Ble "+device.getAddress()+" to BleDeviceList");
}
{% endhighlight %}