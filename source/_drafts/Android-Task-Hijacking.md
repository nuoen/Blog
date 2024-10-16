---
title: Android Task Hijacking
author: nuoen
tags:
---
解压android备份
java -jar  abe.jar unpack  back.ab back.tar
openssl zlib -d -in back.tar -out decompressed.tar

09-02 15:38:30.573 D/CorePlayer(15550): Use user's preferred resolution : 7
09-02 15:38:30.573 I/CorePlayer(15550): decreaseDefinition: resolution = 3
09-02 15:38:30.573 I/IqiyiSdkVideoView(15550): setResolution: 0 -> 3
09-02 15:38:30.574 I/IqiyiSdkVideoView(15550): toQiyiBitStream, resolution : 3, bitstream: null
09-02 15:38:30.574 I/IqiyiSdkVideoView(15550): setResolution: quality is null, errorCode: 2100

09-02 15:38:30.622 I/IqiyiSdkVideoView(15550): createAndPrepareMediaPlayer, playDolby: false, mVipMember: true, mPaid: true, audioType: 0, Bitstream definition: 5, dynamicRangeType: 1