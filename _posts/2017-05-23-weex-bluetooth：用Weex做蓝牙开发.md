---
layout:     post
title:      weex-bluetooth:用Weex做蓝牙开发
subtitle:   用Javascript，一行代码搞定一台蓝牙设备
date:       2017-05-23
author:     彳亍而行
header-img: 
catalog: true
tags:
    - Weex
---

# 缘起

之所以做这个项目，是因为公司要尝试用Weex做开发，以减少开发工作量，达到一个人搞定前端的目标。设想当中，如果顺利的话，一个人可以同时搞定iOS/Android两端的开发，尤其是UI方面的开发。传统的工作方式中，同一个UI设计需要iOS/Android两个平台实现基本一致的开发，非常浪费。

在此过程中，因为业务需要，要能够连接蓝牙设备，而Weex本身（基本上）只提供UI层的封装。为了也能够达到一次开发，两个平台同时运行的目标，我设计开发了供Weex调用的蓝牙库。地址在这里：[https://github.com/lixing123/weex-bluetooth](https://github.com/lixing123/weex-bluetooth)

# Project的优点

设计这个project的时候，考虑到Weex的开发人员一般都是Web开发者，对蓝牙的复杂流程的掌握会比较困难，因此我的目标是，普通的Web开发者，也可以在1-2个小时之内，完成接入一台蓝牙设备的开发。现在看起来，这个目标已经基本上实现了：**只用一行代码，就可以实现接入一台蓝牙设备**。

下面看一下例子：

```javascript
  //open bluetooth
  openBluetoothAdapter()
  .then(data => {//scan for BLE devices
    var services = [];
    return discoverDevice(services,function(device){//scan filter
      var deviceName = device['name'];
      var index = deviceName.indexOf("you-ble-name");
      return index != -1;
    });
  })
  .then(device => {//connect to BLE device
    return connectToDevice(device);
  }).then(device => {//discover service of BLE device
    return discoverServices(device);
  }).then(data => {//discover characteristic of a service
    var deviceID = data['deviceID'];
    var services = data['services'];
    for (var index in services){
      var serviceID = services[index]['UUID'];
      if (serviceID=="FFF0") {
        return discoverCharacteristics([deviceID, serviceID];
      }
    }
  }).then(data => {
    var deviceID = data[0];
    var serviceID = data[1];
    var characteristics = data[2];
    for (var i = 0; i < characteristics.length; i++) {
      var characteristicID = characteristics[i]['UUID'];
      if (characteristicID=="your-characteristic-UUID") {//listen to value change of characteristic
        listenToValueChangeOfCharacteristic(deviceID, serviceID, characteristicID,function(data){
          console.log(data);
        });
      }
    }
  });
```

嗯，的确是一行代码哦！虽然比较长:-D

用的是Javascript的Promise技术，所以能够一气呵成，从**打开蓝牙->扫描设备->连接设备->发现服务->发现特征->读/写/监听特征**。对于开发者来讲，只要有蓝牙设备的协议文档，就可以仿照这个写法，很快就可以完成。

而且project的文档很详细，就在week-bluetooth.js里。具体就不讲了，因为很简单易懂。贴一点代码来看看。比如对于` discoverDevice()`这个方法来说：

``` javascript
/**
 * start to discover BLE device.
 * @param  {Array}    services only scan for devices that broadcast(don't means contains) any of the listed services. Null means no limit.
 * @param  {function} filter   add filter to the device based on properties of device, including deviceID and name. 
 * The parameter "device" is a dictionary:
 * device = {
 *  'deviceID': (String) UUID(iOS) or Mac(Android) of the bluetooth device
 *  'name': (String) name of the device.
 * }
 * @return {Promise}  promise  a promise that will resolve with a device dictionary if a device satisfying requirements is found.
 * @discuss with limitations of Javascript Promise, only the first device will be returned, following device will be ignored.
 */
export function discoverDevice(services = [], filter = function(device){return true}){
  var promise = new Promise(function(resolve, reject){
    wx.startBluetoothDevicesDiscoveryWithServices(services,function(device){
      if (filter(device)) {
        resolve(device);
      }
    });
  });
  return promise;
}
```

# 模块和技术

在开发的过程中，遇到几个问题：

1. Native代码如何与Javascript通信
2. 如何简化使用方式，使得使用者能够方便上手
3. 如何保证接口的灵活性

下面一个一个来讲。

## Native代码如何与Javascript通信

因为是在Weex框架下开发，所以用到了Weex给的方法。Weex官方给出了互相调用的方法，即`WXModuleCallback`和`WXModuleKeepAliveCallback`。这两者的区别在于，`WXModuleKeepAliveCallback`可以被作为变量保存下来，长期使用。具体的文档在这里：http://weex.apache.org/cn/references/ios-apis.html

Weex还提供了一种global event方式的通信，本项目没有采用。考虑到可能会有连接多台设备、以及其它复杂情况，global event可能难以处理，并且native端使用的delegate，很适合用callback的方式，因此整个实现过程都不用global event。

native端的接口设计，模仿了微信小程序的蓝牙接口设计方式。由于Weex通信方式的限制，做了一定的调整。

微信小程序的蓝牙接口文档在这里：https://mp.weixin.qq.com/debug/wxadoc/dev/api/bluetooth.html

##  如何简化使用方式，使得使用者能够方便上手

蓝牙通信是一个蛮麻烦的事情：数据传输有UDP的特性，发出去了对方不一定能收到，收到了也不一定有返回；时常会因为各种原因断开连接；步骤也很复杂，先搜索，再连接，连接完了要搜索services，然后搜索某个service下的characteristics，根据不同的characteristic的用处，或读或写。Native端用的是delegate的方式，连接的时候各种delegate callback，一般不研究个1、2天，还真搞不清楚。

对于这种情况，我通过以下的方式来避免。

首先，用code block方式代替delegate，这样代码组织就更明晰了。这部分主要是在设计Native接口上进行的处理。

但是这样还不够，虽然用了block，但在使用的过程中，会遇到callback hell。下面是刚开始时Javascript调用的方式：

``` javasc
const wx = weex.requireModule('wx-ble');
        var that = this;
        wx.openBluetoothAdapter(function(res){
          // success
          var services = []
          wx.startBluetoothDevicesDiscoveryWithServices(services,function(res){
            var deviceID   = res['peripheral']['deviceID']
            var deviceName = res['peripheral']['name']
            var index = deviceName.indexOf("your-ble-name");
            if (index != -1){
              wx.createBLEConnectionWithDeviceID(deviceID, function(res){
                wx.stopBluetoothDeviceDiscovery(function(res){
                })
                wx.getBLEDeviceServicesWithDeviceID(deviceID, function(res){
                  that.target = res
                  for(var i=0;i<res.length;i++){
                    var serviceID = res[i]
                    that.target = that.target + serviceID
                    if (serviceID=="FFF0") {
                      that.target = that.target + ",," + serviceID
                      wx.getBLEDeviceCharacteristicsWithDeviceID(deviceID, serviceID, function(res){
                        that.target = res['characteristics']
                        var chars = res['characteristics']
                        for (var i = 0; i < chars.length; i++) {
                          var characteristicID = chars[i]
                          if (characteristicID=="FFF2") {//write characteristic
                            var value = "5A5A";
                            wx.writeBLECharacteristicValueWithDeviceID(deviceID, serviceID, characteristicID, value, function(res){
                            })
                          }
                          if (characteristicID=="FFF1") {//nofity characteristic
                            wx.onBLECharacteristicValueChange(function(res){
                              var value = res['value']
                              that.target = value
                            })
                            wx.notifyBLECharacteristicValueChangeWithDeviceID(deviceID, serviceID, characteristicID, true, function(res){
                              that.target = res
                            })
                          }
                        }
                      })
                    }
                  }
                });
              })
            }
          },)
        })
      }
```

嗯，至少有10个callback嵌套。我用了Javascript的Promise来处理这个问题。我的理解中，Promise是一种用于统一处理if/else代码块的方法，在多线程和条件嵌套情况下使用起来很方便。当然Promise也有局限，后面会讲到。

用Promise包装后的调用方法，就是刚开始的例子了。对比一下，区别还是蛮大的，逻辑清晰多了。

还有一个，蓝牙通信是基于"0101"这样的二进制数据来进行的，但一般来说，向蓝牙设备写数据的时候，都会将之转换成16进制数据，比如用"5A"来表示"01011010"。因此在接口的设计中，开发者可以直接用"5A5A"这样的字符串来向蓝牙设备中写数据，这样就简单许多。在读数据的时候，返回的则是直接的"01011010"这样的字符串，这是考虑到许多蓝牙设备的返回结果，每一位都有特定的含义，"01"字符串的方式更方便进行解析。

## 如何保证接口的灵活性

蓝牙开发中的概念还是蛮多的，开发过程中有一些一般情况下其实用不到，比如蓝牙设备的信号强度等。所以我在设计的时候，从实用的角度考虑，简化了一些数据的格式，保留了主要的数据，以使得开发起来更加方便。在一些关键的地方，我保留了自定义的接口，比如过滤设备、过滤service等。这样一般在开发基本上已经算是足够了。

设计接口的时候，受到了Web Bluetooth API的启发，它的实现方式也是Promise。有兴趣的可以了解一下：https://developers.google.com/web/updates/2015/07/interact-with-ble-devices-on-the-web

另外，在设计接口的时候，也参考了BabyBluetooth这个项目：https://github.com/coolnameismy/BabyBluetooth

# 不足和计划

目前只有iOS部分实现了，比较遗憾。以后有机会肯定还是要实现Android部分的，要不然不方便使用。

Promise的特性是一次性调用，而蓝牙通信其实有一些"Stream"的意思，导致了一些局限性。比如，发现了某一台蓝牙设备之后，如果resolve了，那么后面即使再发现一台设备，也无法再次调用resolve。后面准备通过结合callback的方式，弥补这个缺陷。

还有一些尚待完善的地方，比如当蓝牙断开连接时，需要通知到Javascript，这些异常事件的处理；另外，支持多设备同时连接等等，也可能会考虑进来。