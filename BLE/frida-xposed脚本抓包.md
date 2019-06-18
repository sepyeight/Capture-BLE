# frida-xposed脚本抓包
frida脚本
```cpp
/**
 * Colorful output
 */
var Color = {
    Reset: "\x1b[39;49;00m",
    Black: "\x1b[30;01m", Blue: "\x1b[34;01m", Cyan: "\x1b[36;01m", Gray: "\x1b[37;11m",
    Green: "\x1b[32;01m", Purple: "\x1b[35;01m", Red: "\x1b[31;01m", Yellow: "\x1b[33;01m",
    Light: {
        Black: "\x1b[30;11m", Blue: "\x1b[34;11m", Cyan: "\x1b[36;11m", Gray: "\x1b[37;01m",
        Green: "\x1b[32;11m", Purple: "\x1b[35;11m", Red: "\x1b[31;11m", Yellow: "\x1b[33;11m"
    }
};

/**
 * convert bytes to hex string
 * @param arrBytes
 * @returns {string}
 */
function toHexString(arrBytes) {
    var str = "";
    for (var i = 0; i < arrBytes.length; i++) {
        var tmp;
        var num = arrBytes[i];
        if (num < 0) {
            tmp = (255 + num + 1).toString(16);
        } else {
            tmp = num.toString(16);
        }
        if (tmp.length == 1) {
            tmp = "0" + tmp;
        }
        str += tmp;
    }
    return str;
}

/**
 * hook main functions to get ble info.
 */
function getBleInfo() {
    var BluetoothDevice = Java.use("android.bluetooth.BluetoothDevice");
    BluetoothDevice.connectGatt.overload("android.content.Context", "boolean", "android.bluetooth.BluetoothGattCallback").implementation = function (context, autoConnect, callback) {
        var name = this.getName();
        var mac = this.getAddress();
        var bond = this.getBondState();
        var type = this.getType();

        switch (bond) {
            case 10:
                bond = "BOND_NONE";
                break;
            case 11:
                bond = "BOND_BONDING";
                break;
            case 12:
                bond = "BOND_BONDED";
                break;
            default:
                bond = "NONE";
        }
        switch (type) {
            case 1:
                type = "DEVICE_TYPE_CLASSIC";
                break;
            case 2:
                type = "DEVICE_TYPE_LE";
                break;
            case 3:
                type = "DEVICE_TYPE_DUAL";
                break;
            default:
                type = "DEVICE_TYPE_UNKNOWN";
        }
        console.log(Color.Red + "[BLE Connect  =>]connect to " + name + ", MAC: " + mac + ", bond state: " + bond + ", type: " + type);
        return this.connectGatt(context, autoConnect, callback);
    };

    BluetoothDevice.createBond.overload().implementation = function () {
        console.log(Color.Cyan + "[BLE Create Bond  ====]");
        return this.createBond();
    };

    BluetoothDevice.setPin.implementation = function (byte) {
        console.log(Color.Cyan + "[BLE Set Pin]  =>" + toHexString(byte));
        return this.setPin(byte);
    };
    // hook BluetoothGattCallback
    var BluetoothGattCallback = Java.use("android.bluetooth.BluetoothGattCallback");
    BluetoothGattCallback.$init.overload().implementation = function () {
        // console.log("BluetoothGattCallback constructor called from " + this.$className);
        var NewCB = Java.use(this.$className);
        NewCB.onServicesDiscovered.implementation = function (gatt, status) {
            if (status == 0) { // 0 = GATT_SUCCESS
                initBle(gatt);
            }
            return this.onServicesDiscovered(gatt, status);
        };
        NewCB.onCharacteristicRead.implementation = function (gatt, charac, status) {
            if (status == 0) { // 0 = GATT_SUCCESS
                var uuid = charac.getUuid().toString();
                var value = charac.getValue();
                console.log(Color.Purple + "[Thread-" + Process.getCurrentThreadId() + "-BLE onCharac Read   <=]" + Color.Light.Black + " UUID: " + uuid + Color.Reset + " data: 0x" + toHexString(value));
            }
            return this.onCharacteristicRead(gatt, charac, status);
        };
        NewCB.onCharacteristicWrite.implementation = function (gatt, charac, status) {
            if (status == 0) { // 0 = GATT_SUCCESS
                var uuid = charac.getUuid().toString();
                var value = charac.getValue();
                console.log(Color.Purple + "[Thread-" + Process.getCurrentThreadId() + "-BLE onCharac Write  <=]" + Color.Light.Black + " UUID: " + uuid + Color.Reset + " data: 0x" + toHexString(value));
            }
            return this.onCharacteristicWrite(gatt, charac, status);
        };
        NewCB.onCharacteristicChanged.implementation = function (gatt, charac) {
            var uuid = charac.getUuid().toString();
            var value = charac.getValue();
            console.log(Color.Gray + "[Thread-" + Process.getCurrentThreadId() + "-BLE onCharac Changed <=]" + Color.Light.Black + " UUID: " + uuid + Color.Reset + " data: 0x" + toHexString(value));
            return this.onCharacteristicChanged(gatt, charac);
        };
        NewCB.onDescriptorRead.implementation = function (gatt, desc, status) {
            if (status == 0) {
                var uuid = desc.getUuid().toString();
                var value = desc.getValue();
                console.log(Color.Yellow + "[Thread-" + Process.getCurrentThreadId() + "-BLE onDesc Read  <=]" + Color.Light.Black + " UUID: " + uuid + Color.Reset + " data: 0x" + toHexString(value));
            }
            return this.onDescriptorRead(gatt, desc, status);
        };
        NewCB.onDescriptorWrite.implementation = function (gatt, desc, status) {
            if (status == 0) {
                var uuid = desc.getUuid().toString();
                var value = desc.getValue();
                // ENABLE_INDICATION_VALUE = 0x0200
                console.log(Color.Yellow + "[Thread-" + Process.getCurrentThreadId() + "-BLE onDesc Write  <=]" + Color.Light.Black + " UUID: " + uuid + Color.Reset + " data: 0x" + toHexString(value));
            }
            return this.onDescriptorWrite(gatt, desc, status);
        };
        NewCB.onReadRemoteRssi.implementation = function (gatt, rssi, status) {
            if (status == 0) {
                console.log(Color.Green + "[Thread-" + Process.getCurrentThreadId() + "-BLE Rssi  =>]" + Color.Light.Black + " Rssi: " + rssi);
            }
            return this.onReadRemoteRssi(gatt, rssi, status);
        };
        return this.$init();
    };

    // hook BluetoothGatt
    var BluetoothGatt = Java.use("android.bluetooth.BluetoothGatt");
    BluetoothGatt.setCharacteristicNotification.implementation = function (bleGattChar, boolean) {
        console.log(Color.Gray + "[Thread-" + Process.getCurrentThreadId() + "-BLE Set Charac Notify  =>] " + Color.Light.Black + "UUID:" + bleGattChar.getUuid() + Color.Reset + ", notify: " + boolean);
        return this.setCharacteristicNotification(bleGattChar, boolean);
    };

    BluetoothGatt.requestMtu.implementation = function (intValue) {
        console.log(Color.Green + "[Thread-" + Process.getCurrentThreadId() + "-BLE Set Mtu  =>]" + Color.Light.Black + intValue);
        return this.requestMtu(intValue);
    };

    // hook BluetoothGattCharacteristic
    var BluetoothGattCharacteristic = Java.use("android.bluetooth.BluetoothGattCharacteristic");
    BluetoothGattCharacteristic.setValue.overload("[B").implementation = function (bytes) {
        var BluetoothGattCharacteristicObj = Java.cast(this, BluetoothGattCharacteristic);
        console.log(Color.Purple + "[Thread-" + Process.getCurrentThreadId() + "-BLE Set Charac Value  =>]" + Color.Light.Black + "UUID:" + BluetoothGattCharacteristicObj.getUuid() + Color.Reset + ", data: 0x" + toHexString(bytes));
        return this.setValue(bytes);
    };

    BluetoothGattCharacteristic.setValue.overload("int", "int", "int").implementation = function (intValue01, intValue02, intValue03) {
        var BluetoothGattCharacteristicObj = Java.cast(this, BluetoothGattCharacteristic);
        console.log(Color.Purple + "[Thread-" + Process.getCurrentThreadId() + "-BLE Set Charac Value  =>]" + Color.Light.Black + BluetoothGattCharacteristicObj.getUuid() + Color.Reset
            + ", data01: 0x" + intValue01
            + ", data02: 0x" + intValue02
            + ", data03: 0x" + intValue03);
        return this.setValue(intValue01, intValue02, intValue03);
    };

    BluetoothGattCharacteristic.setValue.overload("int", "int", "int", "int").implementation = function (intValue01, intValue02, intValue03, intValue04) {
        var BluetoothGattCharacteristicObj = Java.cast(this, BluetoothGattCharacteristic);
        console.log(Color.Purple + "[Thread-" + Process.getCurrentThreadId() + "-BLE Set Charac Value  =>]" + Color.Light.Black + BluetoothGattCharacteristicObj.getUuid() + Color.Reset
            + ", data01: 0x" + intValue01
            + ", data02: 0x" + intValue02
            + ", data03: 0x" + intValue03
            + ", data04: 0x" + intValue04);
        return this.setValue(intValue01, intValue02, intValue03, intValue04);
    };

    BluetoothGattCharacteristic.setValue.overload("java.lang.String").implementation = function (str) {
        var BluetoothGattCharacteristicObj = Java.cast(this, BluetoothGattCharacteristic);
        console.log(Color.Purple + "[Thread-" + Process.getCurrentThreadId() + "-BLE Set Charac Value  =>]" + Color.Light.Black + BluetoothGattCharacteristicObj.getUuid() + Color.Reset
            + ", str: 0x" + str);
        return this.setValue(str);
    };

    // hook BluetoothGattDescriptor
    var BluetoothGattDescriptor = Java.use("android.bluetooth.BluetoothGattDescriptor");
    BluetoothGattDescriptor.setValue.overload("[B").implementation = function (bytes) {
        var BluetoothGattDescriptorObj = Java.cast(this, BluetoothGattDescriptor);
        console.log(Color.Yellow + "[Thread-" + Process.getCurrentThreadId() + "-BLE Set Desc Value  =>]" + Color.Light.Black + BluetoothGattDescriptorObj.getUuid() + Color.Reset
            + ", byte: 0x" + toHexString(bytes));
        return this.setValue(bytes);
    };


}

/**
 * show ble service, charac, desc uuid
 * @param gatt
 */
function initBle(gatt) {
    if (gatt == null) {
        return;
    }
    var BluetoothGattService = Java.use("android.bluetooth.BluetoothGattService");
    var BluetoothGattCharacteristic = Java.use("android.bluetooth.BluetoothGattCharacteristic");
    var BluetoothGattDescriptor = Java.use("android.bluetooth.BluetoothGattDescriptor");

    console.log(Color.Blue + "===========================BLE channal info============================");
    var services = gatt.getServices();
    for (var i = 0; i < services.size(); i++) {
        var ser = services.get(i);
        var serObj = Java.cast(ser, BluetoothGattService);
        var seruuid = serObj.getUuid().toString();
        console.log(Color.Green + "[BLE Service =>]" + Color.Light.Black + "UUID: " + seruuid);
        var characs = serObj.getCharacteristics();
        for (var j = 0; j < characs.size(); j++) {
            var charac = characs.get(j);
            var characObj = Java.cast(charac, BluetoothGattCharacteristic);
            var characuuid = characObj.getUuid().toString();
            console.log("\t" + Color.Green + "[BLE Characteristic =>]" + Color.Light.Black + "UUID" + characuuid);
            var descs = characObj.getDescriptors();
            for (var k = 0; k < descs.size(); k++) {
                var desc = descs.get(k);
                var descObj = Java.cast(desc, BluetoothGattDescriptor);
                var descuuid = descObj.getUuid().toString();
                console.log("\t\t" + Color.Green + "[BLE Descriptor =>]" + Color.Light.Black + "UUID" + descuuid);
            }
        }
    }
    console.log(Color.Blue + "==================================BLE==================================");
}

/**
 * begin
 */
function hook() {
    if (Java.available) {
        Java.perform(function () {
            getBleInfo();
        });
    }
}

setImmediate(hook);
```

[xposed项目](https://github.com/sepyeight/AndroidXposedBLEHelper)