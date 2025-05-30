---
sidebar_position: 3
---

# Tích hợp Cocos2d-x

Hướng dẫn tích hợp **VisitorTrackingSDK** vào một dự án game Cocos2d-x (Android & iOS) để:

- `Tự động lấy địa chỉ IP của thiết bị`
- `Gửi IP và token lên API để lấy mã MD5 nhận diện người dùng.`
- `Lưu mã MD5 vào bộ nhớ cục bộ.`
- `Cung cấp hàm getMd5() để dùng trong game.`

## Cấu trúc thư mục SDK (VisitorTrackingSDK)

```
Classes/
├── VisitorTrackingSDK/
│   ├── VisitorTrackingSDK.h
│   ├── VisitorTrackingSDK.cpp
│   └── platform/
│       ├── android/
│       │   ├── VisitorTrackingAndroid.h
│       │   └── VisitorTrackingAndroid.cpp
│       └── ios/
│           ├── VisitorTrackingIOS.h
│           └── VisitorTrackingIOS.mm
|   └── CMakeLists.txt
```

```md title="VisitorTrackingSDK"
// VisitorTrackingSDK.h
#pragma once

#include <string>
#include <functional>

class VisitorTrackingSDK {
public:
    static VisitorTrackingSDK* getInstance();

    void init(const std::string& apiUrl,
              const std::string& token,
              std::function<void(std::string)> callback);

    std::string getMd5() const;

private:
    VisitorTrackingSDK();
    std::string md5;
    std::string token;
    std::string apiUrl;

    std::string getMd5FromLocal();
    void setMd5ToLocal(const std::string& md5);
    std::string getIpAddress();
    void fetchMd5FromApi(const std::string& ip,
                         std::function<void(std::string)> callback);
};


// VisitorTrackingSDK.cpp
#include "VisitorTrackingSDK.h"
#include "platform/CCPlatformMacros.h"
#include "platform/CCFileUtils.h"
#include "base/CCUserDefault.h"
#include "network/HttpClient.h"
#include <sstream>

#if (CC_TARGET_PLATFORM == CC_PLATFORM_ANDROID)
#include "platform/android/VisitorTrackingAndroid.h"
#elif (CC_TARGET_PLATFORM == CC_PLATFORM_IOS)
#include "platform/ios/VisitorTrackingIOS.h"
#endif

USING_NS_CC;
using namespace cocos2d::network;

VisitorTrackingSDK* VisitorTrackingSDK::getInstance() {
    static VisitorTrackingSDK instance;
    return &instance;
}

VisitorTrackingSDK::VisitorTrackingSDK() {}

void VisitorTrackingSDK::init(const std::string& apiUrl,
                               const std::string& token,
                               std::function<void(std::string)> callback) {
    this->apiUrl = apiUrl;
    this->token = token;
    this->md5 = getMd5FromLocal();

    if (!md5.empty()) {
        callback(md5);
        return;
    }

    std::string ip = getIpAddress();
    fetchMd5FromApi(ip, [this, callback](std::string md5Val) {
        this->md5 = md5Val;
        setMd5ToLocal(md5Val);
        callback(md5Val);
    });
}

std::string VisitorTrackingSDK::getMd5() const {
    return md5;
}

std::string VisitorTrackingSDK::getMd5FromLocal() {
    return UserDefault::getInstance()->getStringForKey(("visitor_md5_" + token).c_str(), "");
}

void VisitorTrackingSDK::setMd5ToLocal(const std::string& md5) {
    UserDefault::getInstance()->setStringForKey(("visitor_md5_" + token).c_str(), md5);
    UserDefault::getInstance()->flush();
}

std::string VisitorTrackingSDK::getIpAddress() {
#if (CC_TARGET_PLATFORM == CC_PLATFORM_ANDROID)
    return getIpAddressAndroid();
#elif (CC_TARGET_PLATFORM == CC_PLATFORM_IOS)
    return getIpAddressIOS();
#else
    return "";
#endif
}

void VisitorTrackingSDK::fetchMd5FromApi(const std::string& ip, std::function<void(std::string)> callback) {
    std::stringstream ss;
    ss << apiUrl << "?token=" << token << "&ip=" << ip;

    HttpRequest* request = new HttpRequest();
    request->setUrl(ss.str().c_str());
    request->setRequestType(HttpRequest::Type::GET);
    request->setResponseCallback([callback](HttpClient* client, HttpResponse* response) {
        if (!response || !response->isSucceed()) {
            callback("");
            return;
        }

        std::vector<char>* buffer = response->getResponseData();
        std::string jsonStr(buffer->begin(), buffer->end());

        // Giả sử server trả JSON: { "md5": "abc123" }
        size_t start = jsonStr.find(":\"");
        size_t end = jsonStr.find("\"}");
        if (start != std::string::npos && end != std::string::npos && end > start + 2) {
            std::string md5Value = jsonStr.substr(start + 2, end - start - 2);
            callback(md5Value);
        } else {
            callback("");
        }
    });

    HttpClient::getInstance()->send(request);
    request->release();
}  


// VisitorTrackingAndroid.h
#pragma once
#include <string>
std::string getIpAddressAndroid();

// VisitorTrackingAndroid.cpp
#include "platform/android/jni/JniHelper.h"
#include <jni.h>

std::string getIpAddressAndroid() {
    // TODO: viết JNI để lấy IP từ Java
    return "127.0.0.1";
}


// VisitorTrackingIOS.h
#pragma once
#include <string>
std::string getIpAddressIOS();

// VisitorTrackingIOS.mm
#import <ifaddrs.h>
#import <arpa/inet.h>
#import <net/if.h>
#import <Foundation/Foundation.h>

std::string getIpAddressIOS() {
    struct ifaddrs* interfaces = nullptr;
    struct ifaddrs* tempAddr = nullptr;
    std::string address = "";

    if (getifaddrs(&interfaces) == 0) {
        tempAddr = interfaces;
        while (tempAddr != nullptr) {
            if (tempAddr->ifa_addr->sa_family == AF_INET) {
                if (strcmp(tempAddr->ifa_name, "en0") == 0) {
                    address = inet_ntoa(((struct sockaddr_in*)tempAddr->ifa_addr)->sin_addr);
                    break;
                }
            }
            tempAddr = tempAddr->ifa_next;
        }
    }
    freeifaddrs(interfaces);
    return address;
}

```

## 📱 Tích hợp Android

## Bước 1: Thêm Java helper

Tạo **app/src/main/java/org/cocos2dx/cpp/DeviceUtils.java**:

```
package org.cocos2dx.cpp;

import android.content.Context;
import android.net.wifi.WifiManager;
import android.text.format.Formatter;

public class DeviceUtils {
    public static String getIPAddress(Context context) {
        try {
            WifiManager wm = (WifiManager) context.getSystemService(Context.WIFI_SERVICE);
            return Formatter.formatIpAddress(wm.getConnectionInfo().getIpAddress());
        } catch (Exception e) {
            return "";
        }
    }
}
```

## Bước 2: Gọi JNI trong VisitorTrackingAndroid.cpp

```
#include "platform/android/jni/JniHelper.h"

std::string getDeviceIP() {
    std::string ip = "";
    JniMethodInfo methodInfo;

    if (JniHelper::getStaticMethodInfo(methodInfo,
        "org/cocos2dx/cpp/DeviceUtils", "getIPAddress", "(Landroid/content/Context;)Ljava/lang/String;")) {

        jobject context = cocos2d::JniHelper::getContext();
        jstring jIp = (jstring)methodInfo.env->CallStaticObjectMethod(methodInfo.classID, methodInfo.methodID, context);

        const char* ipChars = methodInfo.env->GetStringUTFChars(jIp, nullptr);
        ip = std::string(ipChars);
        methodInfo.env->ReleaseStringUTFChars(jIp, ipChars);
        methodInfo.env->DeleteLocalRef(jIp);
        methodInfo.env->DeleteLocalRef(methodInfo.classID);
    }
    return ip;
}
```
## 📱 Tích hợp iOS

Trong **VisitorTrackingIOS.mm**:

```
#import <ifaddrs.h>
#import <arpa/inet.h>

std::string getDeviceIPIOS() {
    NSString *address = @"";
    struct ifaddrs *interfaces = NULL;
    struct ifaddrs *temp_addr = NULL;
    int success = getifaddrs(&interfaces);
    if (success == 0) {
        temp_addr = interfaces;
        while(temp_addr != NULL) {
            if(temp_addr->ifa_addr->sa_family == AF_INET) {
                if([[NSString stringWithUTF8String:temp_addr->ifa_name] isEqualToString:@"en0"]) {
                    address = [NSString stringWithUTF8String:
                        inet_ntoa(((struct sockaddr_in *)temp_addr->ifa_addr)->sin_addr)];
                    break;
                }
            }
            temp_addr = temp_addr->ifa_next;
        }
    }
    freeifaddrs(interfaces);
    return std::string([address UTF8String]);
}
```

## 📱 Tích hợp CMake

CMakeLists.txt **in root or** Classes/:

```
add_subdirectory(Classes/VisitorTrackingSDK)
```

```
file(GLOB VISITOR_SOURCES
    "*.cpp"
    "platform/android/*.cpp"
    "platform/ios/*.mm"
)

add_library(visitor_tracking STATIC ${VISITOR_SOURCES})

target_include_directories(visitor_tracking PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})
```

## ⚙️ Cài đặt

Gọi SDK trong **AppDelegate.cpp** của bạn hoặc bất cứ nơi nào phù hợp:

```
#include 'VisitorTrackingSDK/VisitorTrackingSDK.h'

VisitorTrackingSDK::getInstance()->init(
    "https://your-api.com/md5",   // API URL
    "your-token",           // Token
    [](std::string md5) {
        CCLOG("MD5 fetched: %s", md5.c_str());
    }
);
```